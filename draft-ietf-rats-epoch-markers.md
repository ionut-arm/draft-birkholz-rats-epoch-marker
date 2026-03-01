---
v: 3

title: Epoch Markers
abbrev: Epoch Markers
docname: draft-ietf-rats-epoch-markers-latest
stand_alone: true
area: "Security"
wg: RATS Working Group
kw: Internet-Draft

venue:
  mail: "rats@ietf.org"
  github: "ietf-rats-wg/draft-birkholz-rats-epoch-marker"

cat: std
consensus: true
submissiontype: IETF

author:
- name: Henk Birkholz
  org: Fraunhofer SIT
  abbrev: Fraunhofer SIT
  email: henk.birkholz@ietf.contact
  street: Rheinstrasse 75
  code: '64295'
  city: Darmstadt
  country: Germany
- name: Thomas Fossati
  organization: Linaro
  email: Thomas.Fossati@linaro.org
  country: Switzerland
- name: Wei Pan
  org: Huawei Technologies
  email: william.panwei@huawei.com
- name: Ionuț Mihalcea
  org: Arm
  email: ionut.mihalcea@arm.com
  country: United Kingdom
- name: Carsten Bormann
  org: Universität Bremen TZI
  street: Bibliothekstr. 1
  city: Bremen
  code: D-28359
  country: Germany
  phone: +49-421-218-63921
  email: cabo@tzi.org

normative:
  RFC3161: TSA
  RFC5652: CMS
  RFC8392: CWT
  RFC8610: CDDL
  RFC9090: CBOR-OID
  RFC9054: COSE-HASH-ALGS
  STD94: CBOR
#    =: RFC8949
  STD96: COSE
#    =: RFC9052
  RFC9581: CBOR-ETIME
  I-D.ietf-cose-cbor-encoded-cert: C509
  I-D.ietf-cbor-edn-literals: EDN
  X.690:
    title: >
      Information technology — ASN.1 encoding rules:
      Specification of Basic Encoding Rules (BER), Canonical Encoding
      Rules (CER) and Distinguished Encoding Rules (DER)
    author:
      org: International Telecommunications Union
    date: 2015-08
    seriesinfo:
      ITU-T: Recommendation X.690
    target: https://www.itu.int/rec/T-REC-X.690

informative:
  RFC9334: rats-arch
  I-D.ietf-rats-reference-interaction-models: rats-models
  I-D.ietf-scitt-architecture: scitt-receipts
  I-D.ietf-rats-eat: rats-eat
  I-D.ietf-lamps-csr-attestation: csr-attestation
  TCG-CoEvidence:
    author:
      org: Trusted Computing Group
    title: "TCG DICE Concise Evidence Binding for SPDM"
    target: https://trustedcomputinggroup.org/wp-content/uploads/TCG-DICE-Concise-Evidence-Binding-for-SPDM-Version-1.0-Revision-53_1August2023.pdf
    date: 2023-06
    rc: Version 1.00
  I-D.ietf-rats-ar4si: rats-ar4si
  IANA.cwt:
  IANA.cbor-tags:

entity:
  SELF: "RFCthis"

--- abstract

This document defines Epoch Markers as a means to establish a notion of freshness among actors in a distributed system.
Epoch Markers are similar to "time ticks" and are produced and distributed by a dedicated system known as the Epoch Bell.
Systems receiving Epoch Markers do not need to track freshness using their own understanding of time (e.g., via a local real-time clock).
Instead, the reception of a specific Epoch Marker establishes a new epoch that is shared among all recipients.
This document defines Epoch Marker types, including CBOR time tags, RFC 3161 TimeStampToken, and nonce-like structures.
It also defines a CWT Claim to embed Epoch Markers in RFC 8392 CBOR Web Tokens, which serve as vehicles for signed protocol messages.

--- middle

# Introduction

Systems that need to interact securely often require a shared understanding of the freshness of conveyed information.
This is certainly the case in the domain of remote attestation procedures.
In general, securely establishing a shared notion of freshness of the exchanged information among entities in a distributed system is not a simple task.

The entire {{Appendix A of -rats-arch}} deals solely with the topic of freshness, which is in itself an indication of how relevant, and complex, it is to establish a trusted and shared understanding of freshness in a RATS system.

This document defines Epoch Markers as a way to establish a notion of freshness among actors in distributed systems.
Epoch Markers are similar to "time ticks" and are produced and distributed by a dedicated system, the Epoch Bell.
Actors in a system that receive Epoch Markers do not have to track freshness using their own understanding of time (e.g., via a local real-time clock).
Instead, the reception of a certain Epoch Marker establishes a new epoch that is shared between all recipients.
In essence, the emissions and corresponding receptions of Epoch Markers are like the ticks of a clock, with these ticks being conveyed over the Internet.

In general (barring highly symmetrical topologies), epoch ticking incurs differential latency due to the non-uniform distribution of receivers with respect to the Epoch Bell.
This introduces skew that needs to be taken into consideration when Epoch Markers are used.

While all Epoch Markers share the same core property of behaving like clock ticks in a shared domain, various "Epoch ID" values are defined as Epoch Marker types in this document to accommodate different use cases and diverse kinds of Epoch Bells.

While most Epoch Markers types are encoded in CBOR {{-CBOR}}, and many of the Epoch ID types are themselves encoded in CBOR, a prominent format in this space is the TimeStampToken (TST) defined by {{-TSA}}, a DER-encoded TSTInfo value wrapped in a CMS envelope {{-CMS}}.
TSTs are produced by Time-Stamp Authorities (TSA) and exchanged via the Time-Stamp Protocol (TSP).
At the time of writing, TSAs are the most common providers of secure time-stamping services.
Therefore, reusing the core TSTInfo structure as an Epoch ID type for Epoch Markers is instrumental for enabling smooth migration paths and promote interoperability.
There are, however, several other ways to represent a signed timestamp or the start of a new freshness epoch, respectively, and therefore other Epoch Marker types.

To inform the design, this document discusses a number of interaction models in which Epoch Markers are expected to be exchanged.
The default top-level structure of Epoch Markers described in this document is CBOR Web Tokens (CWT) {{-CWT}}.
The present document specifies an extensible set of Epoch Marker types, along with the `em` CWT claim to include them in CWTs.
CWTs are signed using COSE {{-COSE}} and benefit from wide tool support.
However, CWTs are not the only containers in which Epoch Markers can be embedded.
Epoch Markers can be included in any type of message that allows for the embedding of opaque bytes or CBOR data items.
Examples include the Collection CMW in {{-csr-attestation}}, Evidence formats such as {{TCG-CoEvidence}} or {{-rats-eat}}, Attestation Results formats such as {{-rats-ar4si}}, or the CWT Claims Header Parameter of {{-scitt-receipts}}.

Epoch markers can be used in the following ways:

- as embeddings in other data formats
- as information elements in protocols
- in systems that integrate the aforementioned protocols or data formats
- in the deployment of such systems

All of these can be considered "users" of Epoch Markers and will be referred to as entities "using Epoch Markers” throughout the document.

## Terminology

This document makes use of the following terms from other documents:

* "conceptual messages" as defined in {{Section 8 of -rats-arch}}
* "freshness" and "epoch" as defined in {{Section 10 of -rats-arch}}
* "handle" as defined in {{Section 6 of -rats-models}}
* "Time-Stamp Authority" as defined by {{-TSA}}

{::boilerplate bcp14-tagged}

In this document, CDDL {{-CDDL}} is used to describe the data formats.  The examples in {{examples}} use the CBOR Extended Diagnostic Notation (EDN, {{-EDN}}).

# Epoch IDs

The RATS architecture introduces the concept of Epoch IDs that mark certain events during remote attestation procedures ranging from simple handshakes to rather complex interactions including elaborate freshness proofs.
The Epoch Markers defined in this document are a solution that includes the lessons learned from TSAs, the concept of Epoch IDs defined in the RATS architecture, and provides several means to identify a new freshness epoch. Some of these methods are introduced and discussed in {{Section 10.3 of -rats-arch}} (the RATS architecture).

# Interaction Models {#interaction-models}

The interaction models illustrated in this section are derived from the RATS Reference Interaction Models {{-rats-models}}.
In general, there are three major interaction models used in remote attestation:

* ad-hoc requests (e.g., via challenge-response requests addressed at Epoch Bells), corresponding to {{Section 7.1 of -rats-models}}
* unsolicited distribution (e.g., via uni-directional methods, such as broad- or multicasting from Epoch Bells), corresponding to {{Section 7.2 of -rats-models}}
* solicited distribution (e.g., via a subscription to Epoch Bells), corresponding to {{Section 7.3 of -rats-models}}

In all three interaction models, Epoch Markers can be used as content for the generic information element `handle` as introduced by {{-rats-models}}.
Handles are used to establish freshness in ad-hoc, unsolicited, and solicited distribution mechanisms of an Epoch Bell.
For example, an Epoch Marker can be used as a nonce in challenge-response remote attestation (e.g., for limiting the number of ad-hoc requests by a Verifier).
If embedded in a CWT, an Epoch Marker can be used as a `handle` by extracting the value of the `em` Claim or by using the complete CWT including an `em` Claim (e.g., functioning as a signed time-stamp token).
Using an Epoch Marker requires the challenger to acquire an Epoch Marker beforehand, which may introduce a sensible overhead compared to using a simple nonce.

# Epoch Marker Structure {#sec-epoch-markers}

This section specifies Epoch Marker types using {{CDDL}} and illustrates their usage and relationship with other IETF work (e.g, {{TSA}}) where applicable.
In general, Epoch Markers are intended to be conveyed securely, e.g., by being conveyed via a signed data structure, such as a CBOR Web Token (CWT), or by being conveyed via a secure channel.
The specification of such "outer" structures and protocols and the means how to secure them is out-of-scope of this document.
This documents defines the different types of Epoch Markers {{sec-iana-cbor-tags}}.
For example, an Epoch Marker can be used to construct a CBOR-based Trusted Time Stamp Token, similar in function to a {{TSA}} TimeStampToken, using CWT and the `em` Claim defined in this document (see {{fig-ex-2}} for an illustration).
The value(s) an Epoch Marker represents are intended to demonstrate freshness in messages and protocols of applications, but can also serve other purposes where trusted timestamps or time intervals are required.
As such, taken as an opaque value it is possible to use Epoch Markers as values for a nonce field in existing data structure or protocols that already support extra-data fields (such as a nonce field).
The similarity between nonce usage and Epoch Marker usage that can occur sometimes can also lead to applications where both are used in the same interaction in different places to serve distinct purposes.
A representative example for such an application scenario is a "nested" use of classical nonces and Epoch Markers: an Epoch Marker can be requested in order to be used as a nonce value for a specific data structure -- while a local generated nonce is used to retrieve that Epoch Marker via the "outer" ad-hoc interaction (e.g., nonce retrieval protocols that interact with an Epoch Bell to fetch an Epoch Marker to be used as a nonce).
As some Epoch Marker types represent certain timestamp variants, these Epoch Markers or the secure conveyance method they are used in do not necessarily come with some hard-coded message imprint (as it is always the case with {{TSA}} TimeStampTokens).
In essence, not all Epoch Marker types come with support for a binding between a message and an Epoch Marker (in contrast to the example in {{fig-ex-2}}.

The following Epoch Marker types are defined in this document:

~~~~ cddl
{::include cddl/epoch-marker.cddl}
~~~~
{: #fig-epoch-marker-cddl artwork-align="left"
   title="Epoch Marker types (tag numbers 2698x are suggested, not yet allocated)"}

~~~~ cddl
{::include cddl/epoch-marker-claim.cddl}
~~~~
{: #fig-epoch-marker-cwt artwork-align="left"
   title="Epoch Marker as a CWT Claim (CWT claim number 2000 is suggested, not yet allocated)"}

## Epoch Marker Types {#epoch-payloads}

This section specifies the Epoch Marker types listed in {{fig-epoch-marker-cddl}}

### CBOR Time Tags

CBOR Time Tags are CBOR time representations choosing from CBOR tag 0 (`tdate`, RFC3339 time as a string), tag 1 (`time`, Posix time as int or float), or tag 1001 (extended time data item).

~~~~ cddl
{::include cddl/cbor-time-tag.cddl}
~~~~

The CBOR Time Tag represents a freshly sourced timestamp represented as either `time` or `tdate`
({{Sections 3.4.2 and 3.4.1 of RFC8949@-CBOR}}, {{Appendix D of -CDDL}}), or `etime` ({{Section 3 of -CBOR-ETIME}}).


#### Creation

To generate the cbor-time value, the emitter MUST follow the requirements in {{sec-time-reqs}}.


### Classical RFC 3161 TST Info {#sec-rfc3161-classic}

DER-encoded {{X.690}} TSTInfo {{-TSA}}.  See {{classic-tstinfo}} for the layout.

~~~~ cddl
{::include cddl/classical-rfc3161-tst-info.cddl}
~~~~

The following describes the classical-rfc3161-TST-info type.

classical-rfc3161-TST-info:

: The DER-encoded TSTInfo generated by a {{-TSA}} Time Stamping Authority.

#### Creation

The Epoch Bell MUST use the following value as MessageImprint in its request to the TSA:

~~~ asn.1
SEQUENCE {
  SEQUENCE {
    OBJECT      2.16.840.1.101.3.4.2.1 (sha256)
    NULL
  }
  OCTET STRING
    BF4EE9143EF2329B1B778974AAD445064940B9CAE373C9E35A7B23361282698F
}
~~~

This is the sha-256 hash of the string "EPOCH_BELL".

The TimeStampToken obtained from the TSA MUST be stripped of the TSA signature.
Only the TSTInfo is to be kept the rest MUST be discarded.
The Epoch Bell COSE signature will replace the TSA signature.

### CBOR-encoded RFC3161 TST Info {#sec-rfc3161-fancy}

The TST-info-based-on-CBOR-time-tag is semantically equivalent to classical {{-TSA}} TSTInfo, rewritten using the CBOR type system.

~~~~ cddl
{::include cddl/tst-info.cddl}
~~~~

The following describes each member of the TST-info-based-on-CBOR-time-tag map.

{:vspace}
version:
: The integer value 1.  Cf. version, {{Section 2.4.2 of -TSA}}.

policy:
: A {{-CBOR-OID}} object identifier tag (111 or 112) representing the TSA's policy under which the tst-info was produced.
Cf. policy, {{Section 2.4.2 of -TSA}}.

messageImprint:
: A {{-COSE-HASH-ALGS}} COSE_Hash_Find array carrying the hash algorithm
identifier and the hash value of the time-stamped datum.
Cf. messageImprint, {{Section 2.4.2 of -TSA}}.

serialNumber:
: A unique integer value assigned by the TSA to each issued tst-info.
Cf. serialNumber, {{Section 2.4.2 of -TSA}}.

eTime:
: The time at which the tst-info has been created by the TSA.
Cf. genTime, {{Section 2.4.2 of -TSA}}.
Encoded as extended time {{-CBOR-ETIME}}, indicated by CBOR tag 1001, profiled as follows:

- The "base time" is encoded using key 1, indicating Posix time as int or float.
- The stated "accuracy" is encoded using key -8, which indicates the maximum
  allowed deviation from the value indicated by "base time". The duration map
  is profiled to disallow string keys. This is an optional field.
- The map MAY also contain one or more integer keys, which may encode
  supplementary information [^tf1].

[^tf1]: Allowing unsigned integer (i.e., critical) keys goes counter interoperability

{:vspace}
ordering:
: boolean indicating whether tst-info issued by the TSA can be ordered solely based on the "base time".
This is an optional field, whose default value is "false".
Cf. ordering, {{Section 2.4.2 of -TSA}}.

nonce:
: int value echoing the nonce supplied by the requestor.
Cf. nonce, {{Section 2.4.2 of -TSA}}.

tsa:
: a single-entry GeneralNames array {{Section 11.8 of -C509}} providing a hint in identifying the name of the TSA.
Cf. tsa, {{Section 2.4.2 of -TSA}}.

$$TSTInfoExtensions:
: A CDDL socket ({{Section 3.9 of -CDDL}}) to allow extensibility of the data format.
Note that any extensions appearing here MUST match an extension in the
corresponding request.
Cf. extensions, {{Section 2.4.2 of -TSA}}.

#### Creation

The Epoch Bell MUST use the following value as messageImprint in its request to the TSA:

~~~ cbor-diag
[
    / hashAlg   / -16, / sha-256 /
    / hashValue / h'BF4EE9143EF2329B1B778974AAD44506
                    4940B9CAE373C9E35A7B23361282698F'
]
~~~

This is the sha-256 hash of the string "EPOCH_BELL".

### Epoch Tick {#sec-epoch-tick}

An Epoch Tick is a single opaque blob sent to multiple consumers.

~~~~ cddl
{::include cddl/multi-nonce.cddl}
~~~~

The following describes the epoch-tick type.

epoch-tick:

: Either a string, a byte string, or an integer used by RATS roles within a trust domain as extra data (`handle`) included in conceptual messages {{-rats-arch}}. Similarly to the use of nonces, this allows the conceptual messages to be associated with a certain epoch. However, unlike nonces (which require uniqueness), Epoch Markers can be used in multiple interactions by every consumer involved.

#### Creation

The emitter MUST follow the requirements in {{sec-nonce-reqs}}.

### Epoch Tick List {#sec-epoch-tick-list}

A list of Epoch Ticks sent to multiple consumers.
The consumers use each Epoch Tick in the list sequentially.
Similarly to the use of nonces, this allows each interaction to be associated with a certain epoch. However, unlike nonces (which require uniqueness), Epoch Markers can be used in multiple interactions by every consumer involved.

~~~~ cddl
{::include cddl/multi-nonce-list.cddl}
~~~~

The following describes the Epoch Tick List type.

epoch-tick-list:

: A sequence of byte strings used by RATS roles in trust domain as extra data (`handle`) in the generation of conceptual messages as specified by the RATS architecture {{-rats-arch}} to associate them with a certain epoch.
Each Epoch Tick in the list is used in a consecutive generation of a conceptual message.
Asserting freshness of a conceptual message including an Epoch Tick from the epoch-tick-list requires some state on the receiver side to assess if that Epoch Tick is the appropriate next unused Epoch Tick from the epoch-tick-list.

#### Creation

The emitter MUST follow the requirements in {{sec-nonce-reqs}}.

#### Usage

Proving freshness requires receiver-side state to identify the “next unused” tick.
Systems using Epoch Tick lists SHOULD define how missing/out-of-order ticks are handled and how resynchronization occurs, as per {{sec-state-seq-mgmt}}.

### Strictly Monotonically Increasing Counter {#sec-strictly-monotonic}

A strictly monotonically increasing counter.

The counter context is defined by the Epoch Bell.

~~~~ cddl
{::include cddl/strictly-monotonic-counter.cddl}
~~~~

The following describes the strictly-monotonic-counter type.

strictly-monotonic-counter:

: An unsigned integer used by RATS roles in a trust domain as extra data in the production of conceptual messages as specified by the RATS architecture {{-rats-arch}} to associate them with a certain epoch. Each new strictly-monotonic-counter value must be higher than the last one.

#### Usage

Systems that use Epoch Markers SHOULD follow the guidance in {{sec-state-seq-mgmt}} in establishing an Epoch Marker acceptance policy for receivers.
To prove freshness, receivers SHOULD track the highest accepted counter and ensure it fulfills the acceptance policy.

## Time Requirements {#sec-time-reqs}

Time MUST be sourced from a trusted clock (see {{Section 10.1 of -rats-arch}}).

## Nonce Requirements {#sec-nonce-reqs}

A nonce value used in a protocol or message to retrieve an Epoch Marker MUST be freshly generated.
The generated value MUST have at least 64 bits of entropy (before encoding).
The generated value MUST be generated via a cryptographically secure random number generator.

A maximum nonce size of 512 bits is set to limit the memory requirements.
All receivers MUST be able to accommodate the maximum size.

## State and Sequencing Management {#sec-state-seq-mgmt}

Data structures containing Epoch Markers could be reordered in-flight even without malicious intent, leading to perceived sequencing issues.
Some Epoch Marker types thus require receiver state to detect replay/rollback or establish sequencing.
Systems that use Epoch Markers SHOULD define an explicit acceptance policy (e.g., bounded acceptance window) that accounts for reordering of markers.

There is a trade-off between keeping a single “global” epoch view versus per-Attester state at the Verifier: global-only policies can exacerbate latency-induced false replay rejections, while per-Attester tracking can be costly.
Systems that use Epoch Markers SHOULD document whether they use global epoch tracking or per-Attester state and, if necessary, the associated window.

# Signature Requirements {#sec-signature-reqs}

The signature over an Epoch Marker MUST be generated by the Epoch Bell.
Conversely, applying the first signature to an Epoch Marker always makes the issuer of a signed message containing an Epoch Marker an Epoch Bell.

# Security Considerations {#sec-seccons}

In distributed systems that rely on Epoch Markers for conveyance of freshness, the Epoch Bell plays a significant role in the assumed trust model.
Freshness decisions derived from Epoch Markers depend on the Epoch Bell’s key(s) and correct behavior.
If the Epoch Bell key is compromised, or the Bell is malicious/misconfigured, an attacker can emit valid-looking “fresh” Epoch Markers.
System deployments using Epoch Markers generally need to protect Bell signing keys (secure storage, rotation, revocation) and scope acceptance to the intended trust domain (e.g., expected issuer/trust anchor).
Similarly, the Bell's clock needs to be securely sourced and managed, to prevent attacks that skew the Bell's perception of time.

## Epoch Signalling Issues

{{Section 12.3 of -rats-arch}} provides a good introduction to attacks on conveyance of Epoch Markers.
A network adversary can replay validly signed Epoch Markers or delay distribution, and differential latency can lead to different parties having different views of the “current” epoch.

The epoch (acceptable window) duration is an operational security parameter: if too long, an Attester can create “good” Evidence in a good state and release it later while the epoch is still acceptable (notably for epoch-tick, epoch-tick-list, and strictly-monotonic-counter); if too short, distant Attesters may be rejected as stale due to latency.
Epoch Markers are also designed to be reusable by multiple consumers, unlike nonces.
Where per-session uniqueness is required, protocols typically need to bind Epoch Markers to an explicit nonce (e.g., see {{sec-epoch-markers}}).
Finally, system deployments using Epoch Markers are normally required to pin which Epoch Marker types are acceptable for a given trust domain to avoid downgrade.

## Operational Examples

The following illustrative cases highlight “reasonable best practice” choices for balancing freshness, replay protection, and scalability.

* *Nonce-bound Bell interaction*: When a Verifier uses a nonce challenge to trigger Evidence creation, the Attester can forward that nonce to the Epoch Bell to request an Epoch Marker with the nonce echoed inside.
For reuse and caching, the typical pattern is to keep the marker generic and embed the Verifier nonce alongside the marker in the Evidence: if the Bell signs a nonce-echoed marker, that marker is not reusable across sessions.
The nonce and marker are thus either bound by the Bell's signature, or by the attester's signature on the Evidence.
The Verifier checks that (1) the nonce matches its challenge, (2) the Epoch Marker signature chains to the expected Bell key, and (3) the marker satisfies the acceptance policy (e.g., highest-seen counter or time window).
This pairing gives per-session uniqueness while still allowing Epoch Marker reuse by multiple consumers.

* *Long-latency paths (e.g., LoRaWAN or DTN profiles)*: High propagation and queuing delays make tight epoch windows brittle.
In system deployments using Epoch Markers, epoch-tick-list can be pre-provisioned to Attesters so that each interaction consumes the next tick, with the Verifier keeping per-Attester sequencing state ({{sec-state-seq-mgmt}}).
Epoch duration should cover worst-case delivery plus clock skew of the Bell, and acceptance policies should allow an overlap (e.g., current and immediately previous epoch) to absorb in-flight drift while still rejecting replays beyond that window.

* *Large fleets sharing a Bell*: When many Attesters reuse the same Epoch Marker, per-Attester state at the Verifier may be impractical.
One approach is to accept a global highest-seen epoch (with a bounded replay window) while requiring each Evidence record to bind the Epoch Marker to the Attester identity and, when feasible, a Verifier-provided nonce.
This limits cross-attester replay of a single Epoch Marker while keeping the Bell stateless, which allows Epoch Markers to be cached and enables their broadcast distribution at scale.

# IANA Considerations {#sec-iana-cons}

[^rfced-replace]

[^rfced-replace]: RFC Editor: please replace {{&SELF}} with the RFC
    number of this RFC and remove this note.

## New CBOR Tags {#sec-iana-cbor-tags}

IANA is requested to allocate the following tags in the "CBOR Tags" registry
{{IANA.cbor-tags}}, preferably with the specific CBOR tag value requested:

| Tag | Data Item | Semantics | Reference |
| -- | -- | -- | -- |
| 26980 | bytes | DER-encoded RFC3161 TSTInfo | {{sec-rfc3161-classic}} of {{&SELF}} |
| 26981 | map | CBOR representation of RFC3161 TSTInfo semantics | {{sec-rfc3161-fancy}} of {{&SELF}} |
| 26982 | tstr / bstr / int | a nonce that is shared among many participants but that can only be used once by each participant | {{sec-epoch-tick}} of {{&SELF}} |
| 26983 | array | a list of multi-nonce | {{sec-epoch-tick-list}} of {{&SELF}} |
| 26984 | uint | strictly monotonically increasing counter | {{sec-strictly-monotonic}} of {{&SELF}} |
{: #tbl-cbor-tags align="left" title="New CBOR Tags"}

## New EM CWT Claim {#sec-iana-em-claim}

This specification adds the following value to the "CBOR Web Token Claims" registry {{IANA.cwt}}.

* Claim Name: em
* Claim Description: Epoch Marker
* Claim Key: 2000 (IANA: suggested assignment)
* Claim Value Type(s): CBOR array
* Change Controller: IETF
* Specification Document(s): {{sec-epoch-markers}} of {{&SELF}}

## New Media Type `application/em+cbor`

IANA is requested to add the `application/epoch-marker+cbor` media types to the "Media Types" registry {{!IANA.media-types}}, using the following template:

{:compact}
Type name:
: application

Subtype name:
: epoch-marker+cbor

Required parameters:
: no

Optional parameters:
: no

Encoding considerations:
: binary (CBOR)

Security considerations:
: {{sec-seccons}} of {{&SELF}}

Interoperability considerations:
: n/a

Published specification:
: {{&SELF}}

Applications that use this media type:
: RATS Attesters, Verifiers, Endorsers and Reference-Value providers, and Relying Parties that need to transfer Epoch Markers payloads over HTTP(S), CoAP(S), and other transports.

Fragment identifier considerations:
: The syntax and semantics of fragment identifiers are as specified for "application/cbor". (No fragment identification syntax is currently defined for "application/cbor".)

Person & email address to contact for further information:
: RATS WG mailing list (rats@ietf.org)

Intended usage:
: COMMON

Restrictions on usage:
: none

Author/Change controller:
: IETF

Provisional registration:
: no

## New CoAP Content-Format

IANA is requested to register the following Content-Format ID in the "CoAP Content-Formats" registry, within the "Constrained RESTful Environments (CoRE) Parameters" registry group {{!IANA.core-parameters}}:

| Content-Type | Content Coding | ID | Reference |
| application/epoch-marker+cbor | - | TBD1 | {{&SELF}} |
{: align="left" title="New CoAP Content Format"}

If possible, TBD1 should be assigned in the 256..9999 range.

--- back

# Examples {#examples}

The example in {{fig-ex-1}} shows an Epoch Marker with an `etime` as the Epoch Marker type.

~~~~ cbor-diag
{::include cddl/examples/1.diag}
~~~~
{: #fig-ex-1 artwork-align="center"
   title="CBOR Epoch Marker based on `etime` (EDN)"}

The encoded data item in CBOR pretty-printed form (hex with comments) is shown in {{fig-ex-1-pretty}}.

~~~~ cbor-pretty
{::include cddl/examples/1.pretty}
~~~~
{: #fig-ex-1-pretty artwork-align="center"
   title="CBOR Epoch Marker based on `etime` (pretty hex)"}

The example in {{fig-ex-2}} shows an Epoch Marker with an `etime` as the Epoch Marker type carried within a CWT.

~~~~ cbor-diag
{::include-fold cddl/examples/1-cwt.diag}
~~~~
{: #fig-ex-2 artwork-align="center"
   title="CBOR Epoch Marker based on `etime` carried within a CWT (EDN)"}

The encoded data item in CBOR pretty-printed form (hex with comments) is shown in {{fig-ex-2-pretty}}.

~~~~ cbor-pretty
{::include-fold cddl/examples/1-cwt.pretty}
~~~~
{: #fig-ex-2-pretty artwork-align="center"
   title="CBOR Epoch Marker based on `etime` carried within a CWT (pretty hex)"}

## RFC 3161 TSTInfo {#classic-tstinfo}

As a reference for the definition of TST-info-based-on-CBOR-time-tag the code block below depicts the original layout of the TSTInfo structure from {{-TSA}}.

~~~~ asn.1
TSTInfo ::= SEQUENCE  {
   version                      INTEGER  { v1(1) },
   policy                       TSAPolicyId,
   messageImprint               MessageImprint,
     -- MUST have the same value as the similar field in
     -- TimeStampReq
   serialNumber                 INTEGER,
    -- Time-Stamping users MUST be ready to accommodate integers
    -- up to 160 bits.
   genTime                      GeneralizedTime,
   accuracy                     Accuracy                 OPTIONAL,
   ordering                     BOOLEAN             DEFAULT FALSE,
   nonce                        INTEGER                  OPTIONAL,
     -- MUST be present if the similar field was present
     -- in TimeStampReq.  In that case it MUST have the same value.
   tsa                          [0] GeneralName          OPTIONAL,
   extensions                   [1] IMPLICIT Extensions   OPTIONAL  }
~~~~

# Acknowledgements
{:unnumbered}

TODO
