+++
title = "03-uefi_sb: UEFI Secure Boot's Root of Trust"
url = 'pages/03-uefi_sb'
date = "2025-09-21"
+++

Now that we took a look to the Qualcomm Boot Flow, we can look to the **UEFI Secure Boot**. UEFI Secure Boot runs on many architectures, x86, AArch32, AArch64, RISC-V.

Actually, a complete implementation of UEFI Secure Boot on ARM is the one used by Windows on ARM on Snapdraon Elite X. This UEFI implementations works on top of the Qualcomm Root of Trust, the UEFI variables aren't stored in the NVRAM region, but in a blob called uefisec.mbn using the keystore feature of the QTEE, so knowledge of the Qualcomm RoT is necessary. On x86 systems compliant with UEFI specification these keys are stored in the NVRAM region on the flash and UEFI is verified by the hardware owner.

# How does UEFI Secure Boot works?

| **Property**   | **Description**                                                                                   | **Rule**                 | **Implementation**                                                                         |
|----------------|---------------------------------------------------------------------------------------------------|--------------------------|--------------------------------------------------------------------------------------------|
| Integrity      | Assurance that CDIs can only be modified in constrained  ways to produce valid CDIs.              | (C1) (C2) (C5) (E1) (E4) | Yes, firmware need to verify  next component                                               |
| Access Control | The ability to control access to resources.                                                       | (C3) (E2) (E3)           | No, there is no user concept (but can work on top of ARM secure environment)                                                              |
| Auditing       | The ability to ascertain the changes made to CDIs and ensure that the system is in a valid state. | (C1) (C4)                | Yes, if TCG trusted boot platform  is enabled. TCG event log may  record such information. |
| Accountability | The ability to uniquely associate users with their actions.                                       | (E3)                     | No, there is no user concept.                                                              |

Every applied rule has a **UEFI Secure Boot** component:

- (C1) **Signature check**. PCR validation can also be done.
- (C2) Not applied. No User in UEFI.
- (C3) Not applied. No User in UEFI.
- (C4) **TPM Event Log**.
- (C5) **Input Verification**. Secure Boot Chain
- (E1) Procedures **performed** by **Verified Firmware**
- (E2) Not applied. No User in UEFI.
- (E3) **System Management Mode**.
- (E4) **OEM** is hardware owner.

**UEFI Secure Boot** expects four elements, two keys and two databases:

* Platform Key *(PK)*
* Key Exchange Key *(KEK)*
- UEFI Secure Boot Database *(DB)*
- Forbidden Signature Database *(DBX)*

Each certificate, according to the UEFI specification should be one of the expected type: **X.509** (commonly used for certificates), **RSA2048** and **SHA256**/**384**/**512**. The X.509 certificate includes public key, digital signature, identity with the issuing CA. There is an associated private key too, and the public and the private key together form a keypair. The X.509 certificate, as a DER encoded binary that includes the public key is stored in the Secure Boot storage and the private key is used to sign the UEFI executable. UEFI executables are custom **PE**/**COFF** (**PE32+**) images with custom Subsystems and Machine type.
Let's take a look at each UEFI key and database.

## Platform Key

The **Platform Key** is the root of trust, enstablishes a trust relationship between platform owner and firmware, the hatdware owner enrolls the public key into the firmware. The recommended PK format in UEFI specification is **RSA2048**, but still the **X.509** is commonly used.

## Key Exchange Key

**Key Exchange Keys** enstablish a trust relationship between the Executable and Firmware. KEKs are enrolled into the firmware and are verified against the PKpub.

## UEFI Secure Boot Database Key

The **UEFI Secure Boot Database Keys** are the public keys that are directly verified against the **Portable Executable**, if bootloaders, kernels, OpROMs, or other executables pass the **signature check**, they can be **loaded** and **executed**.

The DB keys are often updates and are verified against the KEKpub.


## Forbidden Signature Database

The **Forbidden Signature Database keys** are public **keys** or **hashes** known for being **malware**, and instead of revoke the whole key, if the executable hash is in DBX, it won't execute. DBX takes precedence over DB.


<p style="text-align: center;">
    <img src="/pages/root_of_trust/images/uefi_sb.svg" alt="UEFI Secure Boot" style="width: 800px;">
</p>


[Previous Page](/pages/02-intel_flow) < - End <!-- > [Next Page](/pages/04-) -->
