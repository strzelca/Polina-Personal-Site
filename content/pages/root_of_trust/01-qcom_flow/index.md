+++
title = "01-qcom_flow: Qualcomm Secure Boot Flow's Root of Trust"
url = 'pages/01-qcom_flow'
date = "2025-09-19"
+++

| **Property**   | **Description**                                                                                   | **Rule**                 | **Implementation**                                                                |
|----------------|---------------------------------------------------------------------------------------------------|--------------------------|-----------------------------------------------------------------------------------|
| Integrity      | Assurance that CDIs can only be modified in constrained  ways to produce valid CDIs.              | (C1) (C2) (C5) (E1) (E4) | Yes, loader need to verify next image before execute                                  |
| Access Control | The ability to control access to resources.                                                       | (C3) (E2) (E3)           | Yes, users but exception levels, e.g.         SCM call to QTEE manage access to TEE resources |
| Auditing       | The ability to ascertain the changes made to CDIs and ensure that the system is in a valid state. | (C1) (C4)                | No, Qualcomm doesn't provide  auditing, later stages like UEFI and AVB 2.0 yes.   |
| Accountability | The ability to uniquely associate users with their actions.                                       | (E3)                     | No, there is no user concept.                                                     |

Every applied rule has a **Qualcomm Secure Boot** component:

- (C1) **Signature check**.
- (C2) **Exception Levels**, but **calls** are **isolated** by **Secure Call Manager**
- (C3) **Exception Levels**, but there's **isolation** (e.g. QTEE isolates Secure World)
- (C4) No auditing. Provided by later stages.
- (C5) **Input Verification**. Secure Boot Chain
- (E1) **Loads** performed by **Verified Images**.
- (E2) **Exception Levels**
- (E3) **Secure Monitor**
- (E4) **OEM** is hardware owner.


The boot process begins exeuction to the **PBL** (Primary BootLoader), it's written on the **SoC ROM** and *can't be re-flashed*. The PBL does a check, if the QFPROM eFuse is blown and so a trusted cerfificate has been generated and configured, the signature of the next image will be checked and if verified, the PBL will move execution to it.

This second stage image is called XBL (eXtended BootLoader), on Qualcomm SoC inside XBL there are 4 loader regions, but we'll look only to three of them, XBL_SEC (region #0), XBL_Loader (region #1) and XBL_Core (region #3).

**XBL_SEC** does [1] security configuration in EL3, and loads the **XBL_Loader** [2] in EL1 mode, to which moves execution [3]. XBL_Loader calls XBL_SEC via the **Secure Call Manager (SCM)**.
The XBL_Loader does many things:

* **Initialize hardware** and resources *(PMIC, SHRM, Memory, Boot Device)*, does SCM call to XBL_SEC initialize *pIMEM* and *clocks*, loads and verifies *MISC image* and *AOP image* [4a]
- Loads and verifies **TrustZone** device configuration (*DEVCFG*) image from UFS [4b], the **Qualcomm Trusted Execution Environment (QTEE)** image with the **SEC data** image from QFPROM blown eFuse
* Loads and verifies the **Qualcomm Hypervisor Execution Environment (QHEE)** image from UFS [4c], usually *Qualcomm **Gunyah** Hypervisor*
- Loads and verifies the **XBL_CORE** image from UFS [4d]

At this point, XBL_Loader executes an SCM call to XBL_SEC, which moves execution to the **QTEE** [4e]. XBL_SEC is designed to ensure that the most privileged Execution Levels can be accessed via SMC calls only by trusted software.

The QTEE now sets up the secure environment and moves exeuction to the Qualcomm Hypervisor in EL2 (5). The QHEE executes **XBL_CORE** in *EL1* (6), which now loads the **desidered firmware interface** (7) like UEFI, U-Boot or Little Kernel based lk2nd and the High Level OS.


<p style="text-align: center;">
    <img src="/pages/root_of_trust/images/qcom_flow.svg" alt="Qcom Secure Boot" style="width: 600px;">
</p>

[Previous Page](/pages/00-introduction) < - > [Next Page](/pages/02-intel_flow)
