+++
title = "00-introduction: Dialogue Concerning the Root of Trust"
url = 'pages/00-introduction'
date = "2025-09-15"
+++

$$
\def\ret{\operatorname{exec}}
$$

As Galileo spoke about the two chief world systems, I'll speak now about the system to which we entrust our security. On ARM based systems, the Root of Trust is the complex of immutable components that guarantees the security, in theory it's inherently secure. 

Before speaking of the implementation of the RoT, we should pay attention to how ARM (in this case AArch64) defines privileges. ARM systems have 4 Exception Levels, from EL3, the higher privilge, to EL0, the lowest. When an exception occours, CPU saves its state and moves to an higher privilege level (e.g. to handle requests to the hardware).

<p style="text-align: center;">
    <img src="/pages/root_of_trust/images/exception_levels.svg" alt="Exception Levels" style="width: 400px;" class="invertible">
</p>

# So how does the Root of Trust work?

Before speaking about the RoT, we need to take a look to the Clark-Wilson model applies to the verified boot and measured boot. In the Clark-Wilson model we have Data Items, Procedures and Rules.

## Clark-Wilson Security Model

Data Items can be Constrained Data Item (CDI) trusted entities or Unconstrained Data Item (UDI) untrusted entities. Procedures are Integrity Verification Procedure (IVP) and Transformation Procedures (TPs).

Procedures perform functions on data items upon two set of rules, I'll use $v$ for valid and $i$ for invalid and $V \subseteq \text{Valid}$ for certificable valid states in context. Entities, Procedures and States and have a type $\tau$ that is one in: $Item$, $Id$, $State$, $Fun$, $List(\tau)$, $Rel_n$ (with $n$ numbers of fields in the relation).

Certification Rules (CR) - integrity monitoring
1. (C1) [Basic: IVP Certification] IVPs ensure that CDIs are in valid state
$$
ivp : CDI \to \set{v, i} : State \\ \, \\
{ivp(cdi) \overset{\ret}{=} v : State \over [(V \subseteq \text{Valid} : List \subseteq List(State))] \vdash cdi \in V}
$$
1. (C2) [Basic: Validity] TPs must be certified to be valid. For each TP and each set of CDIs that it manipulates, the security officer specifies a relation of the form 
$$
\Validate : TP \to \set{v, i} : State,\, \Allow : TP \to CDI : Item \\
R_{\Allow} \subseteq \text{TP} \times \mathcal{P}^*(CDI) : List(i \subseteq Rel_2)\\ \, \\
{\Validate(tp) \overset{\ret}{=} v_1 : State\,;\Certify(tp,S) \overset{\ret}{=} v_2: State \over [(S \subseteq CDI : List \subseteq List(Item))] \vdash R_{\Allow}(tp) = S}
$$
1. (C3) [Separation of Duty Certification] The list of relations in (E2) must be certified to meet the Separation of Duty requirements 
$$
\Auth: User \times TP \times \mathcal{P}^*(CDI) \to \set{v, i} : State \\
R_{\Auth} \subseteq User \times TP \times \mathcal{P}^*(CDI) : List(i \subseteq Rel_3)\\ \, \\
{\Auth(u,tp,S) \overset{\ret}{=} v: State \over (u,tp,S) \in R_\text{Auth}}
$$
1. (C4) [Journal Certification] Al TPs must be certified to an append-only CDI (log) all info necessary to permit the nature of the operation to be reconstructed
$$
\Validate : TP \to \set{v, i} : State \\ \, \\
{\Validate(tp) \overset{\ret}{=} v \over \operatorname{List}(Rel^3)_\text{log} = \List(Rel^3)_\text{log} \mathbin{||} (\mathcal{P}^*(CDI), {\Braket{tp, v, \text{log}}})}
$$
1. (C5) [Transformation Certification] Any TP that takes a UDI as an input value must be certified to perform only valid transformations ($\succ_v$), or no transformations ($\succ_\empty$), for any possible value of UDI. The transformation should take the input from a UDI to a CDI.
$$
\Validate : TP \to \set{v,i} : State \\
\text{TP} : \mathcal{P}^*(UDI) \to \new \mathcal{P}^*(CDI) : List(i \subseteq List(Item)) \\ \, \\
{\Validate(tp) \overset{\ret}{=} v : State \over [\delta \in \mathcal{P}^*(CDI) : i \subseteq \List(Item)] \vdash tp \succ_v \delta \vee tp \succ_\empty \varnothing} 
$$

Enforcement Rule (ER) – integrity preserving

1. (E1) [Basic: Enforcement of Validity] The system must maintain the list of relation specified in (C2), and must ensure that only TPs certified to run on a CDI manipulate that CDI. 

$$
\text{(C2)} ~ R_{\Auth} \subseteq User \times TP \times \mathcal{P}^*(CDI) : List(i \subseteq Rel_3) \\
\Validate : TP \to \set{v, i} : State \\ \, \\
{\Validate(tp, \mathcal{P}^*(CDI), R_{\Auth}) \overset{\ret}{=} v : State \over  \exec tp(\mathcal{P}^*(CDI))}
$$
1. (E2) [Enforcement of Separation of Duty] The system must associate a user with each TP and set of CDIs in a list of relations of the form: $ (User, TP, \mathcal{P}^*(CDI)) $. It must ensure that only executions described in one of the relations are performed.

$$
\text{(C3)} ~ \Auth: User \times TP \times \mathcal{P}^*(CDI) \to \set{v, i} : State \\
\text{(C2)} ~ R_{\Auth} \subseteq User \times TP \times \mathcal{P}^*(CDI) : List(i \subseteq Rel_3)\\ \, \\
{\text{(C3)} ~ {\Auth(u,tp,S) \overset{\ret}{=} v: State \over (u,tp,S) \in R_\text{Auth}} \over \exec tp(\mathcal{P}^*(CDI)) \as u}
$$

2. (E3) [User Identity] The system must authenticate the identity of each user attempting to execute a $TP$.

$$
\text{(C3)} ~ \Auth: User \times TP \times \mathcal{P}^*(CDI) \to \set{v, i} : State \\
\Validate : User \to \set{v,i} : State \\ \, \\
{\Validate(u) = v : State \over \text{(C3)} ~ {\Auth(u,tp,S) \overset{\ret}{=} v: State \over (u,tp,S) \in R_\text{Auth}}}
$$

3. (E4) [Initiation] Only the agent permitted to certify entities may change the list of such entities associated with other entities, specifically the one associated with a $TP$. An agent that can certify an entity may not have any execute rights concerning that entity.

$$
\text{(E1)} ~ \Validate : TP \to \set{v, i} : State \\ \, \\
{\Agent u : User \text{ is Root} \over {\text{(E1)} ~ \Validate(tp, \mathcal{P}^*(CDI), R_{\Auth}) \overset{\ret}{=} i \as \Agent u\,:\,User \over tp \succ_v \delta \as \Agent u\,:\,User}}
$$

| **Property**   | **Description**                                                                                      | **Rule**                    |
|----------------|------------------------------------------------------------------------------------------------------|-----------------------------|
| Integrity      | Assurance that CDIs can only be modified in constrained ways to produce valid CDIs.              | (C1) (C2) (C5) (E1) (E4) |
| Access Control | The ability to control access to resources.                                                          | (C3) (E2) (E3)              |
| Auditing       | The ability to ascertain the changes made to CDIs and ensure that the system is in a valid state. | (C1) (C4)                   |
| Accountability | The ability to uniquely associate users with their actions.                                          | (E3)                        |


<p style="text-align: center;">
    <img src="/pages/root_of_trust/images/sec_model.svg" alt="Exception Levels" style="width: 600px;">
</p>

The Root of Trust ensures that all executed code (UDI) comes from a trusted source (CDI), from the hardware to the firmware. The Root of Trust usually works on top of a trusted Root Certificate Authority, to which other certificates refers, if a loaded image (UDI) hash is verified (by TPs) against the Root CA (first, always trusted CDI), it'll be executed.

To explain the Root of Trust more in depth, now we'll look to the Qualcomm Secure Boot Flow, Intel Boot Guard and the UEFI Secure Boot.


NaN <-> [Next Page](/pages/01-qcom_flow)

