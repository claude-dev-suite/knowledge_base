# Windows Drivers - Indirect Display - NVIDIA Rivermax

> **Source**: https://developer.nvidia.com/networking/rivermax
> **Skill**: dev-suite skill `windows/indirect-display` — see SKILL.md for the always-loaded quick reference.

## What this covers

Pairing an IDD virtual display with NVIDIA Rivermax for sub-millisecond network
delivery: hardware-timed RTP, SMPTE 2110 / 2022-7 transports, GPUDirect zero-copy
from the encoder output, and ConnectX NIC requirements. The SKILL lists Rivermax
as the lowest-latency option; this file shows how to actually wire it up.

## Deep dive

### Why Rivermax

- **Hardware packet pacing** on ConnectX-6/7 NICs: the NIC times each RTP packet
  to the wire with sub-microsecond accuracy, eliminating the jitter a kernel
  scheduler introduces.
- **GPUDirect**: encoder output (NVENC bitstream or raw pixels for SMPTE 2110-20)
  can be DMA'd straight from GPU memory to the NIC, bypassing CPU staging.
- **PTP (IEEE 1588) timestamping**: send timestamps that downstream broadcast
  equipment can use for genlock.
- **SMPTE 2022-7 dual-path redundancy**: send identical streams over two NICs;
  receiver merges, hides single-NIC packet loss.

### When Rivermax is overkill

- LAN game streaming (use NVENC + RTP/UDP without Rivermax — saves the SDK
  licensing cost).
- Single-monitor remote desktop (NVENC + WebRTC works fine at 30-50ms).

### Hardware required

| Component | Purpose |
|-----------|---------|
| Mellanox/NVIDIA ConnectX-6 Dx, ConnectX-7, BlueField-2/3 | Hardware-timed TX, GPUDirect endpoint |
| NVIDIA dGPU (Quadro RTX / RTX A-series, A40, L40) | NVENC + GPUDirect peer DMA |
| 25/50/100 GbE switch with PTP transparent clock | Genlock and pacing |
| Rivermax SDK + license | Runtime API |

GPUDirect requires the GPU and NIC sit on PCIe paths that the platform allows
peer DMA between (same root complex or chipset that supports ACS bypass).

### Topology

```
IDD swap-chain → NVENC (HEVC bitstream in GPU memory)
                           │
                           ▼
                  rmax_out_create_stream
                           │ packets timed by NIC
                           ▼
                ConnectX NIC ──── 100GbE ──── ConnectX NIC ── decoder
```

### Stream creation (HEVC over RTP)

```c
rmax_init();                        // process-wide
rmax_clock_t clock;
rmax_init_ptp_clock(&clock, ifIndex);   // PTP-disciplined NIC clock

// Output stream: 25 Mbps HEVC, 60 fps, RTP payload 96
rmax_out_stream_params params = {
    .local_addr   = "10.0.1.10",
    .remote_addr  = "239.1.2.3",       // multicast or unicast
    .remote_port  = 50000,
    .pcp          = 0,                 // PCP/DSCP
    .num_packets  = 4096,              // ring depth (~70ms at 60Hz)
    .packet_size  = 1376,              // RTP payload size for 1500 MTU
    .num_chunks   = 256,
    .clock        = clock,
    .pacing_type  = RMAX_PACING_TYPE_NIC_HW,    // HW-timed
};
rmax_stream_id stream;
rmax_status_t s = rmax_out_create_stream(&params, &stream);
```

### Per-frame: chunk submit

Encoder bitstream (~50KB per frame at 25Mbps/60fps) is segmented into RTP packets
and submitted as a chunk:

```c
// 1. Get NVENC bitstream pointer (already on GPU when using GPUDirect)
nvEncLockBitstream(nvEncoder, &lock);

// 2. Build RTP headers in pinned host memory (CPU-side)
BuildRtpHeaders(headers, lock.bitstreamSizeInBytes, sequenceNumber, timestamp);

// 3. Submit chunk; payload is GPU-resident, headers are CPU-resident
rmax_packet_t pkts[N_PACKETS];
for (int i = 0; i < n; ++i) {
    pkts[i].sub_blocks[0].addr   = headers + i*sizeof(RtpHeader);   // CPU
    pkts[i].sub_blocks[0].length = sizeof(RtpHeader);
    pkts[i].sub_blocks[1].addr   = (char*)gpu_bitstream + i*payloadSize;  // GPU
    pkts[i].sub_blocks[1].length = payloadSize;
}
rmax_commit_chunks(stream, pkts, n,
                   /*timestamp*/ frameTime,
                   RMAX_COMMIT_F_PACE);

nvEncUnlockBitstream(nvEncoder, bitstreamBuf);
```

### SMPTE 2110-20 (uncompressed video)

For broadcast-grade uncompressed video, skip the encoder entirely:

```c
rmax_out_stream_params params2110 = {
    // ...
    .video_format = {
        .width        = 1920,
        .height       = 1080,
        .frame_rate   = { .num = 60000, .den = 1001 },   // 59.94
        .pix_fmt      = RMAX_OUT_PIXEL_FMT_YUV422_10,    // 10-bit YCbCr 4:2:2
        .scan         = RMAX_OUT_SCAN_PROGRESSIVE,
    },
    .pacing_type  = RMAX_PACING_TYPE_NIC_HW,
};
```

Bandwidth: 1080p60 4:2:2 10-bit = ~2.5 Gbps. 4K60 = ~10 Gbps. This is why
Rivermax targets 25-100 GbE NICs.

### 2022-7 redundancy

Open the same logical stream on two NICs with `redundant_stream_params`:

```c
rmax_out_stream_redundant_params red = {
    .params[0] = primaryParams,    // NIC A → switch fabric A
    .params[1] = secondaryParams,  // NIC B → switch fabric B
};
rmax_out_create_redundant_stream(&red, &stream);
```

Receiver merges using RTP sequence number — single-path packet loss is invisible.

### Threading

Rivermax callbacks (completion notifications) run on internal threads. Don't
touch IDD swap-chain or D3D resources from the callback. Use it only to recycle
encoder buffers back to the encoder pool.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| GPU and NIC on different PCIe roots | GPUDirect falls back to host bounce buffer, latency doubles | Pin both under same root complex; verify with `lspci -t` |
| `pacing_type = SOFTWARE` in production | Defeats the point of Rivermax | Always `RMAX_PACING_TYPE_NIC_HW` |
| Forgetting PTP discipline | Timestamps drift from broadcast clock | Run `linuxptp` / Windows W32Time PTP and seed `rmax_init_ptp_clock` |
| 9000-byte jumbo frames not enabled end-to-end | NIC fragments → throughput drops | Configure jumbo on NICs, switches, and OS interfaces |
| Holding NVENC `LockBitstream` across rmax submit | Encoder buffer pool stalls | Copy header info, unlock, then commit |

## See also

- https://docs.nvidia.com/networking/display/rivermaxv130/index.html
- https://www.smpte.org/standards/st2110
- https://docs.nvidia.com/cuda/gpudirect-rdma/index.html
