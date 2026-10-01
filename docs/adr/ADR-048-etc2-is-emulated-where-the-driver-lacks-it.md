# ADR-048: ETC2 is emulated where the driver lacks it

**Status:** accepted
**Amends:** [ADR-042](ADR-042-texture-format-query-observability.md) ("nothing is translated")
**Related:** [ADR-001](ADR-001-in-process-hooking.md), [ADR-046](ADR-046-nvidia-is-gated-on-the-vendor-id.md)

## Decision

When a Vulkan device reports `textureCompressionETC2 = false`, Cordial reports
it as true to the engine and emulates it at the Vulkan boundary: ETC2/EAC images
are created as RGBA8 (R16/RG16 for EAC) and their uploads are decoded on the CPU
before the copy is recorded. Devices that support ETC2 natively are not touched,
and their device-level calls are not routed through the emulation.

`CORDIAL_NO_ETC_EMULATION=1` turns it off. `CORDIAL_FORCE_ETC_EMULATION=1` turns
it on even where ETC2 is native, so the decoded path can be compared against the
driver on the same machine.

## What was measured

On an NVIDIA RTX 5070 (driver 615.71.09) the driver reports
`textureCompressionETC2 = false`, and the engine logs
`Caps: Texture: DXT 1 PVR 0 ETC1 0 ETC2 0`. Its ETC1 built-in textures (plastic
detail, sky, water) are then never used, and studs and some materials go
missing on Vulkan while GLES renders them. With emulation on, the same line
reads `ETC1 1 ETC2 1`; with `CORDIAL_NO_ETC_EMULATION=1` it reads `ETC1 0 ETC2 0`
again.

The engine decides this from the `textureCompressionETC2` feature, not from the
per-format query. That is why ADR-042's mask, which only touched
`vkGetPhysicalDeviceFormatProperties`, left the caps line unchanged on Intel:
it never reproduced the NVIDIA state, so its result said nothing about this.

## Why it is gated on the feature, not on the vendor id

ADR-046 gates NVIDIA-specific behaviour on `vendorID`, because those workarounds
are about NVIDIA's driver. This one is not. What matters is whether the device
can sample ETC2, and the driver says so directly in the feature it reports.
Gating on the vendor would emulate on an NVIDIA device that gains native ETC2,
and miss any other driver that reports it false. Reporting the feature as true
is only acceptable because the format is then actually provided; where a path
cannot be emulated, the call fails loudly instead of reaching the driver.

## Costs

Decoding is done once per upload. On the signed-out landing screen it covers 16
images and 150 regions, about 0.3 s in total, with a staging peak of about
42 MiB, released when the command buffer is reset or freed.

## What this does not settle

ASTC is not emulated: the APK ships no ASTC textures and streamed content
arrives as BC, which NVIDIA supports. EAC, cube and array ETC images are
handled, but the engine was not seen creating one.
