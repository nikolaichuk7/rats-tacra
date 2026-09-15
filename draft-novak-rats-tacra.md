---
title: Trustworthy Acquisition of Credentials via Remote Attestation
abbrev: TACRA
category: info

docname: draft-novak-rats-tacra-latest
submissiontype: IETF
number:
date:
# consensus: true
v: 3
area: "Security"
workgroup: "Remote ATtestation ProcedureS"
keyword:
 - trustworthy workload identity
 - remote attestation
 - credential enrollment
 - credential retrieval
venue:
  group: "Remote ATtestation ProcedureS"
  type: "Working Group"
  mail: "rats@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/rats/"
  github: "TheBankster/rats-tacra"
  latest: "https://TheBankster.github.io/rats-tacra/draft-novak-rats-tacra.html"

author:
 - ins: M. Novak
   name: Mark Novak
   org: J.P. Morgan Chase & Co.
   email: mark.f.novak@jpmchase.com

 - ins: M. Richardson
   name: Michael Richardson
   org: Sandelman Software Works
   email: mcr+ietf@sandelman.ca

 - ins: H. Birkholz
   name: Henk Birkholz
   org: Fraunhofer SIT
   email: Henk.Birkholz@ietf.contact

normative:
  RFC9334: RATS

informative:
  RFC5280: PKIX
  RFC7030: EST
  RFC7519: JWT
  RFC8555: ACMEv2
  RFC9266: Channel Binding for TLS 1.3
  WIMSE: I-D.ietf-wimse-workload-creds
  CSR-ATTEST: I-D.ietf-lamps-csr-attestation
  INTERACTION-MODELS: I-D.ietf-rats-reference-interaction-models
  ATTESTATION-FRESHNESS: I-D.ietf-lamps-attestation-freshness
  DAA: I-D.ietf-rats-daa
  ID-CRISIS:
    target: https://doi.org/10.1145/3779208.3785387
    title: "Identity Crisis in Confidential Computing: Formal Analysis of Attested TLS"
    author:
      - name: Muhammad Usama Sardar
      - name: Mariam Moustafa
      - name: Tuomas Aura
    date: 2026
  INTRA-HANDSHAKE-FAIL:
    target: https://www.cve.org/CVERecord?id=CVE-2026-33697
    title: "Intra-handshake.fail (CVE-2026-33697): High-severity CVE in Attested TLS"
    author:
      - name: Muhammad Usama Sardar
      - name: Viacheslav Dubeyko
      - name: Jean-Marie Jacquet
    date: 2026
  TWISIGCharter:
    target: https://github.com/confidential-computing/governance/blob/main/SIGs/TWI/TWI_Charter.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group - Charter
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  TWISIGDef:
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Definitions.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group - Definitions
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  TWISIGReq:
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Requirements.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group - Requirements
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  SPIFFE:
    target: https://github.com/spiffe/
    title: Secure Production Identity Framework for Everyone
    author:
      org: spiffe.io
  SPIRE:
    target: https://spiffe.io/docs/latest/spire-about/spire-concepts/
    title: SPIRE Concepts
    author:
      org: spiffe.io
  ENVOY:
    target: https://envoyproxy.io
    title: The Envoy Proxy
    author:
      org: Envoy
...

--- abstract

There is a large class of "RATS-Unaware" Relying Parties (RUPs) that Attesters nevertheless need to interoperate with.
Existing deployed services, which precede the introduction of Remote Attestation, are often difficult to change/update in significant ways due to, among other reasons, organizational friction, technological inertia, and regulatory policies.
There are significant advantages if workloads can be incrementally updated in the trustworthiness of the platform, without disrupting their clients and servers.

This document describes an architecture by which Remote Attestation is utilized for providing Attesters with Identity Documents (keys or credentials) to authenticate to RUPs.
This architecture is intended to work with common credential acquisition protocols and mechanisms such as EST, SPIFFE/SPIRE, ACMEv2, and many others.

Another important but separate goal is to encapsulate the Attester-side complexity of Remote Attestation and credential acquisition similar to how Envoy does it.
This allows Attesters to be implemented in a way that abstracts away the details of credential acquisition: both the protocols used and the Credential Acquisition Mechanisms employed, whether minting new (Enrollment), or requesting existing (Retrieval) credentials.
Likewise, the choice between RATS Passport and Background Check models is made opaque to the Attester, further simplifying its development.

--- middle


# Introduction {#intro}

Success of a technology is ultimately measured by its adoption.
The Remote ATestation procedureS (RATS) Architecture {{!RFC9334}} requires that RATS Relying Parties understand Attestation Results, execute Appraisal Policy for Attestation Results, and have trust in Verifiers.
A change in Evidence may lead to a change in either the Attestation Results or Appraisal Policy for Attestation Results.
However, it is common for authentication and authorization policies on Relying Parties to remain static for long periods of time.
This is achieved by limiting which entities get to receive the credentials used for authentication and authorization, rather than have the Relying Party make complex decisions based on the credential's changing content.

One key requirement for successful deployment of Remote Attestation-capable workloads is minimal blast radius.
When a workload is moved from a legacy to a remotely attestable Trusted Execution Environment, that workload can use Remote Attestation to obtain a stable and trustworthy Identity Document, while its clients and servers do not notice anything different.
For that, a mechanism is required by means of which a Secret Vault or a Credential Authority takes on the role of RATS Relying Party.
This provides an intermediation between Attestation Results and the RATS-Unaware Relying Parties whose authentication and authorization policies may precede the introduction of Remotely Attestable Workloads and remain static for long periods of time.
For the RATS-Unaware Relying Parties, these adoption barriers are eliminated, as these RUPs are capable of authenticating their clients utilizing Identity Document types they are already familiar with.

In summary, rather than using Remote Attestation directly against the RUP, the Attester uses it to obtain from the RATS Relying Party a key, bearer token or proof-of-possession credential that is compatible with the RUP.
This document details an Architecture by which legacy Identity Document issuance mechanisms are replaced or modified such that functionally identical Identity Documents are issued, but with the additional prerequisite of successful Remote Attestation of the workloads in question.

## Reasons for RATS-Unaware Relying Party Immutability

The most important and most common scenario addressed here is that of a workload that employs Remote Attestation but whose Relying Party has no capacity to process Attestation Results or execute Appraisal Policy for Attestation Results.
This RATS-Unaware Relying Party is typically unable to make the corresponding changes for a number of reasons:

* It may be a compiled object or container provided by a third party
* Or it may be implemented in a language not easily changed or upgraded with new capabilities
* Further, such a system may require extensive and significant review by an authority before changes to the core algorithm can be made
* Or, finally, the reluctance to change may come from organizational friction within an enterprise where the remotely attesting workload is organizationally separate from its Relying Party and different priorities of different parts of organization prevent them moving in lockstep

In all of these cases, it is assumed that the remotely attesting workload can make the necessary changes to perform remote attestation, and that interoperability with the RUP will be preserved so long as the pre-shared key, bearer token, or proof-of-possession credential obtained by the workload following Remote Attestation matches that expected by the RUP.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

* Proof-of-Possession Credential: a credential that requires a private asymmetric signing key to sign statements using this credential
    * PKIX certificate {{RFC5280}}
    * WIMSE Workload Identity Certificate (WIC) and Workload Identity Token (WIT) {{WIMSE}}
    * etc.
* Bearer Token Credential: a credential that does not require proof of possession of a secret to use:
    * Pre-shared symmetric key
    * API key
    * JSON Web Token JWT {{RFC7519}}
    * etc.
* Credential Type: a catch-all term for any concrete credential type from the two lists above
* Credential, a.k.a. Identity Document: an instance of a Credential Type
* Credential Acquisition Mechanism: one of any number of existing or future mechanisms for acquiring credentials, such as {{RFC7030}}, {{SPIFFE}}/{{SPIRE}}, {{RFC8555}}, {{DAA}}, etc.
* Credential Acquisition System (CAS): a client-server architecture comprising a CAS Client and a CAS Server which implements a Credential Acquisition Mechanism
* Credential Acquisition Mode: one of
    1. Credential Enrollment (minting new proof-of-possession credential), or
    2. Credential Retrieval (retrieving an existing, pre-provisioned credential of any Credential Type)
* Replica workloads: workloads that are functionally indistinguishable from the point of view of clients that authenticate to them or servers that they authenticate to; typical in "horizontal scale-out" scenarios where multiple identical workload instances are launched to handle the load in parallel
* Target: the RATS-unaware Relying Party for which the Attester seeks credentials
* Credential Hint: optional, implementation-defined information supplied by the Attester about the credential it expects. The Attester MAY obtain the hint from runtime configuration. A RATS Relying Party MAY use the hint, ignore it, or reject the request. This document does not specify the hint's syntax or meaning.
* CSK: Credential Signing Key for proof-of-possession credentials; CSKpri and CSKpub refer to the private and public portions, respectively
* CEK: Credential Encryption Key for retrieved secrets; CEKpri and CEKpub refer to the private and public portions, respectively
* Freshness Handle: a Handle as defined in {{INTERACTION-MODELS}} (for example an attestation nonce or an Epoch Marker), bound into Evidence to demonstrate freshness
* Freshness Kind: how freshness is established for one credential-acquisition exchange; see {{freshness-kind}}

Note: WITs are JWTs that require key confirmation.
That makes them proof-of-possession credentials, whereas JWTs are used without key confirmation and are thus considered bearer tokens.

## Freshness Kind {#freshness-kind}

This document classifies Evidence freshness for a credential-acquisition exchange by combining the methods in Section 10 of {{RFC9334}} with whether Initiate-Credential-Acquisition ({{Initiate-Credential-Acquisition}}) returns a Freshness Handle.

`present-*` kinds return a Freshness Handle; the Attester embeds that Handle in Evidence.
`absent-*` kinds return no Handle; the Attester uses a trusted clock, an epoch identifier already held locally, or no freshness claim.

The Freshness Kind is one of:

* `absent-timestamp`: no Freshness Handle is returned; the Attester stamps Evidence from a trusted clock ({{RFC9334}}, Section 10.1)
* `absent-none`: no Freshness Handle is returned; Evidence carries no freshness claim
* `absent-epoch`: no Freshness Handle is returned; the Attester embeds an epoch identifier already held locally ({{RFC9334}}, Section 10.3)
* `present-nonce`: Initiate-Credential-Acquisition returns a single-use nonce as the Freshness Handle; the Attester embeds that Handle in Evidence ({{RFC9334}}, Section 10.2)
* `present-epoch`: Initiate-Credential-Acquisition returns the current epoch marker as the Freshness Handle; the Attester embeds that Handle in Evidence and retries Initiate-Credential-Acquisition if the epoch has moved ({{RFC9334}}, Section 10.3)


# Design Goals

This architecture is a result of work by the Confidential Computing Consortium's Trustworthy Workload Identity (TWI) SIG {{TWISIGCharter}} which has published a set of Definitions {{TWISIGDef}} and Requirements {{TWISIGReq}}.
The requirements published by the TWI SIG are deliberately high-level.
The design goals specified here fully align with the TWI SIG requirements, while focusing on a portable and extensible implementation.

1. MUST support mechanisms for enrolling (minting new) as well as retrieving (pre-existing) credentials; the workload knows what Credential Type it will need when it launches, but discovers whether it will have to retrieve an existing or enroll a new credential at runtime.
2. MUST support most current and future credential formats:
   * X.509 Certificates
   * WIMSE Workload Identity Certificates (WICs)
   * WIMSE Workload Identity Tokens (WITs)
   * Bearer tokens (JWTs, API Keys)
   * TPM 2.0 DAA Group Certificates (for Replica workloads)
   * Pre-shared keys
   * Future formats through architectural extensibility
3. SHOULD limit the trust boundary expansion of the Attesting Environment to the minimum
4. MUST support, transparently to the Attester, most current and future Credential Acquisition Mechanisms:
   * EST (RFC 7030)
   * SPIFFE/SPIRE
   * ACMEv2 (RFC 8555)
   * TPM 2.0 DAA Join protocol (based on TPM 2.0 AK Cert)
   * Future mechanisms through architectural extensibility
5. MUST support Workloads utilizing different Credential Acquisition Mechanisms per-target
6. MUST support both Background Check and Passport RATS modes, indistinguishably from the PoV of the Attester
7. MUST support most current and future RATS Verifiers, Credential Authorities, and Secret Vaults, transparently to the Attester
8. MUST NOT assume that the Attester has network access
9. MUST be compatible with all existing and future Confidential Computing platforms meeting minimum requirements around secure cryptography and evidence generation
10. SHOULD restrict visibility of fetched secrets to the Attester, specifically excluding visibility by the CAS Client and CAS Server


# Architecture

## Attester's Role

1. At the Attester level, all work is performed by a new Acquire-Credential call through a dedicated ENVOY-like {{ENVOY}} Credential Acquisition Interface (CAI) which handles all the underlying complexity; this maximally simplifies Attester development.
2. To acquire credentials, CAI always uses a two-phase sequence:
    1. It requests and obtains a Freshness Kind ({{freshness-kind}}); freshness MAY come from the Verifier, or the Relying Party, or MAY even be empty {{INTERACTION-MODELS}} {{ATTESTATION-FRESHNESS}}
    2. It then generates and sends out Evidence, which is how the Attester demonstrates its security, possession of cryptographic key material, and capabilities
3. The Attester receives either an error or a newly acquired credential

## Required Modifications to Existing Credential Acquisition Mechanisms

It is not a goal, and, at any rate, it is not possible, to leave credential acquisition protocols and mechanisms (EST, SPIFFE/SPIRE, etc.) unmodified.
These mechanisms currently do not support Remote Attestation, for the following reasons:

1. None of the existing broadly deployed Credential Acquisition Mechanisms support this two-phase sequence, but all appear extensible to accommodate such changes without a lot of additional effort, and without risking backwards compatibility. CAS protocols MUST be able to carry Initiate-Credential-Acquisition; they need not be nonce-based.
2. The credentials that these existing mechanisms return are typically visible in plaintext to the control plane (the CAS Client and the CAS Server), whereas it is a common requirement to keep the knowledge of authentication secrets to the Attesters and a small number of trusted services, such as Secret Vaults and HSMs.

## Architecture Overview

~~~~ ascii-art
{::include tacra_architecture.txt}
~~~~
{: #fig-tacra title="TACRA architecture"}

This Architecture assumes the existence of a Credential Acquisition System (CAS), such as Enrollment over Secure Transport (EST), Secure Production Identity Framework for Everyone (SPIFFE/SPIRE), Automated Certificate Management Environment (ACMEv2), etc., that comprises a client and a server.
The CAS Client is presumed to be running on the Attester's system, but outside the Attesting Environment.
The CAS Server is a remote service invoked by the CAS Client over the CAS protocol.
Which CAS protocol is used MUST remain opaque to the Attester.

The Attester (the workload) runs inside an Attesting Environment, such as a Confidential Computing Trusted Execution Environment or TEE.
It obtains credentials by invoking the Credential Acquisition Interface (CAI), defined in this document.
CAI insulates the Attester from all the details of platform-specific and protocol-specific aspects and services involved in Remote Attestation and credential enrollment/retrieval.
The CAI MAY be a statically or dynamically linked library, an Envoy-style sidecar, or any other mechanism, but it MUST be part of the Attesting Environment.

CAI interacts with the outside world on behalf of the Attester via two channels:

1. With the underlying hardware platform to utilize its platform-specific functions, such as generating keys and obtaining evidence, via the platform-type-specific plugin, and
2. With the Credential Acquisition System Client, via the well-defined Credential Acquisition API (CAAPI), also outlined later in this document.

## Architecture Meeting Design Goals

In the text that follows, numbers in the format "(Goal #)" refer to the corresponding numbered items in the list of Design Goals in the opening section of this document.

CAAPI and the Platform Plug-in being the only two communication mechanisms needed to interact with the outside world, no network or storage stack are needed by the Attester (Goal 8).
The server side of CAAPI is part of the CAS Client.
There can be as many Credential Acquisition Client implementations as there are Credential Acquisition Mechanisms (Goal 4): EST Client, SPIRE Agent, etc.
There is no restriction against multiple Credential Acquisition Mechanisms collectively serving the same Attester, with different mechanisms utilized for different targets (Goal 5).
Existing CAS Clients are extended to support Remote Attestation via dedicated CAS Client Plug-ins.

The Credential Acquisition System controls which Credential Types and which Credential Acquisition Mechanisms (enrollment, retrieval) can be provisioned to the Attester for any Attester-supplied target, without the Attester's knowledge or involvement (Goal 1).
If a Credential Type specified by the Attester is unavailable due to Credential Acquisition System limitations, an error will result.
It is an administrative error to pair an Attester with a Credential Acquisition System that is unable to supply it with the Credential Type it requires.

The Credential Acquisition Server implements the server side of the corresponding Credential Acquisition Mechanism and interacts with the RATS Verifier, the Identity Provider (e.g., a Credential Authority for minting new certificates) and the Secret Vault for fetching existing keys or credentials, on the Attester's behalf (Goal 7).
The Credential Types supported by this Architecture are limited only by what the Credential Acquisition System can support (Goal 2).
Existing Credential Acquisition Servers are extended to support Remote Attestation via dedicated CAS Server Plug-ins.
The CAS Server's interactions with the Verifier are those of a conduit, not of a Relying Party.
When Initiate-Credential-Acquisition returns a Verifier-originated or RATS Relying Party-originated Freshness Handle, the CAS Server obtains that Handle and forwards it; it MUST NOT generate `present-nonce` or `present-epoch` values.
In the Passport model, the CAS Server forwards Evidence to the Verifier and forwards the resulting Attestation Results to the Credential Authority or Secret Vault, which remain the Relying Parties that execute Appraisal Policy for Attestation Results.
In the Background Check model, the Relying Party typically obtains Attestation Results from the Verifier directly; the CAS Server need not be on that path.
The CAS Server MUST NOT appraise, modify, or replace Attestation Results.
These Verifier exchanges are opaque to the Attester.

This arrangement shields the Attester developers from having to know the details of the platform on which the Attester runs (Goal 9).
It restricts the unavoidable expansion of the Attesting Environment to the smallest possible amount (Goal 3).
There is no difference, from the standpoint of the Attester, whether the RATS Passport or Background Check model is being used (Goal 6).

Under the covers and opaquely to the Attester, the Credential Acquisition Interface discovers and utilizes one of two Credential Acquisition Modes: Enrollment and Retrieval.
Enrollment corresponds to minting new proof-of-possession credentials, and Retrieval is used to fetch preshared keys, bearer tokens and shared proof-of-possession credentials (e.g., for Replica workloads).
In both cases, the associated secrets remain opaque to the CAS at all times (Goal 10) even if the credential, such as an X.509 certificate, is public and can be returned in plaintext.

* Enrollment: the Credential Acquisition Interface generates a CSK and CSR and includes alongside Evidence CSKpub and the CSR. There MUST exist a binding between the CSR/CSKpub and Evidence, so that the Credential Authority can be sure the CSR was produced on the Attester's platform that the Evidence describes. The binding MAY place the Evidence inside the CSR or the CSR inside the Evidence ({{CSR-ATTEST}}); this document RECOMMENDS the latter, achieved by including a collision-resistant digest of the CSR in the Evidence together with the Freshness Handle, as specified in {{binding}}. The CSR carries CSKpub and, being self-signed, proves possession of CSKpri. A Credential Authority MAY use a Credential Hint when assigning a Subject Alternative Name or other certificate properties.
* Retrieval: the Credential Acquisition Interface generates an asymmetric encryption key CEK and includes CEKpub in Evidence. The resulting secrets are encrypted to CEKpub, ensuring that only the Attester in possession of CEKpri can decrypt them. A Secret Vault MAY use a Credential Hint to locate the credential to return.

During both Enrollment and Retrieval, the Attester MAY supply a Credential Hint.
The RATS Relying Party MAY reject the request if it will not honor the hint.

## Binding Credential Keys to Evidence {#binding}

The Attester binds the key material of a credential-acquisition exchange to its Evidence by placing a collision-resistant digest into the freshness input of the Evidence, together with the Freshness Handle of {{freshness-kind}}.
For Enrollment the digest is taken over the CSR (which carries CSKpub and proves possession of CSKpri); for Retrieval it is taken over CEKpub.

Where the Evidence format provides a guest-chosen field for freshness, the binding is placed there directly.
For example, using the field named REPORT_DATA in AMD SEV-SNP, REPORTDATA in Intel TDX, or the extraData of a TPM quote:

~~~
freshness_input = H( Freshness Handle || CSR )      ; Enrollment
freshness_input = H( Freshness Handle || CEKpub )   ; Retrieval
~~~

The Credential Authority (Enrollment) or Secret Vault (Retrieval) recomputes freshness_input from the CSR or CEKpub it received and the Freshness Handle it issued, and rejects the request unless the value matches the one carried in the appraised Evidence.

Producing this binding is the responsibility of the Platform Plug-in, because the shape of the freshness input is platform-specific:

* Direct: the Attesting Environment writes the freshness input into a guest-chosen field of the hardware Evidence (e.g., AMD SEV-SNP, Intel TDX, AWS Nitro).
* Nested: where a lower layer owns the hardware freshness field (e.g., a paravisor that fixes it at boot), the freshness input is carried instead through a nested attestation such as a vTPM quote whose report data the guest controls.
* Provider-scoped: where the Evidence is signed by a key shared across a provider's fleet and the per-machine identity is masked, the binding remains valid but the Evidence identifies the provider's key domain rather than an individual machine; a Credential Authority whose policy requires a per-machine identity treats such Evidence accordingly.

This binding ties the CSR or CEKpub to the Attester's platform and to freshness. It does not by itself tie the exchange to the channel over which the credential is acquired; see {{channel-binding}}.

## Summary of RATS Roles

| TACRA Component | RATS Role | Remarks |
| :--- | :--- | :--- |
| Attester | Attester | Attesting Environment extended and complemented by CAI, CAAPI, Platform Plug-in |
| CAI | Part of Attester | Library or Sidecar assisting Attester in obtaining credentials |
| Platform Plug-in | Part of Attester | Invoked by CAI to perform platform-specific Remote Attestation and key generation operations |
| CAAPI Client | Part of Attester | Invoked by CAI to communicate with CAS Client |
| CAS Client | None: Conduit only | CAS Client extended by Remote Attestation Plug-in, SHOULD be outside the Attester's trust boundary |
| CAS Server | None: Conduit only | CAS Server extended by Remote Attestation Plug-in; forwards Freshness Handles, Evidence, and Attestation Results; does not appraise |
| Secret Vault | RATS Relying Party | Invoked in the Retrieval variant of this architecture; SHOULD encrypt retrieved results to CEKpub in order to keep it from leaking to the CAS Server |
| Credential Authority | RATS Relying Party | Invoked in the Enrollment variant of this architecture |
| Verifier | Verifier | No changes in RATS Verifier role or implementation |
| RATS-Unaware Relying Party | None | No changes in RUP role or implementation |
{: #tab-rats-roles title="RATS roles in TACRA"}


# Credential Acquisition API (CAAPI)

The Credential Acquisition API allows the Attester to communicate with the Credential Acquisition System.
These APIs are invoked by the Credential Acquisition Interface, covered in the next section.
CAAPI can be implemented using any mechanism suitable for interprocess communication, including but not limited to Protobuf/gRPC.
Here only the high-level description is provided.

## Initiate-Credential-Acquisition {#Initiate-Credential-Acquisition}

Initiates the credential acquisition process by obtaining Freshness and validating that the indicated Target name, Credential Type, and Credential Hint, if any, are supported by the CAS.

Parameters:

* Target name, e.g., the server URI to which the Attester wishes to authenticate
* Credential Type the Attester plans to use with this Target

Returns:

* On success:
    * Credential acquisition mechanism: "enroll" or "retrieve"
    * Freshness Kind ({{freshness-kind}}): one of `absent-timestamp`, `absent-none`, `absent-epoch`, `present-nonce`, or `present-epoch`
    * Freshness Handle, when the Freshness Kind is `present-nonce` or `present-epoch`
    * Other TBD pertinent information, such as supported ciphers, etc. (TODO: define)
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Unsupported Credential Type
    * Server error: permission failure
    * Server error: server too busy; try again later
    * Server error: server unreachable
    * etc. (TBD)

## Enroll-Credential

Enrolls (mints) a new proof-of-possession credential.

Parameters:

* Target name matching that of the corresponding Initiate-Credential-Acquisition call
* Credential Type matching that of the corresponding Initiate-Credential-Acquisition call
* Evidence, bound to the previously returned Freshness Handle, if any
* CSR bound to the Evidence as specified in {{binding}}; the CSR carries CSKpub and proves possession of CSKpri
* Credential Hint (optional)

Returns:

* On success: plaintext newly enrolled (minted) credential of the requested type
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Unsupported Credential Type
    * Rejected or unsupported Credential Hint
    * Server error: Remote Attestation failure
    * Server error: permission failure
    * Server error: server too busy; try again later
    * Server error: server unreachable
    * etc. (TBD)

## Retrieve-Credential

Retrieves (fetches pre-existing) credential.

Parameters:

* Target name matching that of the corresponding Initiate-Credential-Acquisition call
* Credential Type matching that of the corresponding Initiate-Credential-Acquisition call
* Evidence, bound to the previously returned Freshness Handle, if any
* CEKpub bound to the Evidence as specified in {{binding}}
* Credential Hint (optional)

Returns:

* On success: wrapped (encrypted to CEKpub) credential of the requested type
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Invalid Credential Type
    * Rejected or unsupported Credential Hint
    * Server error: failed remote attestation
    * Server error: permission failure
    * Server error: remote attestation failure
    * Server error: server too busy; try again later
    * Server error: server unreachable
    * etc. (TBD)

## CAAPI Invocation Sequence

The caller (normally the Credential Acquisition Interface) first decides which Target it wishes to authenticate to, using which Credential Type.
CAAPI offers no facilities for discovering these; they are decided out of band, for example from Attester runtime configuration.

The typical invocation flow is:

1. CAAPI: Initiate-Credential-Acquisition(Target, Credential Type)
    * Returns Freshness kind and the Credential Acquisition Mode for this Target and Credential Type
2. Platform Plug-in: Generate keys and Evidence for the Credential Acquisition Mode and Freshness kind returned in step 1:
    * Freshness kind of `present-nonce` or `present-epoch`: embed the Freshness Handle in Evidence
    * Freshness kind of `absent-timestamp`: stamp Evidence from a trusted clock ({{RFC9334}}, Section 10.1)
    * Freshness kind of `absent-epoch`: embed the locally held epoch marker
    * Freshness kind of `absent-none`: no freshness claim
    * In all cases, include in Evidence CSKpub plus CSR (enrollment) or CEKpub (retrieval)
3. CAAPI: Depending on which Credential Acquisition Mode is returned, either
    * Enroll-Credential(Target, Credential Type, Credential Hint, CSR, Evidence)
    * Retrieve-Credential(Target, Credential Type, Credential Hint, CEKpub, Evidence)
4. CAI examining Enroll-Credential or Retrieve-Credential results:
    * If Enroll-Credential or Retrieve-Credential fails because a `present-epoch` marker has moved (or a `present-nonce` is no longer valid), it retries from Initiate-Credential-Acquisition.
    * These retries are not visible to the Attester.

# Credential Acquisition Interface (CAI)

The Credential Acquisition Interface can be implemented using any mechanism suitable for local communication, including but not limited to statically linked calls and Protobuf/gRPC.
Here only the high-level description is provided.
The CAI consists of a single Acquire-Credential API, outlined below.
The Acquire-Credential implementation follows the recommended CAAPI invocation sequence.

## Acquire-Credential

Orchestrates an opaque-to-Attester process by which the Attester acquires a credential that it would need to authenticate to a given Target utilizing the given Credential Type.

Parameters:

* Target name, e.g., the server URI to which the Attester wishes to authenticate
* Credential Type the Attester plans to use with this Target
* Credential Hint (optional)

All of these parameters the Attester MAY learn from runtime configuration.

Returns:

* On success: newly acquired credential of the requested type
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Unsupported Credential Type
    * Rejected or unsupported Credential Hint
    * Server error: Remote Attestation failure
    * Server error: permission failure
    * Server error: server too busy; try again later
    * Server error: server unreachable
    * etc. (TBD)


# Security Considerations {#security}

## Undesirability and Inevitability of Bearer Tokens

This specification supports but discourages the use of bearer token credentials.
They are supported in the interest of maximizing compatibility.
While the specification takes care to deliver bearer token credentials to the Attester securely, subsequent usage, such as using them in authentication against the RUP, still risks leaking them.

## Credential Acquisition System Considered Untrustworthy

This architecture seeks to protect credentials acquired by the Attester from disclosure to the Credential Acquisition System.
For that reason, it does not treat the Credential Acquisition System as anything other than a conduit between the Attester and the services needed for Remote Attestation and credential issuance.
It leaves open the possibility that the CAS Server would retrieve credentials from Secret Vault in plaintext and itself encrypt them with CEKpub so that only the Attester can read them.
Likewise, it leaves open the possibility that there is no trust boundary between the Attester and CAS Client, and the CAS Client is therefore capable of inspecting the secrets the Attester generates and the secrets the CAS Server returns.
Both of these options are discouraged.

The CAS is expected to remain benevolent and not tamper with or leak the traffic between the Attester, the Verifier, and the RATS Relying Parties. However, the CAS is still untrusted, and the burden on protection from such attacks rests with these trusted endpoints.

## Binding Evidence to the Credential-Acquisition Channel {#channel-binding}

The binding of {{binding}} ties the Evidence to the Attester's platform, to the CSR or CEKpub, and to freshness, but not to the channel over which the credential is acquired.
Because the CAS is an untrusted conduit, an attacker that can act as the Attester's peer on that channel can relay genuine Evidence and obtain a credential, while every check in {{binding}} still passes.
This is possible whenever the attacker holds the Attester's channel key material, whether leaked, provisioned at runtime, or extracted on another machine: binding Evidence to a public key, with or without a nonce, does not correlate the Evidence with the channel ({{ID-CRISIS}}, {{INTRA-HANDSHAKE-FAIL}}).

To prevent this, when a credential-acquisition exchange runs over a secure channel, the Attester SHOULD additionally include a value derived from that channel's shared secret, such as the TLS exporter of {{RFC9266}}, in the freshness input of {{binding}}, and the Relying Party SHOULD require it.
The exchange then resists relay as long as one of the Attester's channel key material or the channel secret remains unknown to the attacker.

## Credential Hint

The Credential Hint is a request, not an authorization.
A Credential Authority or Secret Vault MAY use it when selecting issued credential properties, or locating a stored credential, and MAY ignore or reject it.
The hint MUST NOT cause issuance or release of a credential that Appraisal Policy for Attestation Results or local issuance policy would otherwise deny.

## CAS Client Authentication to CAS Server

In this architecture, Remote Attestation is used to authenticate the Attester to the services (Secret Vault or Credential Authority) involved in Credential Issuance.
In any given implementation, the CAS Client MAY still authenticate to the CAS Server.

Whether CAS Client authentication must be bound to Attester authentication is left to protocol profiles.
When such binding is required, a profile can reuse the channel-binding value of {{channel-binding}} so that CAS Client authentication and the Attester's Evidence refer to the same channel.


# IANA Considerations {#iana}

This document has no IANA actions.


--- back


# Acknowledgments
{:numbered="false"}

The authors thank the Confidential Computing Consortium's Trustworthy Workload Identity (TWI) Special Interest Group {{TWISIGCharter}} for the definitions {{TWISIGDef}} and requirements {{TWISIGReq}} that motivated this work.
