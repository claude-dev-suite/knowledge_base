# Windows Drivers - Indirect Display - NVENC

> **Source**: https://docs.nvidia.com/video-technologies/video-codec-sdk/nvenc-video-encoder-api-prog-guide/
> **Skill**: dev-suite skill `windows/indirect-display` — see SKILL.md for the always-loaded quick reference.

## What this covers

The NVIDIA Video Codec SDK encoder pipeline tuned for IDD swap-chain consumption:
low-latency preset, D3D11 input texture path, B-frames disabled, async output via
event handles. The SKILL mentions NVENC; this file shows the actual API calls and
the tuning that gets you sub-frame latency.

## Deep dive

### Pipeline overview

```
IDD swap-chain texture
        │ OpenSharedResource1 on encoder D3D11 device
        ▼
ID3D11Texture2D (BGRA, R10G10B10A2 for HDR)
        │ NvEncRegisterResource
        ▼
NV_ENC_REGISTERED_PTR
        │ NvEncMapInputResource per frame
        ▼
NV_ENC_INPUT_PTR  →  NvEncEncodePicture  →  NV_ENC_OUTPUT_PTR
                                                │ event signaled
                                                ▼
                                       NvEncLockBitstream → bytes
                                                │
                                                ▼
                                       Network sender
```

### Session creation tuned for low latency

```c
NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS open = {
    .version = NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS_VER,
    .deviceType = NV_ENC_DEVICE_TYPE_DIRECTX,
    .device = pD3D11Device,        // shared with the IDD swap-chain GPU
    .apiVersion = NVENCAPI_VERSION
};
void* nvEncoder = NULL;
NVENCSTATUS s = nvenc.nvEncOpenEncodeSessionEx(&open, &nvEncoder);

NV_ENC_INITIALIZE_PARAMS init = { .version = NV_ENC_INITIALIZE_PARAMS_VER };
init.encodeGUID    = NV_ENC_CODEC_HEVC_GUID;            // or H264, AV1
init.presetGUID    = NV_ENC_PRESET_P3_GUID;             // P1 fastest, P7 best quality
init.tuningInfo    = NV_ENC_TUNING_INFO_ULTRA_LOW_LATENCY;
init.encodeWidth   = monitorW;
init.encodeHeight  = monitorH;
init.darWidth      = monitorW;
init.darHeight     = monitorH;
init.frameRateNum  = 60;
init.frameRateDen  = 1;
init.enablePTD     = 1;            // picture type decision driven by encoder

NV_ENC_CONFIG cfg = { .version = NV_ENC_CONFIG_VER };
nvenc.nvEncGetEncodePresetConfigEx(nvEncoder, init.encodeGUID, init.presetGUID,
                                   init.tuningInfo,
                                   &(NV_ENC_PRESET_CONFIG){
                                       .version = NV_ENC_PRESET_CONFIG_VER,
                                       .presetCfg.version = NV_ENC_CONFIG_VER
                                   });
cfg.gopLength            = NVENC_INFINITE_GOPLENGTH;     // intra refresh, no IDR after first
cfg.frameIntervalP       = 1;                            // P only, no B
cfg.rcParams.rateControlMode = NV_ENC_PARAMS_RC_CBR;
cfg.rcParams.averageBitRate  = 25 * 1000 * 1000;         // 25 Mbps
cfg.rcParams.vbvBufferSize   = cfg.rcParams.averageBitRate / 60;  // 1-frame VBV
cfg.rcParams.vbvInitialDelay = cfg.rcParams.vbvBufferSize;
cfg.encodeCodecConfig.hevcConfig.idrPeriod = NVENC_INFINITE_GOPLENGTH;
init.encodeConfig = &cfg;

nvenc.nvEncInitializeEncoder(nvEncoder, &init);
```

Key tunings for IDD streaming:

- **No B-frames**: `frameIntervalP = 1`. B-frames add reorder latency = 1+ frames.
- **CBR + small VBV**: 1-frame VBV keeps bitrate flat → predictable network pacing.
- **Infinite GOP + intra refresh**: avoid periodic IDR spikes that cause network
  jitter. Enable intra refresh via
  `cfg.encodeCodecConfig.hevcConfig.repeatSPSPPS = 1` and
  `cfg.encodeCodecConfig.hevcConfig.intraRefreshPeriod = 600`.
- **ULTRA_LOW_LATENCY tuning**: enables encoder internal pacing optimizations.

### Per-frame encode

```c
NV_ENC_REGISTER_RESOURCE reg = { .version = NV_ENC_REGISTER_RESOURCE_VER };
reg.resourceType         = NV_ENC_INPUT_RESOURCE_TYPE_DIRECTX;
reg.resourceToRegister   = pInputTexture;            // ID3D11Texture2D from OpenSharedResource1
reg.width                = monitorW;
reg.height               = monitorH;
reg.bufferFormat         = NV_ENC_BUFFER_FORMAT_ARGB;   // BGRA8, or ABGR10 for HDR
nvenc.nvEncRegisterResource(nvEncoder, &reg);

NV_ENC_MAP_INPUT_RESOURCE map = { .version = NV_ENC_MAP_INPUT_RESOURCE_VER };
map.registeredResource = reg.registeredResource;
nvenc.nvEncMapInputResource(nvEncoder, &map);

NV_ENC_PIC_PARAMS pic = { .version = NV_ENC_PIC_PARAMS_VER };
pic.inputBuffer    = map.mappedResource;
pic.bufferFmt      = map.mappedBufferFmt;
pic.inputWidth     = monitorW;
pic.inputHeight    = monitorH;
pic.outputBitstream= bitstreamBuf;        // pre-allocated NV_ENC_OUTPUT_PTR
pic.completionEvent= hCompletionEvent;     // OS event signaled when ready
pic.pictureStruct  = NV_ENC_PIC_STRUCT_FRAME;
NVENCSTATUS encS = nvenc.nvEncEncodePicture(nvEncoder, &pic);
// encS == NV_ENC_SUCCESS or NV_ENC_ERR_NEED_MORE_INPUT (only first frame)
```

Then on a separate "drain" thread:

```c
WaitForSingleObject(hCompletionEvent, INFINITE);
NV_ENC_LOCK_BITSTREAM lock = { .version = NV_ENC_LOCK_BITSTREAM_VER,
                               .outputBitstream = bitstreamBuf };
nvenc.nvEncLockBitstream(nvEncoder, &lock);
SendBytes(lock.bitstreamBufferPtr, lock.bitstreamSizeInBytes);
nvenc.nvEncUnlockBitstream(nvEncoder, bitstreamBuf);
nvenc.nvEncUnmapInputResource(nvEncoder, map.mappedResource);
```

Always `Unmap` after lock — leaking mapped resources rapidly exhausts the
encoder's internal pool.

### HDR encode

For 10-bit HDR swap chains:

- `bufferFormat = NV_ENC_BUFFER_FORMAT_ABGR10`
- `cfg.profileGUID = NV_ENC_HEVC_PROFILE_MAIN10_GUID`
- Set `cfg.encodeCodecConfig.hevcConfig.outputAUD = 1` so the receiver can sync
  HDR metadata SEI.
- Inject `Mastering Display Color Volume` and `Content Light Level` SEI messages
  via `pic.codecPicParams.hevcPicParams.seiPayloadArray`.

### Cleanup ordering

```c
nvEncDestroyBitstreamBuffer(...)   // for each NV_ENC_OUTPUT_PTR
nvEncUnregisterResource(...)        // for each registered input
nvEncDestroyEncoder(nvEncoder);
```

Skipping unregister produces a slow leak that only manifests after hours of
streaming.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Calling `nvEncEncodePicture` from the swap-chain thread without `completionEvent` | Encoder blocks the IDD loop | Use async event-based submission |
| `frameIntervalP = 3` (default for some presets) | Adds 2-frame B reorder latency | Set to 1 for streaming |
| Letting encoder pick its own GOP | Periodic IDR keyframes spike bandwidth and cause hitches | `gopLength = NVENC_INFINITE_GOPLENGTH` + intra refresh |
| Encoder GPU != IDD swap-chain GPU | Forces cross-adapter copy every frame | Pin both to same `IDXGIAdapter` via LUID |
| Forgetting to call `nvEncFlush` on shutdown | Last 1-2 frames lost | Call `nvEncEncodePicture` with `encodePicFlags = NV_ENC_PIC_FLAG_EOS` |

## See also

- https://docs.nvidia.com/video-technologies/video-codec-sdk/nvenc-application-note/
- https://github.com/NVIDIA/video-sdk-samples
- https://docs.nvidia.com/video-technologies/video-codec-sdk/13.0/nvenc-video-encoder-api-prog-guide/index.html#low-latency-encoding
