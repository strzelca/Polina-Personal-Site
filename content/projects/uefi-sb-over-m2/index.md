+++
title = "UEFI Secure Boot on Apple Silicon with U-Boot/m1n1"
date = "2025-11-24"
+++

I hate UEFI and Secure Boot so much, that I deployed it on my **Macbook Air M2**, running **Arch Linux ARM** (Asahi Linux Kernel). The final result is an Apple Silicon machine, loading an U-Boot environment (_iBoot2 -> m1n1[1st->2nd] -> U-Boot_), compiled with **UEFI Secure Boot** support.

<!--more-->

This wasn't simple, U-Boot documentation is hard to understand (same argument spread across different pages), or lacks of necessary information.

Before talking about the steps, we must know how Linux is (or can be, because there are also FIT images) booted on Apple Silicon (and similar in other embedded platforms).

<p style="text-align: center;">
    <img src="/projects/uefi-sb-over-m2/images/untrusted-hlos.svg" alt="Apple Silicon Boot" style="width: 600px;">
</p>

Apple Silicon executes **Boot ROM** (always trusted, I already wrote about the [root of trust](/pages/00-introduction)), that validates and loads the **LLB** (_Low Level Bootloader_), the LLB validates firmware signatures, _LocalPolicy_ (together with **SEP**), verifies and loads **iBoot2**.

From **iBoot2** we can manipulate the boot flow, iBoot2 loads **m1n1**, **m1n1 1st stage** is verified by iBoot2, using the signed _LocalPolicy_ (that can't be changed without user password prompt), initializes hardware and loads a payload that can be _DTBs_, _initcpio_ images, or a proper _Linux kernel_ as _aarch64 boot image_. If we don't boot Linux from the 1st stage, m1n1 loads it's **2nd stage** from the _ESP_ (_EFI System Partition_), the 2nd stage can load blobs like the 1st stage, DTBs and **U-Boot**. 

The **m1n1 2nd stage** is preferable, because the 1st stage requires to reboot in a MacOS environment and execute the Asahi installer to update the binary, reboot in **1TR** (_RecoveryOS_) and reconfigure boot for the new m1n1 1st stage. With a 2nd stage we can update the 1st stage only if needed, and update easily the 2nd stage from the ESP directly from Linux.

Now that we loaded **U-Boot** from m1n1 2nd stage, we can boot Linux in many ways, loading the kernel, DTBs and initcpio manually, building a FIT image (not supported by the U-Boot build from Asahi), or a **PE EFI Stub** containing all the required blobs.

I'm using a custom U-Boot build with **UEFI Secure Boot** and FIT Image support, loading systemd-boot and an EFI stub with Linux+initcpio+dtbs. Everything from U-Boot is signed with my own keys _PK, KEK_ and _db_, and because it's funny, I put Microsoft UEFI keys in dbx (for more info about how UEFI Secure Boot works and key variables [I made an article about this](/pages/03-uefi_sb)).

<p style="text-align: center;">
    <img src="/projects/uefi-sb-over-m2/images/verified-boot-hlos.svg" alt="UEFI Secure Boot on Apple Silicon" style="width: 600px;">
</p>

To enable **UEFI Secure Boot** on the U-Boot from Asahi repo, we should change some settings with `make menuconfig`

```
CONFIG_FIT=y
CONFIG_FIT_SIGNATURE=y

CONFIG_EFI_SECURE_BOOT=y
CONFIG_EFI_VARIABLE_FILE_STORE=y
CONFIG_EFI_VARIABLES_PRESEED=y
CONFIG_EFI_VAR_SEED_FILE="ubootefi.var"

CONFIG_CMD_EFIDEBUG=y
CONFIG_CMD_EFICONFIG=y
CONFIG_CMD_EDITENV=y
CONFIG_CMD_GREPENV=y
CONFIG_CMD_SAVEENV=y
CONFIG_CMD_ERASEENV=y
CONFIG_CMD_ENV_EXISTS=y
CONFIG_CMD_ENV_FLAGS=y
CONFIG_CMD_NVEDIT_EFI=y
CONFIG_CMD_NVEDIT_INDIRECT=y
CONFIG_CMD_NVEDIT_INFO=y
CONFIG_CMD_NVEDIT_LOAD=y
CONFIG_CMD_NVEDIT_SELECT=y

```

In italian I would say "e qui casca l'asino" that means that now we'll encounter a millennium prize problem.

This is not (at all) well documented. There is an `enum` in `include/efi_variable.h`:
```C
enum efi_auth_var_type {
	EFI_AUTH_VAR_NONE = 0,
	EFI_AUTH_MODE,
	EFI_AUTH_VAR_PK,
	EFI_AUTH_VAR_KEK,
	EFI_AUTH_VAR_DB,
	EFI_AUTH_VAR_DBX,
	EFI_AUTH_VAR_DBT,
	EFI_AUTH_VAR_DBR,
};
```
so probably, the Secure Boot variables are handled differently, we know that when UEFI Secure Boot is in **SetupMode**, there isn't a PK, so we can deploy our own key, after that, PK is configured and SetupMode disabled, now all the other keys must be signed with the PK, or, if we deploy a KEK signed with the private PK, _db,dbx,dbt,dbr_ can be signed with the KEK too.

My ESP is `nvme 0:4`, your can be different. Commands are the same for the other variables.

```sh
fatload nvme 0:4 ${loadaddr} PK.auth
setenv -e -nv -bs -rt -at -i ${loadaddr}:$filesize PK
fatload nvme 0:4 ${loadaddr} KEK.auth
setenv -e -nv -bs -rt -at -i ${loadaddr}:$filesize KEK
fatload nvme 0:4 ${loadaddr} db.auth
setenv -e -nv -bs -rt -at -i ${loadaddr}:$filesize db
``` 

Yes. but, on this platform we can save EFI Variables only as a **file** on a **FAT32 **filesystem. If we try to set our keys on U-Boot, and we boot our EFI enabled kernel, launching `bootctl` it'll state `Secure Boot: enabled (user)`, but after a reboot the keys will disappear and Secure Boot disabled.

But why? If we check the `ESP/bootefi.var`, we can see that the EFI var with the keys are actually configured, so why are variables about the boot entries (`BootXXXX`) and order restored from the file, but Secure Boot related no?

This is not documented too, but searching in the source code, the routine that restores the variables from the file is `lib/efi_loader/efi_var_file.c:99 -> efi_status_t efi_var_restore(struct efi_var_file *buf, ...)`, there is a check:

```C
if (!safe &&
		    (efi_auth_var_get_type(var->name, &var->guid) !=
		     EFI_AUTH_VAR_NONE ||
		     !guidcmp(&var->guid, &shim_lock_guid) ||
		     !(var->attr & EFI_VARIABLE_NON_VOLATILE)))
			continue;

// ...

ret = efi_var_mem_ins(var->name, &var->guid, var->attr,
				      var->length, data, 0, NULL,
				      var->time);
//...

return EFI_SUCCESS;

```

if `efi_auth_var_type` from `efi_auth_var_get_type` is not `EFI_AUTH_VAR_NONE`, so a Secure Boot related variable, execution continue without setting them and return `EFI_SUCCESS`.

So, in order to set the EFI var regarding Secure Boot, we need to create a `ubootefi.var` from runtime variables, and recompile U-Boot using it as a **preseed**, as configured before.

This was the hard part, understanding WHY U-Boot was not restoring Secure Boot variables from file.

Now we have UEFI Secure Boot configured with hardcoded keys, **like in the most beautiful Qualcomm's dream**.

<div style="text-align: center;">
    <div style="display: inline-block;"><img src="/projects/uefi-sb-over-m2/images/uboot.jpg" alt="U-Boot" style="width: 450px;"></div>
    <div style="display: inline-block;"><img src="/projects/uefi-sb-over-m2/images/bootctl.png" alt="bootctl" style="width: 450px;"></div>
</div>


