# Braswell and the insane quest for Un-accelerated macOS Mojave

AKA: a writeup on the things that went down to get an unsupported CPU booting macOS Mojave

Legitimately don't do this unless you're looking for suffering.

Table Of Contents:
- [The Beginning](#the-beginning)
  - [OC Setup](#oc-setup)
- [CPUID Shenanigans](#cpuid-shenanigans)
- [Storage? What Storage?](#storage-what-storage)
- [MSRs?](#msrs)
- [Ending Notes](#ending-notes)

## The Beginning

### OC Setup

This whole thing is tied together very loosely, don't expect it to work right off the bat

SSDTs:
- EC
- HPET
- PLUG
- PNLF (addendum: this does nothing)

Booter Quirks:
| Quirks                 | Active?    |
| ---------------------- |:----------:|
| AvoidRuntimeDefrag     | Yes        |
| EnableSafeModeSlide    | Yes        |
| EnableWriteUnprotector | Yes        |
| ProvideCustomSlide     | Yes        |
| SetupVirtualMap        | Yes        |

All others NOT on this list are either:
- Set to False
- Set to their default values.

Kernel Quirks:
| Quirks                   | Active?    |
| :----------------------- |:----------:|
| AppleXcpmCfgLock         | Yes        |
| DisableLinkeditJettison  | Yes        |
| LapicKernelPanic         | Yes        |
| PanicNoKextDump          | Yes        |
| PowerTimeoutKernelPanic  | Yes        |
| ProvideCurrentCpuInfo    | Yes        |

All others NOT on this list are either:
- Set to False
- Set to their default values.

ProvideCurrentCpuInfo is needed because we are *running on an unsupported processor*

## CPUID shenanigans

Before doing this, I already knew I'd have to dabble with some shenanigans pertaining to the CPUID.
This led me to going on a rabbit hole, eventually stumbling on [this helpful Acidanthera Bug-Tracker post](https://github.com/acidanthera/bugtracker/issues/365).

After having done the spoof and adding the patches, the kernel should've been fine.

## Booter Problems

Initially, I was skeptical about even getting it to boot `boot.efi` at all.
And I was right.

OpenCore at that time didn't have support for fetching the FSB Frequency for Braswell platforms, meaning it legitimately would not get past patching XNU.

Dead end, right?

Thanks to [Goldfish64's awesome work](https://github.com/acidanthera/OpenCorePkg/commit/dc5165b660fde9fe7ceba16de2db3bbf7fd94c73), the system subsequently managed to boot macOS Mojave's recoveryOS

## Storage? What Storage?

As soon as I thought it couldn't get worse, it did.

Earlier Intel Platforms such as Bay Trail and Braswell have their eMMC cards running over ACPI as opposed to PCI.
This was an annoying, albeit temporary issue.

Support for said controllers was merged in EmeraldSDHC in [this PR](https://github.com/acidanthera/EmeraldSDHC/commit/7a0535097055b7b53f2456af692f7d596546f318).

Within the last several days I just realized an oversight in my PR.
I'll fix it up at some point; I need to figure out a couple of things.

The gist of it is that it's prone to attaching to `PNP0D40` IDs that are SDIO/not eMMC devices

After having gone through that whole process, I was finally on my way to install macOS Mojave!

## MSRs?

Having finally managed to Install the OS to the disk, one would think that the first-time setup would just run, right?

Think again.
An MSR read in XNU's `monotonic` subsystem was causing a general protection fault, leaving the OS to hang.
You can read the mentioned source code [here](https://github.com/apple-oss-distributions/xnu/blob/main/osfmk/x86_64/monotonic_x86_64.c)

The solution?

Patch out the part that sets `mt_core_supported` to true.

As I found out, the routine that does it is inlined into `_vstart` as of macOS Mojave's copy of XNU.

On later macOS versions, said logic can be disabled using the `-nomt_core` boot argument!

The patch used for macOS Mojave is as follows:
```
	<dict>
		<key>Comment</key>
		<string>Force mt_core_supported as false</string>
		<key>Arch</key>
		<string>x86_64</string>
		<key>Base</key>
		<string>_vstart</string>
		<key>Count</key>
		<integer>1</integer>
		<key>Enabled</key>
		<true/>
		<key>Find</key>
		<data>xgUAAAAAAQ==</data>
		<key>Identifier</key>
		<string>kernel</string>
		<key>Limit</key>
		<integer>0</integer>
		<key>Mask</key>
		<data>//8AAAAA/w==</data>
		<key>MaxKernel</key>
		<string>18.99.99</string>
		<key>MinKernel</key>
		<string>18.0.0</string>
		<key>Replace</key>
		<data>AAAAAAAAAA==</data>
		<key>ReplaceMask</key>
		<data>AAAAAAAA/w==</data>
		<key>Skip</key>
		<integer>0</integer>
	</dict>
```

Other attempts at fixing this involed setting `mt_early_init` to flat-out return on call, which was found to be ineffective.

## Ending Notes

In conclusion, nightmares are real and this device is it.