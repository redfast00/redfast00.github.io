---
layout: post
title:  "Reverse engineering UEFI firmware updater"
date:   2019-12-31 00:00:00 +0000
category: Hacking
tags: [Reversing, UEFI]
---

According to Wikipedia, UEFI is the layer between the firmware and the operating
system on a computer. Since this is just a binary blob, I was interested in finding out what
it actually does. More concrete, my goal is to see or change what the Fn-key does
on my keyboard (it's not a regular key, since the keypresses don't get registered in
xev, and it changes the backlight if pressed together with space).
Additionally, I'd like to figure out if it is possible to control the
backlight from software instead of by pressing Fn+space.

Another goal is to find out if the code running on my laptop is secure.

I wanted to reverse engineer the UEFI code running on my own laptop (a Lenovo Y520-15IKBN).
To do this, I will first need to obtain the firmware. This should be extractable from
a firmware update, so I downloaded the firmware updater from the Lenovo support site.
The firmware version I analyze in this blog post is 4KCN45WW:

```
#> sha256sum bios_update.exe 
a85e7a8eccee6966fc4366a20ee49e5f08655ac9af72004bb485b1494fa2fee7  bios_update.exe
```

# Packed executable part 1: InnoSetup

The firmware update downloaded from the site is a Windows executable, and not the actual firmware.
After looking through
the strings in the executable (with `rabin2 -zzqq`), I came across the string
`This installation was built with Inno Setup`. The binary was packed with InnoSetup,
so I unpacked it with [innounp](http://innounp.sourceforge.net/). This resulted
in another binary (`4KCN45WW.exe`) and a bunch of embedded files, among them `CompiledCode.bin`.

The binary file `CompiledCode.bin` started with `IFPS`, and after some research,
I found out that it was a compiled PascalScript executable. I reverse engineered
this with [IFPSTools](https://github.com/Wack0/IFPSTools). It turned out that this
just contained a check if the BIOS can be flashed by checking the current version of
the BIOS in the registry: it checked the `BIOSVersion` key in `HARDWARE\DESCRIPTION\System\BIOS`.

{% gist redfast00/40e4877343a5045a928bc0514d465a7d %}

It then just executed the embeddded executable (`4KCN45WW.exe`) without any arguments.

# Packed executable part 2: 7z SFX

The second executable was a 7zip self-extracting archive (I determined this by finding the
string "Igor Pavlov", the author of 7zip, in the binary). It can be extracted with
the `7z` commandline tool. However, this only unpacks the files, and doesn't show
what happens with them after they are unpacked. The configuration for the unpacker
can be found by searching for the string `!@Install@!UTF-8!` in the executable:

```
!@Install@!UTF-8!
RunProgram="H2OFFT-W.exe -sfx7z %%S execApp "
Protect="no"
;!@InstallEnd@!
```

I found [some documentation](https://sevenzip.osdn.jp/chm/cmdline/switches/sfx.htm)
of what these parameters do online: after unpacking, it runs the program in the RunProgram key,
replacing `%%S` with the archive path.

The other files in the archive were:

```
├── BIOS.fd
├── BiosImageProc.dll
├── Ding.wav
├── FlsHook.exe
├── FWUpdLcl.exe
├── H2OFFT-W.exe
├── iscflash.sys
├── iscflashx64.sys
├── mfc90u.dll
├── Microsoft.VC90.CRT.manifest
├── Microsoft.VC90.MFC.manifest
├── msvcp90.dll
├── msvcr90.dll
└── platform.ini
```

`BIOS.fd` seemed interesting, because `rabin2 -I` said it was a PE EFI Application.

# BIOS.fd EFI updater

I first opened the file in the opensource reverse engineering tool
[Ghidra](https://github.com/NationalSecurityAgency/ghidra). Since UEFI executables
are just PE files (the filetype for binaries on Windows), Ghidra has support for them.
However, Ghidra doesn't really have support for the datatypes used in these binaries,
but support is on the way in [this pull request](https://github.com/NationalSecurityAgency/ghidra/pull/501).
For the time being, I just downloaded the `.gdt` files (archives containing type information) from the PR and imported them in Ghidra.

After some reversing in Ghida, it seemed that the UEFI application was just an updater
for the BIOS.

I then looked through the strings in the application, and found some strings that started with `$_IFLASH_`:
this [blog post about Insyde BIOS](https://antoniovazquezblanco.github.io/blog/New-InsydeH2O-BIOS-update-format.html) says that this is a file that consists of multple sections, each starting
with a string that begins with `$_IFLASH_` and is 16 characters long. I then adapted
the Python script used to extract these images (my image had different sections).

This resulted in some more files, which I ran the `file` command on:

```
BEFORE_TAGS:                MS-DOS executable
_IFLASH_BIOSCER:            data
_IFLASH_BIOSCR2:            data
_IFLASH_BIOSIMG:            Intel serial flash for PCH ROM
_IFLASH_DRV_IMG:            MS-DOS executable
_IFLASH_INI_IMG:            data
```

I was interested in the `_IFLASH_BIOSIMG` file, because it contained the content
of the BIOS flash.

# Extracting the content of the Intel flash

I extracted the content of the flash ROM with UEFIExtract in the `new_engine`
branch of [UEFITool](https://github.com/LongSoft/UEFITool/tree/new_engine).
This resulted in a folder containing over 300 PE images. One of them sounded interesting: `SecureBackDoorPeim`,
but I think this blog post is long enough, so it may be the topic of another post.

# Next steps?

Maybe use [uefireverse](https://github.com/jethrogb/uefireverse) to reverse engineer
the EFI binaries? This seems really interesting, since there are a lot of dependencies
between EFI modules: each module can register protocols (a sort of syscalls), and
other EFI modules can call these. The hard part to reverse engineer is that these
protocols are identified by a 16-byte long EFI_GUID, and that it's kinda hard to
see what protocols a binary uses without either running it or manually reverse engineering it.
