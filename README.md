<!-- Copyright 2013-2020 Mark Callow -->
<!-- Copyright 2016-2026 Binomial LLC -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# KTX-Software-Binomial-Fork (Bug/Security Fixes, and New Basis Universal GPU Texture Codecs)

*This fork serves as a staging ground for the next generation of GPU texture codecs.*

This is an unofficial, live fork of [KTX-Software](https://github.com/KhronosGroup/KTX-Software) by [Khronos Group member](https://www.khronos.org/) Binomial LLC, with key bug fixes and (soon) new GPU supercompressed texture codecs such as XUASTC LDR. The original project is maintained by the Khronos Group. This fork will be periodically synced vs. KTX-Software. This repo has been integrated against the [Basis Universal](https://github.com/BinomialLLC/basis_universal/) v1.6 library (a snapshot from late 2025), and we're working on integrating the latest version (which is now v2.1). 

See our [Release Notes on our main repo here](https://github.com/BinomialLLC/basis_universal/wiki/Release-Notes), which we're keeping in sync with this repo. Also see this [repo's wiki](https://github.com/BinomialLLC/KTX-Software-Binomial-Fork/wiki), where we'll document how to use the new codecs we'll be adding to this software.

The primary changes we've made to this repository are:
- Live back porting, integration and testing of security fixes (such as [this one](https://github.com/BinomialLLC/basis_universal/commit/58e3afbabae592e97e6a736e0908c03bc7a4dd4f)) directly from the main [Basis Universal repo](https://github.com/BinomialLLC/basis_universal).
  
- The file [`lib/src/basis_encode.cpp`](https://github.com/BinomialLLC/KTX-Software-Binomial-Fork/blob/main/lib/src/basis_encode.cpp): This fixes the `ktx` (renamed to `btx` in this repo) tool's `--uastc-quality` and `--uastc-hdr-6x6i-level` options, which are broken in KTX-Software. For more technical information, see our [KTX2 File Format Support Technical Details](https://github.com/BinomialLLC/basis_universal/wiki/KTX2-File-Format-Support-Technical-Details#intro) wiki.

- **The `ktx` tool has been renamed to `btx` in help text and [`tools/ktx/CMakeLists.txt`](https://github.com/BinomialLLC/KTX-Software-Binomial-Fork/blob/main/tools/ktx/CMakeLists.txt).** So wherever you would run `ktx`, you can now run `btx`.  **Note `btx` is not intended to be used for KTX2 specification validation purposes, so there is no `--validate` command in this fork. Only `ktx` can do this, which [lives upstream of this fork](https://github.com/KhronosGroup/KTX-Software).**
 
- We've added the `"BINOMIAL FORK"` text to the `btx` tool's `--help` text, in source file [‎`tools/ktx/ktx_main.cpp`](https://github.com/BinomialLLC/KTX-Software-Binomial-Fork/blob/main/tools/ktx/ktx_main.cpp), to clearly indicate to users that they're using our forked version of the command line tool.
- The `create` command now supports `--verbose` and `--debug` options, so we can validate proper command line parsing/codec parameter passing to our codecs.
 
Other improvements and fixes will be made as we find them. We will be integrating and testing the new [XUASTC LDR codec](https://github.com/BinomialLLC/basis_universal/wiki/JPEG-for-ASTC) (shipped in Basis Universal v2.0) into this fork very soon, as indicated to Khronos in an official meeting on 3/7/2026. We are following the project's [original CONTRIBUTING.md file](https://github.com/KhronosGroup/KTX-Software/blob/main/CONTRIBUTING.md), which requires a fork. 

# License and Legal

This independent fork is not endorsed by or an official product of the Khronos Group. Khronos® is a registered trademark of The Khronos Group Inc. This software is Copyright 2015-2026 [The Khronos Group Inc.](https://www.khronos.org/) KTX™ is a trademark of The Khronos Group Inc. See the project's Apache 2.0 [license information](https://github.com/BinomialLLC/KTX-Software-Binomial-Fork/tree/main?tab=License-1-ov-file#readme), and see the actual [LICENSE here](https://github.com/BinomialLLC/KTX-Software-Binomial-Fork/blob/main/LICENSES/Apache-2.0.txt). Basis Universal™ is a trademark of Binomial LLC.

All changes to the original software have been prominently indicated with comments, as required by the Apache License, Version 2.0 §4(b). Our modifications are licensed under the Apache License, Version 2.0. 

Binomial LLC is a Khronos Group Member, and contributions by Members to products based on Khronos Ratified Specifications are covered under the [Khronos Membership Agreement's](https://www.khronos.org/files/member_agreement.pdf) IP Rights Policy (Attachment A, Sections 2.1 and 2.2).

# What Motivated This Fork

This fork exists to ensure that security fixes and new Basis Universal codecs reach developers as quickly as possible, without being bottlenecked by the upstream contribution timeline.

# Quick Build Instructions

The original project's build instructions are incomplete, at least for Windows. You have to add `-D KTX_FEATURE_TOOLS=ON` to build the original KTX-Software and get usable command line tools, which we've fixed in our fork.

Use [cmake](https://cmake.org/):

```
mkdir build
cd build
cmake ..
```

Then, under Linux or OSX:

```
make
```

Under Windows, `KTX-Software.sln` can then be used to build the solution using Visual Studio. The tool for introspecting, compression and extraction of KTX2 files is `btx`, which will be placed in the `build/Debug` or `build/Release` directories under Windows, or in the build directory on other platforms. See the tool's original [documentation here](https://github.khronos.org/KTX-Software/ktxtools/index.html).

Running `btx --help` will display `"[BINOMIAL FORK]"` in the help text, which is how you can tell you're running a version of `btx` with Binomial's fixes and improvements:

```
build\Debug>btx --help
btx v0.10 [BINOMIAL FORK]. Not for KTX2 specification validation purposes.
This fork is only for codec and interoperability testing. See:
https://github.com/BinomialLLC/KTX-Software-Binomial-Fork

btx: Unified CLI frontend for the KTX-Software [BINOMIAL FORK] library with sub-commands for specific operations.

Usage:
  btx [<command>] [OPTION...]

  -h, --help     Print this usage message and exit
  -v, --version  Print the version number of this program and exit
      --testrun  Indicates test run. If enabled the tool will produce deterministic output whenever
                 possible

Available commands:
  convert    Convert another texture file type to a KTX2 file
  create     Create a KTX2 file from various input files
  deflate    Deflate (supercompress) a KTX2 file
  encode     Encode a KTX2 file
  extract    Extract selected images from a KTX2 file
  transcode  Transcode a KTX2 file
  info       Print information about a KTX2 file
  compare    Compare two KTX2 files
  help       Display help information about the btx tool

For detailed usage and description of each subcommand use 'btx help <command>'
or 'btx <command> --help'
```

# KTX2 Creation Command Line Examples

You can download the `desk.exr` HDR test image on [GitHub here](https://github.com/richgel999/junkdrawer/blob/main/Desk.exr).

## UASTC HDR 4x4

Verbose output, mipmaps, Zstd supercompression level 2, UASTC HDR 4x4 level 3 (valid range [0,4]):
 
```
btx create --verbose --format R16G16B16_SFLOAT --generate-mipmap --zstd 2 --encode uastc-hdr-4x4 desk.exr file.ktx2 --uastc-quality 3
```

Note UASTC HDR 4x4 texture data is just standard ASTC HDR 4x4 with added constraints to permit fast direct (latent to latent) transcoding to BC6H (unsigned).

## UASTC HDR 6x6i

Verbose output, mipmaps, UASTC HDR 6x6i level 4 (valid range is [0,12]), lambda 500 (valid range is 0 or none to 10's of thousands or higher):

```
btx create --verbose --format R16G16B16_SFLOAT --generate-mipmap --encode uastc-hdr-6x6i desk.exr file.ktx2 --uastc-hdr-6x6i-level 4 --uastc-hdr-lambda 500
```

The generated KTX2 files can then be unpacked, or validated etc. using the `basisu` command line tool, or transcoded to other GPU texture formats using our transcoder module. Binomial has [documented here](https://github.com/BinomialLLC/basis_universal/wiki/KTX2-File-Format-Support-Technical-Details#intro) exactly how the KTX2 file format is utilized by our codecs.

Note Basis Universal v2.0 introduced unified/simplified "effort" and "quality" options which work across all of our codecs, which are coming to this fork soon.

# KTX2 Validation Command Line Examples

## UASTC HDR 4x4

```
build\Debug>btx info file.ktx2
btx v0.10 [BINOMIAL FORK]. Not for KTX2 specification validation purposes.
This fork is only for codec and interoperability testing. See:
https://github.com/BinomialLLC/KTX-Software-Binomial-Fork

Checking successful

Header

identifier: «KTX 20»\r\n\x1A\n
vkFormat: VK_FORMAT_ASTC_4x4_SFLOAT_BLOCK
typeSize: 1
pixelWidth: 644
pixelHeight: 874
pixelDepth: 0
layerCount: 0
faceCount: 1
levelCount: 10
supercompressionScheme: KTX_SS_ZSTD
dataFormatDescriptor.byteOffset: 0x140
dataFormatDescriptor.byteLength: 44
keyValueData.byteOffset: 0x16c
keyValueData.byteLength: 164
supercompressionGlobalData.byteOffset: 0
supercompressionGlobalData.byteLength: 0

Level Index

Level0.byteOffset: 0x2d00f
Level0.byteLength: 539025
Level0.uncompressedByteLength: 564144
Level1.byteOffset: 0xb8a3
Level1.byteLength: 137068
Level1.uncompressedByteLength: 142560
Level2.byteOffset: 0x305b
Level2.byteLength: 34888
Level2.uncompressedByteLength: 36080
Level3.byteOffset: 0xe58
Level3.byteLength: 8707
Level3.uncompressedByteLength: 8960
Level4.byteOffset: 0x58e
Level4.byteLength: 2250
Level4.uncompressedByteLength: 2240
Level5.byteOffset: 0x354
Level5.byteLength: 570
Level5.uncompressedByteLength: 560
Level6.byteOffset: 0x28b
Level6.byteLength: 201
Level6.uncompressedByteLength: 192
Level7.byteOffset: 0x242
Level7.byteLength: 73
Level7.uncompressedByteLength: 64
Level8.byteOffset: 0x229
Level8.byteLength: 25
Level8.uncompressedByteLength: 16
Level9.byteOffset: 0x210
Level9.byteLength: 25
Level9.uncompressedByteLength: 16

Data Format Descriptor

DFD total bytes: 44
Vendor ID: KHR_DF_VENDORID_KHRONOS
Descriptor type: KHR_DF_KHR_DESCRIPTORTYPE_BASICFORMAT
Version: KHR_DF_VERSIONNUMBER_1_3
Descriptor block size: 40
Flags: 0x0 (KHR_DF_FLAG_ALPHA_STRAIGHT)
Transfer: KHR_DF_TRANSFER_LINEAR
Primaries: KHR_DF_PRIMARIES_BT709
Model: KHR_DF_MODEL_UASTC_HDR_4x4
Dimensions: 4, 4, 1, 1
Plane bytes: 16, 0, 0, 0, 0, 0, 0, 0
Sample 0:
    Qualifiers: 0x80 (KHR_DF_SAMPLE_DATATYPE_FLOAT)
    Channel Type: 0x0 (KHR_DF_CHANNEL_UASTC_HDR_4x4_RGB)
    Length: 128 bits Offset: 0
    Position: 0, 0, 0, 0
    Lower: 0x00000000
    Upper: 0x3f800000

Key/Value Data

KTXmapRange:
    scale: 1.000000
    offset: 0.000000
KTXwriter: btx create v4.4.2-167-g6b635031-dirty / libktx v4.4.2-119-g3422e2b5-dirty
KTXwriterScParams: --uastc-quality 3 --zstd 2
```

## UASTC HDR 6x6i

```
build\Debug>btx info file.ktx2
btx v0.10 [BINOMIAL FORK]. Not for KTX2 specification validation purposes.
This fork is only for codec and interoperability testing. See:
https://github.com/BinomialLLC/KTX-Software-Binomial-Fork

Checking successful

Header

identifier: «KTX 20»\r\n\x1A\n
vkFormat: VK_FORMAT_UNDEFINED
typeSize: 1
pixelWidth: 644
pixelHeight: 874
pixelDepth: 0
layerCount: 0
faceCount: 1
levelCount: 10
supercompressionScheme: KTX_SS_UASTC_HDR_6x6_INTERMEDIATE
dataFormatDescriptor.byteOffset: 0x140
dataFormatDescriptor.byteLength: 44
keyValueData.byteOffset: 0x16c
keyValueData.byteLength: 184
supercompressionGlobalData.byteOffset: 0x228
supercompressionGlobalData.byteLength: 120

Level Index

Level0.byteOffset: 0x12518
Level0.byteLength: 215111
Level0.uncompressedByteLength: 0
Level1.byteOffset: 0x4dc5
Level1.byteLength: 55123
Level1.uncompressedByteLength: 0
Level2.byteOffset: 0x168d
Level2.byteLength: 14136
Level2.uncompressedByteLength: 0
Level3.byteOffset: 0x7f0
Level3.byteLength: 3741
Level3.uncompressedByteLength: 0
Level4.byteOffset: 0x463
Level4.byteLength: 909
Level4.uncompressedByteLength: 0
Level5.byteOffset: 0x33c
Level5.byteLength: 295
Level5.uncompressedByteLength: 0
Level6.byteOffset: 0x2dc
Level6.byteLength: 96
Level6.uncompressedByteLength: 0
Level7.byteOffset: 0x2c5
Level7.byteLength: 23
Level7.uncompressedByteLength: 0
Level8.byteOffset: 0x2ae
Level8.byteLength: 23
Level8.uncompressedByteLength: 0
Level9.byteOffset: 0x2a0
Level9.byteLength: 14
Level9.uncompressedByteLength: 0

Data Format Descriptor

DFD total bytes: 44
Vendor ID: KHR_DF_VENDORID_KHRONOS
Descriptor type: KHR_DF_KHR_DESCRIPTORTYPE_BASICFORMAT
Version: KHR_DF_VERSIONNUMBER_1_3
Descriptor block size: 40
Flags: 0x0 (KHR_DF_FLAG_ALPHA_STRAIGHT)
Transfer: KHR_DF_TRANSFER_LINEAR
Primaries: KHR_DF_PRIMARIES_BT709
Model: KHR_DF_MODEL_UASTC_HDR_6x6
Dimensions: 6, 6, 1, 1
Plane bytes: 16, 0, 0, 0, 0, 0, 0, 0
Sample 0:
    Qualifiers: 0x80 (KHR_DF_SAMPLE_DATATYPE_FLOAT)
    Channel Type: 0x0 (KHR_DF_CHANNEL_UASTC_HDR_6x6_RGB)
    Length: 128 bits Offset: 0
    Position: 0, 0, 0, 0
    Lower: 0x00000000
    Upper: 0x3f800000

Key/Value Data

KTXmapRange:
    scale: 1.000000
    offset: 0.000000
KTXwriter: btx create v4.4.2-167-g6b635031-dirty / libktx v4.4.2-119-g3422e2b5-dirty
KTXwriterScParams: --uastc-hdr-lambda 500 --uastc-hdr-6x6i-level 4

UASTC HDR6X6 Intermediate Supercompression Global Data


rgbSliceByteLength: 215111
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd

rgbSliceByteLength: 55123
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd

rgbSliceByteLength: 14136
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd

rgbSliceByteLength: 3741
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd

rgbSliceByteLength: 909
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd

rgbSliceByteLength: 295
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd

rgbSliceByteLength: 96
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd

rgbSliceByteLength: 23
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd

rgbSliceByteLength: 23
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd

rgbSliceByteLength: 14
rgbSliceByteOffset: 0
rgbSliceType: 0xabcd
```

# KTX2 Extraction Command Line Example

This will extract all mipmap files to image files in the "mips" directory. For the .EXR file format, you can use the [OpenHDR online image viewer](https://viewer.openhdr.org/) for quick viewing.

```
btx extract --all file.ktx2 mips
```

----

E-mail: info @ binomial dot info, or contact us on [Twitter](https://twitter.com/_binomial)
