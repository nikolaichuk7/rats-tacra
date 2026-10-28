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

 - ins: S. Nikolaichuk
   name: Serhii Nikolaichuk
   org: The Capital Index LLC
   email: nikolaichuk.s.f@gmail.com

normative:
  RFC9334: RATS
  RFC9180: HPKE

informative:
  RFC5280: PKIX
  RFC7030: EST
  RFC7519: JWT
  RFC8555: ACMEv2
  WIMSE: I-D.ietf-wimse-workload-creds
  CSR-ATTEST: I-D.ietf-lamps-csr-attestation
  INTERACTION-MODELS: I-D.ietf-rats-reference-interaction-models
  ATTESTATION-FRESHNESS: I-D.ietf-lamps-attestation-freshness
  DAA: I-D.ietf-rats-daa
  HPKE-BIS: I-D.ietf-hpke-hpke
  HPKE-PQ: I-D.ietf-hpke-pq
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

`present-*` kinds return a Freshness Handle. The Attester binds that Handle into Evidence ({{binding}}).
`absent-*` kinds return no Handle. The Attester binds a timestamp from a trusted clock or an epoch marker already held locally, or makes no freshness claim.

The Freshness Kind is one of:

* `absent-timestamp`: no Freshness Handle is returned; the Attester binds a timestamp from a trusted clock into Evidence ({{RFC9334}}, Section 10.1)
* `absent-none`: no Freshness Handle is returned; Evidence carries no freshness claim
* `absent-epoch`: no Freshness Handle is returned; the Attester binds an epoch marker already held locally into Evidence ({{RFC9334}}, Section 10.3)
* `present-nonce`: Initiate-Credential-Acquisition returns a single-use nonce as the Freshness Handle; the Attester binds that Handle into Evidence ({{RFC9334}}, Section 10.2)
* `present-epoch`: Initiate-Credential-Acquisition returns the current epoch marker as the Freshness Handle; the Attester binds that Handle into Evidence and retries Initiate-Credential-Acquisition if the epoch has moved ({{RFC9334}}, Section 10.3)

A Freshness Handle has a lifetime, set by its originator, after which the originator no longer accepts it. Initiate-Credential-Acquisition MAY return the lifetime with the Handle.
The lifetime SHOULD leave the Attester time to produce the Evidence that carries the Handle: a platform may rate-limit attestation requests, so that an Attester waits several seconds for one report.
A Freshness Handle originator SHOULD NOT set a lifetime below 60 seconds.

Under `absent-timestamp`, the RATS Relying Party accepts Evidence whose timestamp is no older than the maximum age its policy sets, and no later than its own clock by more than the skew it tolerates. Initiate-Credential-Acquisition MAY return that maximum age, so that the Attester knows it.
Section 10.1 of {{RFC9334}} notes that in many TEEs a clock is only available outside the TEE and cannot be trusted by it.
On such a platform the timestamp bounds the age of the Evidence by a clock set by whoever serves the Attester the current time: Evidence made with that clock set ahead is accepted once the RATS Relying Party's clock reaches the timestamp, less the skew, and for the maximum age after that.
A RATS Relying Party that needs a bound on the real age of Evidence uses a `present-*` Freshness Kind, or requires the Attester's time to be attested.


# Design Goals

This architecture is a result of work by the Confidential Computing Consortium's Trustworthy Workload Identity (TWI) SIG {{TWISIGCharter}} which has published a set of Definitions {{TWISIGDef}} and Requirements {{TWISIGReq}}.
The requirements published by the TWI SIG are deliberately high-level.
The design goals specified here fully align with the TWI SIG requirements, while focusing on a portable and extensible implementation.

1. MUST support mechanisms for enrolling (minting new) as well as retrieving (pre-existing) credentials; the workload knows what Credential Type it will need when it launches, but discovers whether it will have to retrieve an existing or enroll a new credential at runtime.
2. MUST support current and future credential formats:
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

At the high level, the TACRA architecture allows an Attester to obtain credentials from either a Credential Authority or a Secret Vault, both RATS Relying Parties, by interacting with these Relying Parties and the Verifier through an untrusted Credential Acquisition System (CAS).
The CAS can be any existing system, such as Enrollment over Secure Transport (EST), Secure Production Identity Framework for Everyone (SPIFFE/SPIRE), Automated Certificate Management Environment (ACMEv2), etc., extended with additional commands related to Remote Attestation.
The job of the Credential Acquisition System is to carry messages between the Attester and these back-end services using the Credential Acquisition System's own protocol.
This is illustrated below.

~~~~ ascii-art
                                      Enroll     +------------+
                               +-----------------> Credential |
                               |  +--------------+ Authority  |
                               |  |  Credential  +----+--^----+
             Initiate/         |  |                   |  |
             Enroll/           |  |                   |  |
+----------+ Retrieve  +-------+--v----+         +----v--+----+
|          +----------->  Credential   +--------->            |
| Attester |           |  Acquisition  |         |  Verifier  |
|          <-----------+    System     <---------+            |
+----------+ Initiate- +-------+--^----+         +----+--^----+
             Response/         |  |                   |  |
             Credential        |  |                   |  |
                               |  |  Credential  +----v--+----+
                               |  +--------------+   Secret   |
                               +----------------->   Vault    |
                                     Retrieve    +------------+
~~~~
{: #fig-hla title="High-Level TACRA Architecture"}

A more detailed architecture is illustrated below.

~~~~ ascii-art
{::include tacra_architecture.txt}
~~~~
{: #fig-tacra title="Detailed TACRA architecture"}

The CAS comprises a client and a server.
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

* Enrollment: the Credential Acquisition Interface generates a CSK and a CSR. The CSR carries CSKpub and, being self-signed, proves possession of CSKpri. The CSR MUST be bound to the Evidence as specified in {{binding}}, so that the Credential Authority can be sure the CSR was produced on the Attester's platform that the Evidence describes. Carrying Evidence inside the CSR ({{CSR-ATTEST}}) does not replace this binding.
* Retrieval: the Credential Acquisition Interface generates an asymmetric encryption key CEK. CEKpub MUST be bound to the Evidence as specified in {{binding}}. The resulting secrets are sealed to CEKpub, so that only the Attester in possession of CEKpri can read them, and the Secret Vault signs the container it releases ({{container}}).

During both Enrollment and Retrieval, the Attester MAY supply a Credential Hint.
During Enrollment, a Credential Authority MAY use the Credential Hint when assigning a Subject Alternative Name or other certificate properties.
During Retrieval, a Secret Vault MAY use the Credential Hint to locate the credential to return.
The RATS Relying Party MAY reject the request if it will not honor the hint.

A Credential Authority MAY include information from the Attestation Results in the credential, for parties that accept the credential and can act on that information.
In that case, the Credential Authority is responsible for ensuring that what it adds does not break RATS-unaware Relying Parties. For example, it keeps the addition small and, in a certificate, places it in a non-critical extension, which a certificate-using system that does not recognize may ignore (Section 4.2 of {{RFC5280}}).
Attestation Results can reveal much about the Attester's platform (Section 11 of {{RFC9334}}), and every party the credential is presented to can read what it carries, so the Credential Authority SHOULD limit what it adds to what those parties need.

## Binding Credential Keys to Evidence {#binding}

The Evidence of a credential-acquisition exchange MUST carry a digest of three values in its freshness input, the field of the Evidence whose content the Attester chooses:

1. the freshness element of the Freshness Kind ({{freshness-kind}}): the Freshness Handle for the `present-*` kinds, the locally held epoch marker for `absent-epoch`, the Attester's timestamp for `absent-timestamp`, and the empty string for `absent-none`;
2. the Target of the corresponding Initiate-Credential-Acquisition call;
3. the subject: for Enrollment, the CSR, which carries CSKpub and proves possession of CSKpri; for Retrieval, CEKpub.

The digest is computed over the octet string

~~~
binding_input = len32(freshness) || freshness
             || len32(target)    || target
             || len32(subject)   || subject
~~~

where:

* `len32(x)` is the length of `x` in octets, as a 32-bit big-endian unsigned integer;
* `freshness` is the freshness element; a timestamp is an 8-octet big-endian unsigned count of seconds since 1970-01-01T00:00:00Z;
* `target` is the UTF-8 encoding of the Target, used as an exact octet string;
* `subject` is, for Enrollment, the DER encoding of the CSR; for Retrieval, CEKpub in the encoding the Credential Acquisition Mechanism carries it in, which the protocol profile specifies.

The length prefixes make the encoding unambiguous: no two different sequences of values produce the same octets.
The digest is SHA-512 where the freshness input is 64 octets, as REPORT_DATA of AMD SEV-SNP and REPORTDATA of Intel TDX are. For a field of another size, a protocol profile specifies a hash whose output is the size of the field.

The Verifier MUST report in the Attestation Results the value in the freshness input of the Evidence it appraised. That is the value it read from the Evidence. Where the value to expect is submitted with the Evidence, as in challenge-response appraisal, it is that value, after the Verifier has checked that the Evidence carries it.
The Credential Authority (Enrollment) or Secret Vault (Retrieval) MUST recompute the digest from the freshness element and the Target of the request and the CSR or CEKpub it received. It MUST reject the request unless the result equals the value reported in the Attestation Results.
This is the same in the Passport and the Background Check model: either way, the Relying Party compares against a value the Verifier took from, or checked against, the appraised Evidence, never one it has from the CAS alone.

The Freshness Kind reaches the Attester through the CAS, so the Relying Party MUST take the Freshness Kind for a Target from its own policy, not from the request.
The freshness element MUST be checked under that kind before a credential is issued or released.
For the `absent-*` kinds, which return no Freshness Handle, the Relying Party makes that check.
For the `present-*` kinds, the Freshness Handle is either Verifier-originated or RATS Relying Party-originated.
The party that issues a Freshness Handle checks it. A party, the Verifier included, MUST NOT issue a Freshness Handle that it cannot check:

* Verifier-originated: the Verifier issues the Handle and checks it, and the Relying Party relies on the Verifier's check, never on the request alone. The Evidence carries the digest, not the Handle. The Verifier therefore either recomputes the digest from the three values or, in the Background Check model, checks at the Relying Party's request the same Handle the Relying Party used to recompute the digest.
* RATS Relying Party-originated: the Credential Authority (Enrollment) or Secret Vault (Retrieval) issues the Handle and checks it itself. A `present-nonce` Handle is one it issued and has not been used before. A `present-epoch` Handle is the current epoch marker.

Because the CAS is untrusted ({{untrusted-cas}}), each value closes a substitution:

* without the Target, the CAS could initiate, at the intended Relying Party, for a Target the Attester did not ask for;
* without the freshness element, it could replay;
* without the CSR or CEKpub, it could substitute a key.

The binding carries no Relying Party identifier. Which RATS Relying Party serves the Attester, and what a deployment does about it, is in {{untrusted-cas}}.

Producing the binding is the responsibility of the Platform Plug-in, because where the freshness input lives is platform-specific.
It takes one of three forms:

* Direct: the Attesting Environment writes the digest into a guest-chosen field of the hardware Evidence. Examples are REPORT_DATA of an AMD SEV-SNP attestation report, REPORTDATA of an Intel TDX quote, and the user data of an AWS Nitro attestation document.
* Nested: a lower layer owns that field (for example, a paravisor that sets it at boot), and the digest travels in a nested attestation whose report data the guest controls, such as a vTPM quote.
* Provider-scoped: the Evidence is signed by a key shared across a provider's fleet and identifies the provider's key domain rather than a device. The binding holds, but a Relying Party whose policy requires a per-device identity rejects such Evidence.

A Relying Party MUST NOT assume the Direct form; it learns the form from the Attestation Results or from its appraisal policy.

This binding ties the CSR or CEKpub to the Attester's platform, to the intended Relying Party and Target, and to freshness.
It does not by itself tie the exchange to the channel over which the credential is acquired.
Nor does it cover the Credential Type or the Credential Hint, which the CAS can change; a Relying Party MUST NOT issue or release a credential beyond what its policy allows for the attested context.

## Sealing the Released Credential {#container}

In Retrieval, the Secret Vault releases a credential bundle inside a container sealed to CEKpub ({{binding}}).
The CAS carries CEKpub and the container, and it is not trusted ({{untrusted-cas}}).
Sealing alone does not bound it: anyone who saw CEKpub can seal a container to it.
The container therefore MUST authenticate the Secret Vault as its origin, and the Attester MUST verify that origin before it opens anything.

The construction is as follows. A protocol profile specifies the HPKE suite, the signature algorithm, the labels and the encoding of the container and of its parts.

1. The Secret Vault seals the bundle to CEKpub with HPKE in base mode (Section 5.1.1 of {{RFC9180}}), with the `info` and the associated data below.
2. `info` is the length-prefixed concatenation, as in {{binding}}, of a label the profile sets, the Target and the freshness element of the request.
3. The associated data names the credential group, the Target, the freshness element and the Credential Hint of the request, in an encoding the profile sets. The AEAD authenticates it, and the Attester checks that it names what its Evidence bound.
4. The Secret Vault signs the length-prefixed concatenation of a label the profile sets, the HPKE suite identifier (Section 5.1 of {{RFC9180}}), the name of the signature algorithm, CEKpub as the request carried it, `info`, the associated data, the encapsulated key and the ciphertext. CEKpub among the signed octets ties the signature to one request's key.
5. The container carries the encapsulated key, the ciphertext, the associated data and the signature, and names its suite.

The Attester, through its Credential Acquisition Interface, holds the Secret Vault's signature key for each Target it retrieves from, as it holds the rest of its configuration (Design Goal 5).
It MUST refuse a container whose signature does not verify under that key, whose associated data does not name its Target, its freshness element and its Credential Hint, or whose suite is not the one the profile sets.
A container the CAS sealed itself, or the Secret Vault's container for another request, fails these checks.

A signature, and not an authenticated HPKE mode, because the successor of {{RFC9180}} removes the authenticated modes ({{HPKE-BIS}}), and the post-quantum KEMs have no authenticated encapsulation (Section 7.2 of {{HPKE-PQ}}).
A signature over the sealed container works with any KEM.

CEKpub travels in the encoding its KEM defines, and those octets are the subject of the binding ({{binding}}): the DER encoding of a SubjectPublicKeyInfo ({{RFC5280}}) for a Diffie-Hellman KEM, and the encapsulation key itself for the KEMs of {{HPKE-PQ}}, which have no SubjectPublicKeyInfo form.
The CEK is generated for one exchange inside the Attesting Environment and is not reused.

## Summary of RATS Roles

| TACRA Component | RATS Role | Remarks |
| :--- | :--- | :--- |
| Attester | Attester | Attesting Environment extended and complemented by CAI, CAAPI, Platform Plug-in |
| CAI | Part of Attester | Library or Sidecar assisting Attester in obtaining credentials |
| Platform Plug-in | Part of Attester | Invoked by CAI to perform platform-specific Remote Attestation and key generation operations |
| CAAPI Client | Part of Attester | Invoked by CAI to communicate with CAS Client |
| CAS Client | None: Conduit only | CAS Client extended by Remote Attestation Plug-in; SHOULD be outside the Attester's trust boundary and, wherever it runs, is trusted for nothing beyond carrying messages ({{untrusted-cas}}) |
| CAS Server | None: Conduit only | CAS Server extended by Remote Attestation Plug-in; forwards Freshness Handles, Evidence, and Attestation Results; does not appraise |
| Secret Vault | RATS Relying Party | Invoked in the Retrieval variant of this architecture; seals what it releases to CEKpub and signs the container ({{container}}), so that the CAS Server neither reads nor forges it |
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
* Credential Hint (optional) the Attester wishes to communicate when obtaining the credential

Returns:

* On success:
    * Credential acquisition mechanism: "enroll" or "retrieve"
    * Freshness Kind ({{freshness-kind}}): one of `absent-timestamp`, `absent-none`, `absent-epoch`, `present-nonce`, or `present-epoch`
    * Freshness Handle, when the Freshness Kind is `present-nonce` or `present-epoch`
    * Lifetime of the Freshness Handle, when one is returned (optional, {{freshness-kind}})
    * Maximum Evidence age, when the Freshness Kind is `absent-timestamp` (optional, {{freshness-kind}})
    * Other TBD pertinent information, such as supported ciphers, etc. (TODO: define)
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Unsupported Credential Type
    * Invalid Credential Hint
    * Server error: permission failure
    * Server error: server too busy; try again later
    * Server error: server unreachable
    * etc. (TBD)

## Enroll-Credential

Enrolls (mints) a new proof-of-possession credential.

Parameters:

* Target name matching that of the corresponding Initiate-Credential-Acquisition call
* Credential Type matching that of the corresponding Initiate-Credential-Acquisition call
* Freshness element of {{binding}}: the Freshness Handle returned by the corresponding Initiate-Credential-Acquisition call, the locally held epoch marker, or the Attester's timestamp; absent for `absent-none`
* Evidence carrying the binding of {{binding}}
* CSR bound to the Evidence as specified in {{binding}}; the CSR carries CSKpub and proves possession of CSKpri
* Credential Hint (optional) matching that of the corresponding Initiate-Credential-Acquisition call

Returns:

* On success: plaintext newly enrolled (minted) credential of the requested type
* On failure: enumerated reason for failure, such as:
    * Invalid Target
    * Unsupported Target
    * Unsupported Credential Type
    * Invalid Credential Hint
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
* Freshness element of {{binding}}: the Freshness Handle returned by the corresponding Initiate-Credential-Acquisition call, the locally held epoch marker, or the Attester's timestamp; absent for `absent-none`
* Evidence carrying the binding of {{binding}}
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
2. Platform Plug-in: Generate keys and Evidence for the Credential Acquisition Mode and Freshness kind returned in step 1, with the binding of {{binding}}, whose freshness element is:
    * Freshness kind of `present-nonce` or `present-epoch`: the Freshness Handle
    * Freshness kind of `absent-timestamp`: a timestamp from a trusted clock ({{RFC9334}}, Section 10.1)
    * Freshness kind of `absent-epoch`: the locally held epoch marker
    * Freshness kind of `absent-none`: empty (no freshness claim)
    * In all cases, the binding covers the CSR (enrollment) or CEKpub (retrieval)
3. CAAPI: Depending on which Credential Acquisition Mode is returned, either
    * Enroll-Credential(Target, Credential Type, Credential Hint, freshness element, CSR, Evidence)
    * Retrieve-Credential(Target, Credential Type, Credential Hint, freshness element, CEKpub, Evidence)
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

## Trust Model: the Credential Acquisition System as an Untrusted Conduit {#untrusted-cas}

This section exists because in this architecture the Attester never communicates with the Verifier or the RATS Relying Party directly. Instead, all messages between the Attester and these services flow through the Credential Acquisition System, and the Attester need not know what those entities are.
Transport security is therefore not end to end. There is a secure channel between the CAS Client and the CAS Server, and another between the CAS Server and the Verifier or the RATS Relying Party. The CAAPI between the Attester and the CAS Client is local to the Attester's system.
This architecture seeks to protect credentials acquired by the Attester from disclosure to the Credential Acquisition System.
For that reason, the Attester does not treat the Credential Acquisition System as anything other than a conduit between itself and the services needed for Remote Attestation and credential issuance. The CAS Client and the CAS Server hold no RATS role and no credential, and they are not assumed to be benevolent.
What a malicious CAS that delays, replays, rewrites or redirects messages can do is restricted as follows.

What each party holds, and what bounds it:

* The Attester, with its Credential Acquisition Interface, holds CSKpri or CEKpri inside the environment its Evidence attests.
  It can produce Evidence over any value it chooses as the freshness input, and ask for any Target. The Verifier appraises its platform and its launch, and the Credential Authority or Secret Vault decides by its policy, not by what the Attester asks.
* The CAS Client and the CAS Server see every message in transit: the Freshness Handle, the Evidence, the CSR or CEKpub, the Credential Type and the Credential Hint, and the credential or the sealed container. In the Passport model, the CAS Server also sees the Attestation Results.
  The CAS Client and CAS Server can delay, replay and reorder messages. They can obtain a Freshness Handle from any RATS Relying Party for any Target, carry a request to another RATS Relying Party, and change what the binding ({{binding}}) does not cover. In the Passport model, they can also choose the Verifier.
  What bounds them:
  * the binding, recomputed by the RATS Relying Party;
  * the freshness element;
  * the RATS Relying Party's policy;
  * the Verifiers the RATS Relying Party trusts;
  * the container sealed to CEKpub and signed by the Secret Vault ({{container}}).
* The Verifier holds Reference Values, the hardware vendors' roots and, when it originates them, Freshness Handles.
  It appraises Evidence and reports the freshness input it read. It decides nothing about credential issuance or retrieval.
* The Credential Authority and the Secret Vault hold the issuance policy, the signing key or the secrets, and the Verifiers they trust.
  Each recomputes the binding from the request and from what it holds ({{binding}}), and checks the freshness element under the Freshness Kind of its policy. The Secret Vault seals what it releases to CEKpub and signs the container ({{container}}).

The various attacks the CAS is capable of and their mitigations are listed below:

* Substitution of a value. If the CAS changes a request by modifying its Target, CSR, CEKpub, or freshness element, that request would be refused by the RATS Relying Party: the digest the RATS Relying Party recomputes no longer equals the one the Verifier read from the Evidence ({{binding}}).
* Substitution of the Attester. The CAS can replace the Evidence the Attester supplied with Evidence from a platform of its choosing. It obtains nothing for the Attester's keys:
  * a platform that is not genuine: its Evidence does not chain to the vendor's root;
  * a genuine platform running another launch: it fails the Target's Reference Values;
  * a genuine platform running the same launch: it is another instance of the attested workload, with its own keys, and what it obtains is a credential for those keys.
* Replay at the same RATS Relying Party. The RATS Relying Party keeps no record of a CSR or CEKpub it served. Only the freshness element can refuse a second offer of the same request:
  * `present-nonce`: one Evidence serves once, as the Handle is consumed on first use;
  * `present-epoch` and `absent-epoch`: the same request is served again until the epoch moves;
  * `absent-timestamp`: the same request is served again within the window the RATS Relying Party accepts;
  * `absent-none`: the same request is served again without limit.

  What the CAS obtains is a second credential, or a second release, for the Attester's key. A deployment that counts or audits credentials uses `present-nonce`.
* Redirection to another RATS Relying Party. The CAS can offer Evidence made for one Credential Authority or Secret Vault to another that trusts the same Verifier and serves a Target of the same name.
  Whatever that RATS Relying Party issues or releases is bound to the Attester's keys, so the CAS obtains nothing it can use. What is at stake is which RATS Relying Party serves the Attester, and who chooses it:
  * This document binds no Relying Party identifier ({{binding}}). Binding one would make the second RATS Relying Party refuse every Freshness Kind, including a Freshness Handle it originated itself.
  * As it is, the CAS Server chooses among the RATS Relying Parties that trust the same Verifier and serve that Target name. `present-nonce` keeps one Evidence from serving a second RATS Relying Party, but not the one the CAS Server started the exchange at.
  * A deployment that wants one RATS Relying Party, and no other, to serve a given Attester gives each RATS Relying Party its own Target names. The Target is bound, and a RATS Relying Party refuses a request for a Target it does not serve.
* Substitution of the response. In Retrieval, the CAS can return a container of its own making, sealed to CEKpub, which it saw in the request, or the Secret Vault's container for another request. The Attester opens only a container the Secret Vault signed for this request ({{container}}), so it obtains nothing from either.
* The choice of Verifier in the Passport model. The CAS Server chooses the Verifier it sends Evidence to, and relays the Attestation Results.
  A RATS Relying Party MUST accept Attestation Results only from a Verifier it trusts, and MUST check that the freshness input they report equals its own recomputation of the binding ({{binding}}). Any other Verifier chosen by the CAS is therefore of no use to it.
* What the binding does not cover. The Credential Type and the Credential Hint travel unbound, and a CAS can change them. Two things bound that. The RATS Relying Party's policy issues or releases nothing beyond what it allows for the attested context, whatever the request says. The Attester sees whether the credential it receives is the one it asked for ({{binding}}, {{credential-hint}}, {{credential-scope}}).
  The Freshness Kind is set by the RATS Relying Party's policy for the Target, not by the request ({{binding}}). The Freshness Handle, for the `present-*` kinds, is originated by the Verifier or by the RATS Relying Party ({{binding}}).
* Denial. The CAS can delay or drop any message, and obtain Freshness Handles it never uses. This is mitigated by the CAS Client authenticating to the CAS Server, and the CAS Server authenticating to the Verifier and to the RATS Relying Party. Whichever of them originates Freshness Handles can then bound what it originates per client ({{cas-client-auth}}).

What the CAS reads. Evidence is signed, not encrypted, and the CAS Client and the CAS Server read it in transit. In the Passport model, the CAS Server can also inspect the Attestation Results.
Evidence identifies the platform, its firmware level and the launched software, and may identify the device (Section 7.2 and Section 11 of {{RFC9334}}). The RATS architecture expects an entity that relays Evidence, without being the Attester or the intended Verifier, not to have access to it (Section 12.2 of {{RFC9334}}).
This architecture accepts that the CAS reads Evidence and Attestation Results, and a deployment accepts it with the CAS it operates.
A deployment that cannot accept it encrypts Evidence to the Verifier, which requires the Attester to hold the Verifier's key before it produces Evidence. This document does not specify that.
What the CAS cannot read is CSKpri, CEKpri and the secret the Secret Vault seals to CEKpub.

The CAS Client SHOULD be outside the Attester's trust boundary. Outside it, the CAS Client is not attested as part of the Attester, and this architecture relies on it for nothing beyond carrying messages. Inside it, as when the CAS Client runs in the same Confidential VM, it shares the Attester's environment and can inspect the secrets the Attester generates and the secrets the CAS Server returns. A deployment that runs it there accepts that.
A CAS Server that took a credential from the Secret Vault in plaintext and sealed it to CEKpub itself would expose the secret to the CAS Server, and could not produce the Secret Vault's signature ({{container}}). This architecture does not permit that variant: the Secret Vault seals and signs what it releases (Design Goal 10).

A compromised platform attestation key breaks every guarantee above. The Verifier's defences are the vendor's root, which the key must chain to, and the TCB level the Evidence reports, which the Verifier's policy can require to be current.
Under `absent-timestamp`, Evidence carries the current value of the Attester's clock, and its freshness is only as good as that clock ({{freshness-kind}}).
Where a credential is honoured once issued or released is set by the Credential Authority's or Secret Vault's policy, as for any credential, and not by the binding, which only keeps Evidence for one Target from obtaining a credential for another ({{credential-scope}}).

## Credential Hint

The Credential Hint is a request, not an authorization.
A Credential Authority or Secret Vault MAY use it when selecting issued credential properties, or locating a stored credential, and MAY ignore or reject it.
The hint MUST NOT cause issuance or release of a credential that Appraisal Policy for Attestation Results or local issuance policy would otherwise deny.

## Credential Scope {#credential-scope}

The binding ({{binding}}) ensures that Evidence produced for one Target does not obtain a credential for another.
It does not limit where the credential is honored once issued or released: that is the credential's scope, set, as for any credential, by the policy of the Credential Authority or Secret Vault and by the RATS-unaware Relying Parties that accept the credential.
A certificate, for example, is bound to the identities in its subject alternative names (Section 4.2.1.6 of {{RFC5280}}), and a JSON Web Token can name its intended recipients in its audience claim, which a recipient not named there rejects (Section 4.1.3 of {{RFC7519}}).
The scope comes from that policy and not from the request: the Credential Type and the Credential Hint, which the binding does not cover, cannot widen it ({{binding}}, {{credential-hint}}).
A deployment that needs a credential usable only at the requested Target issues or releases it with that scope.

## CAS Client Authentication to CAS Server {#cas-client-auth}

In this architecture, Remote Attestation is used to authenticate the Attester to the services (Secret Vault or Credential Authority) involved in Credential Issuance.
In any given implementation, the CAS Client MAY still authenticate to the CAS Server.

Whether CAS Client authentication must be bound to Attester authentication is left to protocol profiles.
TODO: Define what that binding looks like.


# IANA Considerations {#iana}

This document has no IANA actions.


--- back


# Acknowledgments
{:numbered="false"}

The authors thank the Confidential Computing Consortium's Trustworthy Workload Identity (TWI) Special Interest Group {{TWISIGCharter}} for the definitions {{TWISIGDef}} and requirements {{TWISIGReq}} that motivated this work.
