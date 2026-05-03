# Windows Drivers - Indirect Display - EDID

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/display/edid-extension-blocks
> **Skill**: dev-suite skill `windows/indirect-display` — see SKILL.md for the always-loaded quick reference.

## What this covers

The byte layout of an EDID 1.4 base block, the CEA-861 extension format used to
declare HDMI/HDR/audio capabilities, the HDR Static Metadata Data Block, and the
checksum formula. The SKILL warns that bogus EDIDs cause weird modes; this file
shows how to build one that the OS accepts as a real monitor.

## Deep dive

### Base 128-byte EDID 1.4

| Offset | Length | Field |
|--------|--------|-------|
| 0x00 | 8 | Header (00 FF FF FF FF FF FF 00) |
| 0x08 | 2 | Manufacturer ID (3 letters, 5 bits each, big-endian) |
| 0x0A | 2 | Product code (LE) |
| 0x0C | 4 | Serial number (LE) |
| 0x10 | 1 | Manufacture week (1-54), or 0xFF for model year |
| 0x11 | 1 | Manufacture year - 1990, or model year - 1990 |
| 0x12 | 1 | EDID version (1) |
| 0x13 | 1 | EDID revision (4 for v1.4) |
| 0x14 | 1 | Video input definition (digital: bit 7=1; HDMI sets bit 1) |
| 0x15 | 1 | Max horizontal image size (cm) or 0 for projector |
| 0x16 | 1 | Max vertical image size (cm) |
| 0x17 | 1 | Display gamma ((value+100)/100) |
| 0x18 | 1 | Feature support (DPMS, color type, default GTF/sRGB) |
| 0x19-0x22 | 10 | Color characteristics (CIE coords) |
| 0x23-0x25 | 3 | Established timings 1, 2, 3 (legacy mode bitmasks) |
| 0x26-0x35 | 16 | Standard timings (8 entries x 2 bytes) |
| 0x36-0x47 | 18 | Detailed timing 1 (preferred, 18 bytes) |
| 0x48-0x5A | 18 | Detailed timing 2 |
| 0x5B-0x6D | 18 | Detailed timing 3 (or descriptor) |
| 0x6E-0x80 | 18 | Detailed timing 4 (or descriptor) |
| 0x7E | 1 | Number of extension blocks |
| 0x7F | 1 | Checksum: 256 - sum(bytes 0..126) mod 256 |

Manufacturer ID encoding for "MSF": each letter is letter-'A'+1 in 5 bits, packed
big-endian: M=13, S=19, F=6 → bits `01101 10011 00110` → bytes `0x6D, 0x66`.

### Detailed Timing Descriptor (DTD)

18 bytes describing one mode in pixel-clock granularity:

```c
struct DTD {
    uint16_t pixel_clock_10khz;     // pixel rate in 10kHz units (148.5MHz = 14850)
    uint8_t  h_active_lo;
    uint8_t  h_blank_lo;
    uint8_t  h_active_hi_blank_hi;  // upper nibble: hactive[11:8], lower: hblank[11:8]
    uint8_t  v_active_lo;
    uint8_t  v_blank_lo;
    uint8_t  v_active_hi_blank_hi;
    uint8_t  h_sync_offset_lo;
    uint8_t  h_sync_pulse_lo;
    uint8_t  v_sync_offset_pulse_lo;
    uint8_t  sync_offsets_hi;
    uint8_t  h_image_size_mm_lo;
    uint8_t  v_image_size_mm_lo;
    uint8_t  image_sizes_hi;
    uint8_t  h_border;
    uint8_t  v_border;
    uint8_t  flags;                 // sync polarity, interlace
};
```

For a virtual display, blanking can be small; many drivers pick CVT-RB (Coordinated
Video Timings - Reduced Blanking) values to be safe with strict OSes.

### CEA-861 extension block (128 bytes)

When `extensions = 1` at offset 0x7E, append a CEA block at offset 0x80:

| Offset | Field |
|--------|-------|
| 0x00 | 0x02 (CEA tag) |
| 0x01 | Revision (3 for CEA-861-D, 4 for HDR support) |
| 0x02 | DTD start offset within this block |
| 0x03 | Native format flags + #DTDs |
| 0x04..d-1 | Data Block Collection (audio, video, vendor, HDR static metadata) |
| d..0x7E | Detailed timings |
| 0x7F | Checksum: 256 - sum(bytes 128..254) mod 256 |

Each Data Block in the collection has a tag/length header byte:

```
Bits 7-5: Tag    (1=Audio, 2=Video, 3=Vendor, 4=Speaker Allocation,
                  7=Use Extended Tag)
Bits 4-0: Length (bytes following the tag byte)
```

### HDR Static Metadata Data Block

Tag 7, extended tag 6. Declares which HDR transfer functions the panel can decode:

```
Byte 0: 0xE0 | 0x06   (extended tag=6 means HDR static metadata)
Byte 1: 0x06          (extended tag value)
Byte 2: ET (electrooptical transfer functions): bit0=SDR, bit1=HDR Trad,
        bit2=PQ (HDR10), bit3=HLG
Byte 3: SM (static metadata types): bit0=Type 1
Byte 4: Desired max luminance (50 * 2^(N/32) cd/m²)
Byte 5: Desired max frame-average luminance
Byte 6: Desired min luminance (0.0001 * 50 * 2^(N/255) cd/m²)
```

Set bit 2 (PQ) of ET for Windows HDR to recognize the monitor as HDR-capable.
DWM will then offer the "Use HDR" toggle in Settings → System → Display.

### Checksum verification

```c
uint8_t checksum(const uint8_t* block, size_t n) {
    uint8_t s = 0;
    for (size_t i = 0; i < n - 1; ++i) s += block[i];
    return (uint8_t)(256 - s);   // s + this == 0 mod 256
}
```

Compute and write into byte 127 of the base block, byte 127 of each extension.
Off-by-one here is the #1 reason Windows refuses to recognize an EDID.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Wrong manufacturer ID encoding | Not bytewise; it's 3x5 bits big-endian | Use a tested helper: `MFG('M','S','F')` macro |
| Forgetting to bump `extensions` byte to 1 when adding CEA block | Driver gets EDID parser failure, falls back to base modes | Set offset 0x7E correctly |
| Bad checksum | OS treats EDID as invalid, generates synthetic | Recompute on every modification |
| Declaring HDR in CEA but not handling 10-bit swap chain in driver | DWM tries to send `R10G10B10A2_UNORM` and your encoder fails | Either declare HDR + handle, or skip the HDR block |
| Detailed timing pixel clock = 0 | Some drivers crash | Always set to a real value (e.g. 148500 for 1080p60) |

## See also

- https://www.vesa.org/wp-content/uploads/2010/12/thumb_DisplayID-v13.pdf (DisplayID is EDID's successor)
- https://glenwing.github.io/docs/CTA-861-G.pdf (CEA-861 spec)
- https://learn.microsoft.com/en-us/windows-hardware/drivers/display/wdk-edid
