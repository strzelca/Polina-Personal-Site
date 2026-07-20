+++
title = "UEFI Sucks"
url = 'pages/why_does_uefi_suck'
date = "2026-04-29"
+++

UEFI Sucks, but why?
If you can't answer to this question by yourself, you must rethink your life choices.

We know (I know, I don't trust you), that UEFI stands for Unified Extensible Firmware Interface, but there is the first problem

### "" _Unified_ "" Extensible Firmware Interface

They call UEFI an _"Unified"_ interface, because they think that the requirements for an unified standard is create a sect of big corps, call it a _"Forum"_, write a specification for the interface, and after that: do anything **BUT don't** follow the specification.

You have many firmware vendors, AMI Aptio, InsydeH20, HP UEFI, Lenovo UEFI, etc. everyone with different quirks. Those can break the runtime and boot services at any stage. They can break the boot services, breaking certain expected behaviors in events, protocols and image loader _(image = a PE32+ executable)_ that other executables like bootloaders expect. They can also break runtime services, so bootloaders' and drivers' developers will go crazy because EFI runtime return an unexpected memory map (so you must fix it in your bootloader, parsing the SMBIOS and creating a _CUSTOM LOGIC™️_ 🤮, or create a EFI driver in DXE, to split it in another executable and don't make your bootloader worse than it already is (~GRUB~)).

### UEFI Secure (for them) Boot

UEFI Secure Boot is a feature to enstablish a root of trust between the vendor's firmware and the operating system. It would be great if only it weren't just a tool for the big corps (and their gov friends, expecially two, one is an acronym of three letters, the other is a country in the middle east, that starts with I, and whose hobby is to threaten and bomb neighbouring countries because it was promised them 4,000 years ago) to increase their control over the platform.

If UEFI Secure Boot exist to protect the user from running untrusted executables, why are OpROMs signed with Microsoft's db key and in many platforms you can't boot your system without Microsoft keys? It's because it's secure for them, not for you, you think it's secure because you can enroll you keys, but you can't without Microsoft's too. There are workarounds, like extracting the OpROM from the external (usually a PCIe card) and enroll the hash, but you can't sign it with your keys and OEMs can't let you do this, or they'll lose the Microsoft WHQL certification for their drivers.

### Archictecture Indipendent? It can('t) be

The UEFI specification states that _«One of the design goals in the UEFI Specification is keeping the driver images as small as possible»_, so they implemented a thing, that has its reason to exist, compiling the source into an **EFI Byte Code** object, so instead of having many objects inside an OpROM for every architecture, we have a single **EBC** object that runs on the **EFI Byte Code Virtual Machine**.

I don't really like Bytecode solutions, but in this case it could be useful to address the space issue, but there is another problem, that isn't a technical issue.

Microsoft has a documentation on the dev center, the _"Microsoft UEFI Signing Requirements"_, these requirements must be complied, if you want Microsoft to sign your Loader/OpROM with their 3rd party key and ship these executable on real platform (because the majority of laptop sold have these keys). But now we'll take a look to some of these requirements:

- Code submitted for UEFI signing must not be subject to GPLv3 or any license that purports to give someone the right to demand authorization keys to be able to install modified forms of the code on a device. 

- SHIM submissions are exempt from obtaining an independent security audit only if your SHIM is handing off execution to an open source bootloader.

- Use of EFI Byte Code (EBC): Microsoft will not sign EFI submissions that are EBC-based submissions. 

So Ok, we know that Microsoft doesn't really like open-source (If you think that they do, ask for a psychiatrist), so they don't sign GPLv3 objects, because _"that purports to give someone the right to demand authorization keys to be able to install modified forms of the code on a device"_, so they don't like that you can modify open source software, but they trust SHIM submissions... but that wasn't what I wanted you to notice.

Microsoft DOES NOT sign EBC-based submissions, so the UEFI ~Firmware cartel~ Forum, of which Microsoft is a member, addressed the space problem of having many objects for every architecture, and Microsoft doesn't recognize it as "trusted" (???).

There is also a presentation by American Megatrends (Inc., a member of the UEFI Cartel) at the UEFI Plugfest 2011, that is called _«Best Practices for UEFI Driver & Option ROM Developers»_, that states that a combo of Legacy ROM + EBC OpROM is ok, but Legacy + UEFI x64 (+ UEFI IA32) is better. So is better to have an object for EVERY architecture? Are they coherent on something?

_**To be continue...**_

