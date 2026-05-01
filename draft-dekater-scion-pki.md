---
title: "SCION Control Plane PKI"
abbrev: "SCION CP-PKI"
category: info
submissiontype: independent

docname: draft-dekater-scion-pki-latest
v: 3
# area: "IRTF"
# workgroup: "Path Aware Networking RG"
keyword: Internet-Draft
venue:
#  group: "Path Aware Networking RG"
#  type: "Research Group"
#  mail: "panrg@irtf.org"
#  arch: "https://www.ietf.org/mail-archive/web/panrg/"
  github: "scionassociation/scion-cppki_I-D"
  latest: "https://scionassociation.github.io/scion-cppki_I-D/draft-dekater-scion-pki.html"

author:
 -   ins: C. de Kater
     name: Corine de Kater
     org: SCION Association
     email: c_de_kater@gmx.ch

 -   ins: N. Rustignoli
     name: Nicola Rustignoli
     org: SCION Association
     email: nic@scion.org

 -   ins: S. Hitz
     name: Samuel Hitz
     org: Anapaya Systems
     email: hitz@anapaya.net

normative:
  I-D.dekater-scion-controlplane:
  I-D.dekater-scion-dataplane:
  RFC5280:
  RFC5480:
  RFC5652:
  RFC5758:
  RFC9217:
  X.509:
    title: "ITU-T X.509 (10/2016) | Information technology - Open Systems Interconnection - The Directory: Public-key and attribute certificate frameworks"
    date: 10/2016
    target: https://handle.itu.int/11.1002/1000/13031
  X.680:
    title: "ITU-T X.680 (02/2021) | Information technology - Abstract Syntax Notation One (ASN.1): Specification of basic notation"
    date: 02/2021
    target: https://handle.itu.int/11.1002/1000/14468
  X.690:
    title: "ITU-T X.690 (02/2021) | Information technology - ASN.1 encoding rules: Specification of Basic Encoding Rules (BER), Canonical Encoding Rules (CER) and Distinguished Encoding Rules (DER)"
    date: 02/2021
    target: https://handle.itu.int/11.1002/1000/14472
  X9.62:
    title: "ANSI X9.62-1998 | Public Key Cryptography For The Financial Services Industry: The Elliptic Curve Digital Signature Algorithm"
    date: 1998


informative:
  ISD-AS-assignments:
    title: "SCION Registry"
    date: 2026
    target: http://scion.org/registry/
  RFC8210:
  BARRERA17: DOI.10.1145/3085591
  CHUAT22:
    title: "The Complete Guide to SCION"
    date: 2022
    target: https://doi.org/10.1007/978-3-031-05288-0
    seriesinfo:
      ISBN: 978-3-031-05287-3
    author:
      -
        ins: L. Chuat
        name: Laurent Chuat
        org: ETH Zuerich
      -
        ins: M. Legner
        name: Markus Legner
        org: ETH Zuerich
      -
        ins: D. Basin
        name: David Basin
        org: ETH Zuerich
      -
        ins: D. Hausheer
        name: David Hausheer
        org: Otto von Guericke University Magdeburg
      -
        ins: S. Hitz
        name: Samuel Hitz
        org: Anapaya Systems
      -
        ins: P. Mueller
        name: Peter Mueller
        org: ETH Zuerich
      -
        ins: A. Perrig
        name: Adrian Perrig
        org: ETH Zuerich

--- abstract

This document presents the trust concept and design of the SCION *Control Plane Public Key Infrastructure (CP-PKI)*. SCION (Scalability, Control, and Isolation On Next-generation networks) is a path-aware, inter-domain network architecture that relies on the CP-PKI to handle cryptographic material, authenticate control plane messages used to securely disseminate path information.

This specification introduces its localized trust model, anchored in Isolation Domains (ISDs). It defines the distinct certificate types, and specifies the structure, format and lifecycle of the Trust Root Configuration (TRC). Furthermore, it provides practical guidelines for deploying and maintaining the CP-PKI infrastructure.

This document contains new approaches to secure path aware networking. It is not an Internet Standard, has not received any formal review of the IETF, nor was the work developed through the rough consensus process. The approaches offered in this work are offered to the community for its consideration in the further evolution of the Internet.

--- middle

# Introduction

SCION is a path-aware internetworking routing architecture as described in {{RFC9217}}. A more detailed introduction, motivation, and problem statement are provided in {{I-D.dekater-scion-controlplane}}. Readers are encouraged to read the introduction in that document first.

SCION relies on three main components:

*PKI* - providing cryptographic material within an unique trust model. It is described in this document.

*Control Plane* - performing inter-domain routing by discovering and securely disseminating path information. It is described in {{I-D.dekater-scion-controlplane}}.

*Data Plane* - carrying out secure packet forwarding between SCION-enabled ASes over paths selected by endpoints. It is described in {{I-D.dekater-scion-dataplane}}.


## Terminology

**Control Plane PKI (CP-PKI)**: It is the public key infrastructure upon which SCION's Control Plane relies for the authentication of messages. It is a set of policies, roles, and procedures that are used to manage Trust Root Configurations (TRCs) and certificates.

**SCION Autonomous System (AS)**: A SCION Autonomous System is a network under a common administrative control. For example, the network of a SCION service provider, company, or university can constitute an AS. While functionally similar to a BGP AS, a SCION AS operates within an Isolation Domain (ISD), utilizes a different address scheme, and serves as a locator in the addressing of end hosts. References to ASes throughout this document refer to SCION ASes.

**Isolation Domain (ISD)**: SCION ASes are organized into logical groups called Isolation Domains or ISDs. Each ISD consists of ASes that span an area with a uniform trust environment (e.g. a common jurisdiction).

**Core AS**: Each Isolation Domain (ISD) is administered by a set of distinguished SCION Autonomous Systems (ASes) called core ASes, which are responsible for initiating the path discovery and path construction process (called "beaconing" in SCION). Each ISD has at least one Core AS.

**Trust Root Configuration (TRC)**: A Trust Root Configuration or TRC is a signed collection of certificates pertaining to an Isolation Domain (ISD). TRCs also contain ISD-specific policies.

**Authoritative AS**: Authoritative ASes are those ASes in an ISD that always have the latest TRCs of the ISD. As a consequence, authoritative ASes also start the announcement of a TRC update.

**Base TRC**: A base TRC is a Trust Root Configuration (TRC) that other parties trust axiomatically. In other words, trust for a base TRC is assumed, not derived from another cryptographic object. ISD operators create and sign a base TRC when the ISD is established. A base TRC is either the first TRC of the ISD or the result of a trust reset.

**TRC Signing Ceremony**: The ceremony during which the very first base TRC of an ISD, called the initial TRC, is signed. The initial TRC is a special case of the base TRC where the number of the ISD is assigned.

**TRC Update**: A *regular* TRC update is a periodic re-issuance of the TRC where the entities and policies listed in the TRC remain unchanged. A *sensitive* TRC update is an update that modifies critical aspects of the TRC, such as the set of core ASes. In both cases, the base TRC remains unchanged.

**Voting ASes**: Those ASes within an ISD that may sign TRC updates. The process of appending a signature to a new TRC is called "casting a vote".

**Voting Quorum**: The voting quorum is a Trust Root Configuration (TRC) field that indicates the number of votes (signatures) needed on a successor TRC for it to be verifiable. A voting quorum greater than one will thus prevent a single entity from creating a malicious TRC update.

**Grace Period**: The grace period is an interval during which the previous version of a trust root configuration (TRC) is still considered active after a new version has been published.

**Trust Reset**: A trust reset is the action of announcing a new base TRC for an existing ISD, to mitigate a compromised TRC.

## Conventions and Definitions

{::boilerplate bcp14-tagged}


## Trust Model

Given the diverse nature of the constituents in the current Internet, an important challenge is how to scale authentication of network elements (such as AS ownership, hop-by-hop routing information, name servers for DNS, and domains for TLS) to the global environment. The roots of trust of currently prevalent public key infrastructure (PKI) models do not scale well to a global environment because (1) mutually distrustful parties cannot agree on a single trust root (monopoly model), and because (2) the security of a plethora of roots of trust is only as strong as its weakest link (oligopoly model) - see also {{BARRERA17}}.

The monopoly model suffers from two main drawbacks: First, all parties must agree on a single root of trust. Secondly, the single root of trust represents a single point of failure, the misuse of which enables the forging of certificates. Its revocation can also result in a kill switch for all the entities it certifies.

The oligopoly model relies on several roots of trust, all equally and completely trusted. However, this is not automatically better: whereas the monopoly model has a single point of failure, the oligopoly model has the drawback of exposing more than one point of failure.

Thus, there is a need for a trust architecture that supports meaningful trust roots in a global environment with inherently distrustful parties. This new trust architecture should provide the following properties:

- Trust agility (see further below);
- Resilience to single root of trust compromise;
- Multi-party governance; and
- Support for policy versioning and updates.

Ideally, the trust architecture allows parties that mutually trust each other to form their own trust domain, and to freely decide whether to trust other trust domains.

To fulfill the above requirements, which in fact apply well to inter-domain networking, SCION introduces the concept of **Isolation Domains**. An Isolation Domain (ISD) is a building block to support heterogeneous trust while achieving high availability and scalability in its control plane ({{I-D.dekater-scion-controlplane}}). It consists of a logical grouping of SCION ASes that share a uniform trust environment (i.e. a common jurisdiction).

An ISD is governed by one or multiple ASes, known as the voting ASes. Furthermore, each ISD has a set of ASes that form the ISD core, known as the core ASes. The set of core and voting ASes may be, but do not necessarily have to be the same ASes. Governance is implemented by a policy called the **Trust Root Configuration** (TRC), which is negotiated by the voting ASes, and which defines the locally scoped roots of trust used to validate bindings between names and public keys.

Authentication in SCION is based on X.509 certificates that bind identifiers to public keys and carry digital signatures that are verified by roots of trust. SCION allows each ISD to define its own set of trust roots, along with the policy governing their use. Such scoping of trust roots within an ISD improves security as compromise of a private key associated with a trust root cannot be used to forge a certificate outside the ISD. An ISD's trust roots and policy are encoded in the TRC, which has a version number, a list of public keys that serves as root of trust for various purposes, and a voting quorum governing the number of signatures required to update TRCs. The TRC serves as a way to bootstrap all authentication within SCION. Additionally, TRC versioning is used to efficiently revoke compromised roots of trust.

The TRC also provides *trust agility* - enabling users to select the trust roots used to initiate certificate validation. This implies that users are free to choose an ISD they believe maintains an uncompromised set of trust roots. ISD members can also decide whether to trust other ISDs and thus transparently define trust relationships between parts of the network. The SCION trust model therefore, differs from the one provided by other PKI architectures.

The need for trust agility also means that SCION does not by design provide IP prefix origin validation as provided by RPKI {{RFC8210}}. RPKI's trust model is currently reliant on the trust roots provided by the five Regional Internet Registries, and therefore outside of the governance of an ISD.


## Trust Relations within an Isolation Domain {#trust-relations}

The Control Plane PKI is organized at an ISD level whereby each ISD can independently specify its own Trust Root Configuration (TRC) and build its own verification chain. Each TRC consists of a collection of signed root certificates which are used to sign issuing CA certificates, which are in turn used to sign AS certificates. The TRC also includes ISD policies that specify, for example, the TRC's usage, validity, and future updates. The so-called base TRC constitutes the ISD's trust anchor which is signed during a signing ceremony by the voting ASes and then distributed throughout the ISD.

While it is not necessary that all the ASes of the ISD trust each other, all ASes MUST trust the ISD's core ASes, authoritative ASes, voting ASes, as well as its CA(s).

### Updates and Trust Resets {#trust-reset}

There are two types of TRC updates: regular and sensitive (see [](#update)). A **regular TRC update** is a periodic re-issuance of the TRC where the entities and policies listed in the TRC remain unchanged, whereas a **sensitive TRC update** is an update that modifies critical aspects of the TRC, such as the set of core ASes. In both cases the base TRC remains unchanged.

If the ISD's TRC has been compromised, it is necessary for an ISD to re-establish the trust root. This is possible with a process called **trust reset** (if permitted by the ISD's trust policy). In this case, a new base TRC is created.


### Substitutes to Certificate Revocation {#substitutes-to-revocation}

The Control Plane PKI does not explicitly support certificate revocation. Instead it relies on the two mechanisms described above and on short-lived certificates. This approach constitutes an attractive alternative to a revocation system for the following reasons:

- Both short-lived certificates and revocation lists must be signed by a CA. Instead of periodically signing a new revocation list, the CA can re-issue all the non-revoked certificates. Although the overhead of signing multiple certificates is greater than that of signing a single revocation list, the overall complexity of the system is reduced. In the Control Plane PKI the number of certificates that each CA must renew is manageable as it is limited to at most the number of ASes within an ISD. The absence of CRL/OCSP checks improves performance by removing additional network lookups during PKI processing.
- Even with a revocation system, a compromised key cannot be instantaneously revoked. Through their validity period, both short-lived certificates and revocation lists implicitly define an attack window (i.e. a period during which an attacker who managed to compromise a key could use it before it becomes invalid). In both cases, the CA must consider a tradeoff between efficiency and security when picking this validity period.

## Overview of Certificates, Keys, and Roles

The base TRC constitutes the root of trust within an ISD. {{figure-1}} provides a view of the trust chain within an ISD, based on its TRC. For detailed descriptions, please refer to [](#cert-specs) and [](#trc-specification).

~~~aasvg

               +----------------------------------------+
               |                 TRC 2                  |
               | +------------------------------------+ |
               | |- Version       - Core ASes         | |
+--------+     | |- ID            - Description       | |    +--------+
| TRC 1  |     | |- Validity      - No Trust Reset    | |    | TRC 3  |
| (Base  |---->| |- Grace Period  - Voting Quorum     | |--->|        |
|  TRC)  |     | |- ...                               | |    |        |
+--------+     | +------------------------------------+ |    +--------+
               |                                        |
               | +----------------+  +----------------+ |
               | | Regular Voting |  |Sensitive Voting| |
               | |  Certificate   |  |  Certificate   | |
               | +----------------+  +----------------+ |
               |                                        |
               | +----------------+  +----------------+ |
               | |     Votes      |  |   Signatures   | |
               | +----------------+  +----------------+ |
               |                                        |
               | +------------------------------------+ |
               | |        CP Root Certificates        | |
               | +------+---------------------+-------+ |
               |        |                     |         |
               +--------+---------------------+---------+
                        |                     |
                        |                     |
                        v                     v
                +-------------+        +-------------+
                |CP Issuing CA|        |CP Issuing CA|
                | Certificate |        | Certificate |
                +---+-------+-+        +------+------+
                    |       |                 |
                    |       |                 |
                    v       v                 v
           +-----------+ +-----------+  +-----------+
           |   CP AS   | |   CP AS   |  |   CP AS   |
           |Certificate| |Certificate|  |Certificate|
           +-----------+ +-----------+  +-----------+

~~~~
{: #figure-1 title="Chain of trust within an ISD"}

All certificates used in the Control Plane PKI are in X.509 v3 format {{RFC5280}} and additionally the TRC contains self-signed certificates instead of plain public keys. Self-signed certificates have the following advantages over plain public keys: (1) They make the binding between name and public key explicit; and (2) the binding is signed to prove possession of the corresponding private key. The public keys of voting AS certificates must therefore be explicitly verified during the signing ceremony ([](#initial-ceremony)) that is used to bootstrap trust for the initial TRC.

SCION ASes sign and verify control plane messages. Certain ASes have additional roles:

- **Core ASes**: They are a distinct set of ASes in the SCION Control Plane. For each ISD, the core ASes are listed in the TRC and each core AS has links to the other core ASes (in the same or in different ISDs).
- **Certification authorities (CAs)**: CAs are responsible for issuing AS certificates to other ASes and/or themselves.
- **Voting ASes**: They may sign TRC updates. The process of appending a signature to a new TRC is called "casting a vote", and the designated ASes that hold the private keys to sign a TRC update are "voting ASes".
- **Authoritative ASes**: They always have the latest TRCs of the ISD. They start the announcement of a TRC update.


# Certificate Specification {#cert-specs}

There are three types of control plane (CP) certificates: root certificates, issuing CA certificates, and AS certificates. Together, they build a chain of trust that is anchored in the Trust Root Configuration (TRC) file of the respective Isolation Domain (ISD). Additionally, there are regular and sensitive voting certificates which define the keys to cast votes in a regular or sensitive TRC update.

All certificates in the Control Plane PKI are in X.509 v3 format {{RFC5280}}.

The trust is anchored in the TRC for each ISD. The trust root is axiomatic: All trust derived from this anchor relies on all parties transitively trusting the TRC.


## Control Plane Root Certificate {#cp-root-cert}

The private key of the control plane root certificate is used to sign control plane issuing CA certificates. Consequently, the public key of the control plane Root certificate is used to verify control plane issuing CA certificates, i.e. root certificates determine which ASes act as a CA in an ISD.

In X.509 terms, control plane root certificates are CA certificates. For simplicity, this document calls them 'root certificates', distinguishing them from the subordinate 'issuing CA certificates'. Root certificates are self-signed; the issuer and subject are the same entity, and the public key within the certificate is used to verify its own signature. The public key of the control plane root certificate and proof of ownership of the private key are embedded in the TRC of an ISD, via the self-signed control plane root certificate. This facilitates the bootstrapping of trust within an ISD, and marks the control plane root certificates as the starting point of an ISD's certificate verification path.

The RECOMMENDED maximum validity period of a control plane root certificate is 5 years.

Note that the TRC of each ISD contains a trusted set of control plane root certificates, and this set builds the root of each ISD's verification path. For more information on the selection of this trusted set of root certificates, see [](#trc-specification).

## Control Plane Issuing CA Certificate

The private key of the control plane issuing CA certificate is used to sign control plane AS certificates. Consequently, control plane issuing CA certificates holding the public key of the control plane CA are used to verify control plane AS certificates.

The public key needed to verify the issuing CA certificate is in a control plane root certificate. Issuing CA certificates do not bundle the root certificate needed to verify them. In order to verify an issuing CA certificate, a pool of root certificates must first be extracted from one or more active TRCs (as described in [](#signing-verifying-cp-messages)).

The RECOMMENDED maximum validity period of a control plane issuing CA certificate is 15 days.
This is much shorter than root certificates, which have a longer recommended maximum validity period, because they are part of the TRC of an ISD, which itself also has a longer recommended maximum validity (see {{table-2}}). This ensures that the TRC need not be updated all the time and is thus relatively stable.

## Control Plane AS Certificate

SCION ASes sign control plane messages, such as Path Construction Beacons, with their AS private key. Consequently, control plane AS certificates holding the corresponding AS public key are required to verify control plane messages.

In X.509 terms, control plane AS certificates are end entity certificates. That is, they cannot be used to verify other certificates.

The RECOMMENDED maximum validity period of a CP AS certificate is 3 days.

## Voting Certificates {#cp-voting-cert}

There are two types of voting certificates: the (1) regular voting certificates and the (2) sensitive voting certificates. They contain the public keys associated with the private keys that may cast votes in the TRC update process.

Regular and sensitive voting certificates are used to verify regular and sensitive TRC updates respectively, and are embedded in the TRC.

### Regular Voting Certificate

Regular voting certificates may be used to cast a vote in a regular update. In X.509 terms, regular voting certificates are self-signed end-entity certificates. This means that the issuer and subject of a regular voting certificate are the same entity, and the public key within the certificate can be used to verify the certificate's signature. However, a regular voting certificate cannot be used to verify other certificates.

The RECOMMENDED maximum validity period of a regular voting certificate is 5 years.

### Sensitive Voting Certificate

Sensitive voting certificates may be used to cast a vote in a sensitive update. In X.509 terms, sensitive voting certificates are self-signed end-entity certificates. This means that the issuer and subject of a sensitive voting certificate are the same entity, and the public key within the certificate can be used to verify the certificate's signature. However, a sensitive voting certificate cannot be used to verify other certificates.

The RECOMMENDED maximum validity period of a sensitive voting certificate is 5 years.

## Key Pairs Overview and Notations {#key-pair-notation}

{{table-2}} and {{table-3}} below provide an overview of certificates and corresponding key pairs.


| Name                 | Notation (1)         | Used to verify/sign        |
|----------------------+----------------------+----------------------------|
| Sensitive voting key | K<sub>sens</sub>     | TRC updates (sensitive)    |
| Regular voting key   | K<sub>reg</sub>      | TRC updates (regular)      |
| CP root key          | K<sub>root</sub>     | CP issuing CA certificates         |
| CP CA key            | K<sub>CA</sub>       | CP AS certificates         |
| CP AS key            | K<sub>AS</sub>       | CP messages, path segments |
{: #table-2 title="Key chain"}

(1)  K<sub>x</sub> = PK<sub>x</sub> + SK<sub>x</sub>, where x = certificate type, PK<sub>x</sub> = public key, and SK<sub>x</sub> = private key

| Name                   | Notation          | Signed with                              | Contains                                                | Validity (2) |
|------------------------+-------------------+------------------------------------------+---------------------------------------------------------+--------------|
| TRC (trust root conf.) | TRC               | SK<sub>sens</sub>, SK<sub>reg</sub> (1)  | C<sub>root</sub>, C<sub>sens</sub>, C<sub>reg</sub> (1) | 1 year       |
| Sensitive voting cert. | C<sub>sens</sub>  | SK<sub>sens</sub>                        | PK<sub>sens</sub>                                       | 5 years      |
| Regular voting cert.   | C<sub>reg</sub>   | SK<sub>reg</sub>                         | PK<sub>reg</sub>                                        | 5 years      |
| CP root certificate    | C<sub>root</sub>  | SK<sub>root</sub>                        | PK<sub>root</sub>                                       | 5 years      |
| CP issuing CA certificate      | C<sub>CA</sub>    | SK<sub>root</sub>                        | PK<sub>CA</sub>                                 | 15 days (3)  |
| CP AS certificate      | C<sub>AS</sub>    | SK<sub>CA</sub>                          | PK<sub>AS</sub>                                         | 3 days       |
{: #table-3 title="Certificates"}

(1) Multiple signatures and certificates of each type MAY be included in a TRC.<br>
(2) Recommended maximum validity period. Note that initial AS certificates may have a longer validity (e.g. 10-30 days) to allow for enough time for deployment.<br>
(3) A validity of 15 days with 8 days overlap between two issuing CA certificates is RECOMMENDED to enable the best possible operational procedures when performing an issuing CA certificate rollover.

{{figure-2}} shows the content of a base/initial TRC, and the relationship between a TRC and the five types of certificates. The initial signatures are replaced by those of the Regular Voting Certificates with the first regular update to the base TRC.

~~~aasvg
+----------------------------------------------+
|                    TRC 1                     |
|                (base/initial)                |
| +------------------------------------------+ |
| | - Version          - Core ASes           | |
| | - ID               - Description         | |
| | - Validity         - No Trust Reset      | |
| | - Grace Period     - Voting Quorum       | |
| | - ...                                    | |
| +------------------------------------------+ |
|                                              |	
| +-------------------+ +--------------------+ |
| |        Votes      | |   Regular Voting   | |
| |  (cert. indices)  | |    Certificates    | |
| |                   | |  +-----+ +-----+   | |
| |       (empty)     | |  | (1) | | (2) |   | |
| |                   | |  |  C  | |  C  |   | |
| |                   | |  | reg | | reg |   | |
| |                   | |  +-----+ +-----+   | |
| +-------------------+ +--------------------+ |
|                                              |
| +--------------------+ +-------------------+ |
| |     Signatures     | | Sensitive Voting  | |
| | +----------------+ | |    Certificates   | |
| | | 73 A9 4E AO ...| | |                   | |
| | +----------------+ | | +-----+ +-----+   | |
| |         ...        | | | (3) | | (4) |   | |
| | +----------------+ | | |  C  | |  C  |   | |
| | | 53 B7 7C 98 ...| | | | sens| | sens|   | |
| | +----------------+ | | +-----+ +-----+   | |
| +--------------------+ +-------------------+ |
|                                              |	
| +------------------------------------------+ |
| |          CP Root Certificates            | |
| |                                          | |
| | +-----+ +-----+ +-----+ +-----+          | |
| | | (5) | | (6) | | (7) | | (8) |          | |
| | |  C  | |  C  | |  C  | |  C  |          | |
| | | root| | root| | root| | root| ...      | |
| | +-----+ +--+--+ +-----+ +--+--+          | |
| +------------+---------------+-------------+ |
+--------------+---------------+---------------+
               |               |
               v               v
     +-------------+       +-------------+
     |CP Issuing CA|       |CP Issuing CA|
     | Certificate |       | Certificate |
     +------+------+       +------+------+
            |                     |
            v                     v
      +-----------+         +-----------+
      |   CP AS   |         |   CP AS   |
      |Certificate|         |Certificate|
      +-----------+         +-----------+

~~~~
{: #figure-2 title="TRC and the different types of associated certificates. Arrows indicate the certificate hierarchy."}

## X.509 Certificate Profiles and Constraints

Whilst the certificates used in the Control Plane PKI are X.509 v3 certificates, this specification is more restrictive. This section defines these additional constraints and conditions in comparison to {{RFC5280}}, which apply to all SCION certificate types. For the baseline X.509 v3 format, refer to {{RFC5280}} and {{X.509}} Clause 7.2.

The following subsections define the specific constraints for the fields contained in the `TBSCertificate` sequence.

### `version`

The `version` field describes the version of the encoded certificate. It MUST be set to "v3" because extensions are required.

### `serialNumber`

The `serialNumber` field contains a positive integer assigned by the CA to each certificate. It MUST be unique for each certificate issued by a given CA.

### `signature` {#certsign}

The `signature` field contains the identifier for the signature algorithm used by the CA to sign the certificate. Current implementations use the ECDSA signature algorithm defined in {{X9.62}}.
As a consequence, the `parameters` field in the `AlgorithmIdentifier` sequence MUST NOT be used.

The Object Identifiers (OIDs) for ECDSA are defined as `ecdsa-with-SHA256`, `ecdsa-with-SHA384`, and `ecdsa-with-SHA512` in {{RFC5758}}.

SCION implementations MUST include support for the ECDSA curves below.

- NIST P-256 (NISTFIPS186-4, section D.1.2.3) (named `secp256r1` in {{RFC5480}})
- NIST P-384 (NISTFIPS186-4, section D.1.2.4) (named `secp384r1` in {{RFC5480}})
- NIST P-521 (NISTFIPS186-4, section D.1.2.5) (named `secp521r1` in {{RFC5480}})

The OIDs for the above curves are specified in section 2.1.1.1 of {{RFC5480}}.

Other algorithms or curves MAY be employed. Implementations deviating from the mandatory set generally lose the guarantee of global interoperability and are suitable primarily for isolated ISDs that do not require external interconnection. Future protocol versions may update the set of mandatory algorithms.

The appropriate hash size to use when producing a signature with an ECDSA key is:

- ECDSA with SHA-256, for a P-256 signing key
- ECDSA with SHA-384, for a P-384 signing key
- ECDSA with SHA-512, for a P-521 signing key

### `issuer` {#issuer}

The `issuer` field contains the distinguished name (DN) of the entity that has issued and signed the certificate (usually a CA). This field MUST be non-empty.
In addition to the attributes described in {{RFC5280}} section 4.1.2.4, SCION implementations MUST also support the SCION-specific `id-at-ia` attribute identifying the SCION ISD and AS numbers.

#### `id-at-ia` Attribute {#isd-as-nr}

The `id-at-ia` attribute identifies the SCION ISD and AS numbers. Its object identifier is defined in [](#cert-asn1).

The `id-at-ia` attribute MUST be included in the `issuer` and `subject` fields of root, issuing CA, and AS certificates. It SHOULD be included in voting certificates.
In issuing CA certificates, the presence of this attribute ensures that the Control Plane can identify from which AS to retrieve the intermediate CA certificate, thereby avoiding circular dependencies.

When present, the `id-at-ia` attribute MUST appear exactly once in a given distinguished name (DN), and implementations MUST reject certificates if the `id-at-ia` appears more than once.

The string representation of the `id-at-ia` attribute MUST follow the formatting defined in {{I-D.dekater-scion-controlplane}}, section "Text Representation" where AS numbers in the lower 32-bit range are represented in decimal notation, and others in hexadecimal notation.


### `validity`

The `validity` field defines the validity period of the certificate. All certificates MUST have a well-defined expiration date. `GeneralizedTime` value "99991231235959Z" MUST NOT be used.
The recommended maximum validity period for each type of certificate is described in [](#key-pair-notation). SCION implementations SHOULD adopt these values.

### `subject`

The `subject` field defines the entity that owns the certificate. It MUST NOT be empty.
The same constraints as the `issuer` field apply. For details, see [](#issuer) and [](#isd-as-nr).

### `subjectPublicKeyInfo`

The `subjectPublicKeyInfo` field carries the public key of the certificate's subject (the entity that owns the certificate, as defined in the `subject` field). The `subjectPublicKeyInfo` field also identifies which algorithm to use with the key.

  - **SCION constraints**: For constraints regarding the algorithm, see the `signature` field.

### `issuerUniqueID`

The `issuerUniqueID` field MUST NOT be used.

### `subjectUniqueID`

The `subjectUniqueID` field MUST NOT be used.


## Extensions {#exts}

{{RFC5280}}, section 4.2.1, defines the syntax of the `Extensions` sequence in a X.509 certificate. Descriptions of each standard certificate extension can be found in {{RFC5280}}, section 4.2.1. The corresponding clauses in {{X.509}} are clause 7.2 and clause 9, respectively.

The following extensions are relevant for the SCION PKI:

- `authorityKeyIdentifier`
- `subjectKeyIdentifier`
- `keyUsage`
- `extKeyUsage`
- `basicConstraints`

The following sections describe the SCION-specifics in regard to these extensions.

### `authorityKeyIdentifier` Extension

The `authorityKeyIdentifier` extension identifies the public key corresponding to the private key used to sign a certificate. For its syntax and definition, see {{RFC5280}}, section 4.2.1.1, and {{X.509}}, clause 9.2.2.1.

To ensure deterministic matching, the `authorityKeyIdentifier` attributes are strictly restricted:

- `keyIdentifier`: MUST be included.
- `authorityCertIssuer` & `authorityCertSerialNumber`: MUST NOT be included. Implementations MUST return an error if either is present.

This extension MUST be marked as non-critical. Implementations MUST return an error if the extension is not present AND the certificate is not self-signed.

### `subjectKeyIdentifier` Extension {#subject-key-id-ext}

The `subjectKeyIdentifier` extension identifies certificates that contain a particular public key. It can be used, for example, by control plane messages to identify which certificate to use for verification. The extension allows for overlapping control plane CA keys, for example during updates.

For the syntax and definition of the `subjectKeyIdentifier` extension, see {{RFC5280}}, section 4.2.1.2, and {{X.509}}, clause 9.2.2.2.

This extension MUST be marked as non-critical. Implementations MUST return an error if the extension is not present.

### `keyUsage` Extension {#key-usage-ext}

The `keyUsage` extension identifies the intended usage of the public key in the corresponding certificate. For the syntax and definition of the `keyUsage` extension, see {{RFC5280}}, section 4.2.1.3, and {{X.509}}, clause 9.2.2.3.

The attributes of the `keyUsage` extension define possible ways of using the public key. The attributes have the following meaning in SCION:

- `digitalSignature`: The public key can be used to verify the digital signature of a control plane payload.
- `keyCertSign`: The public key can be used to verify the CA signature on a control plane AS certificate.

Other attributes are not used.

If a relying party uses the certificate's public key to verify the signature of a control plane payload (`digitalSignature` attribute), the relying party MUST be able to trace back the private key used to sign the certificate. This is done by referencing the ISD-AS and the subject key identifier (via the `subjectKeyIdentifier` extension). For more information about the `subjectKeyIdentifier` extension (see [](#subject-key-id-ext)).

When present, this extension SHOULD be marked as critical.

Each Control Plane PKI certificate type uses the public key differently, and consequently also specifies the attributes of the `keyUsage` extension differently. The next table shows the specifications per certificate type.

| Certificate Type             | Root                     | Issuing CA                | AS                    | Voting               |
| ---------------------------- | ------------------------ | ------------------------- | --------------------- | -------------------- |
| *Attribute:*                 |                          |                           |                       |                      |
| `keyUsage` extension itself  | REQUIRED                 | REQUIRED                  | REQUIRED              | OPTIONAL             |
| `digitalSignature` bit       | MUST NOT be asserted (1) | MUST NOT be asserted (2)  | MUST be asserted      | MUST NOT be asserted |
| `keyCertSign` bit            | MUST be asserted         | MUST be asserted          | MUST NOT be asserted  | MUST NOT be asserted |
{: #table-4 title="keyUsage extension - Specifications per certificate type"}

(1)  Root certificates SHOULD NOT be used to verify control plane messages.<br>
(2)  Issuing CA certificates SHOULD NOT be used to verify control plane messages.


### `extKeyUsage` Extension {#ext-key-usage-ext}

The `extKeyUsage` extension specifies additional usages of the public key in the certificate. For the syntax and definition of the `extKeyUsage` extension, see {{X.509}}, clause 9.2.2.4.

SCION uses the following attributes of the Extended Key Usage extension, as defined in Section 4.2.1.12 of {{RFC5280}}:

- `id-kp-serverAuth`: If set, the public key can be used for SCION Control Plane server authentication.
- `id-kp-clientAuth`: If set, the public key can be used for SCION Control Plane client authentication.
- `id-kp-timeStamping`: If set, the public key can be used for the verification of timestamps.

Additionally, the Extended Key Usage extension sequence MAY include the SCION-specific attributes `id-kp-root`, `id-kp-regular`, and `id-kp-sensitive`. These attributes are used in the TRC setup to distinguish root certificates, regular voting certificates, and sensitive voting certificates from each other. For more information, see [](#cert).

The specifications of the `extKeyUsage` extension differ per SCION Control Plane PKI certificate type. The next table provides an overview of the specifications per certificate type.

| Certificate Type               | Root                   | Issuing CA            | AS                    | Voting                     |
| ------------------------------ | ---------------------- | --------------------- | --------------------- | --------------------------------- |
| *Attribute:*                   |                        |                       |                       |                                   |
| `extKeyUsage` extension itself | REQUIRED               | OPTIONAL              | REQUIRED              | REQUIRED                        |
| `id-kp-serverAuth`             | MUST NOT be included   | MUST NOT be included  | MUST be included, if the certificate is used on the server-side of a control plane TLS session. | MUST NOT be included |
| `id-kp-clientAuth`             | MUST NOT be included   | MUST NOT be included  | MUST be included, if the certificate is used on the client-side of a control plane TLS session. | MUST NOT be included |
| `id-kp-timeStamping`           | MUST be included       |                       | MUST be included      | MUST be included                  |
| SCION-specific attributes (see [](#specatt)) | `id-kp-root` MUST be included |  |                       | Regular voting cert: `id-kp-regular` MUST be included.<br> Sensitive voting cert: `id-kp-sensitive` MUST be included |
{: #table-5 title="extKeyUsage extension - Specifications per certificate type"}

Note that the use of `extKeyUsage` in Root certificates renders them incompatible with standard TLS handshakes according to {{RFC5280}}, because the `id-kp-serverAuth` attribute is not set. While current implementations follow what described in this document, the use of `extKeyUsage` should be revised in future protocol iterations.

#### SCION-Specific Key Purposes {#specatt}

Three additional key purpose attributes differentiate certificate roles within the CP-PKI:

- `id-kp-sensitive` (OID 1.3.6.1.4.1.55324.1.3.1): identifies sensitive voting certificate
- `id-kp-regular` (OID 1.3.6.1.4.1.55324.1.3.2): identifies a regular voting certificate
- `id-kp-root` (OID 1.3.6.1.4.1.55324.1.3.3): identifies a root certificate

The formal ASN.1 definitions for these attributes are provided in [](#cert-asn1).

### `basicConstraints` Extension {#basic-constr-ext}

The `basicConstraints` extension specifies whether the certificate subject acts as a CA. For the syntax and definition of the `basicConstraints` extension, see {{X.509}}, clause 9.4.2.1.

The `basicConstraints` extension includes the following attributes relevant for SCION:

- `cA` attribute: Specifies whether the certificate subject acts as a CA. If yes, this attribute MUST be asserted and the extension MUST be marked as critical.
- `pathLenConstraint` attribute: This attribute is only relevant if the `cA` attribute is set to TRUE and specifies the maximum number of CA certificates that may follow this CA certificate in the certification chain. Value "0" means that this CA may only issue end-entity certificates, but no CA certificates. If the attribute is not set, there is no limit to the maximum length of the certification path.

The settings of the `basicConstraints` extension differ for each SCION Control Plane PKI certificate type. The next table shows the specifications per certificate type.

| Certificate Type                    | Root                   | Issuing CA               | AS                            | Voting (regular and sensitive)    |
| ----------------------------------- | ---------------------- | ------------------------ | ----------------------------- | --------------------------------- |
| *Attribute:*                        |                        |                          |                               |                                   |
| `basicConstraints` extension itself | REQUIRED               | REQUIRED                 | SHOULD NOT be present         | SHOULD NOT be present             |
| `cA`                               | MUST be asserted        | MUST be asserted         | If the extension is present, this attribute MUST NOT be asserted | If the extension is present, this attribute MUST NOT be asserted |
| `pathLenConstraint`                | MUST be set to "1"    | MUST be set to "0" (1) | MUST NOT be included          | MUST NOT be included              |
{: #table-6 title="basicConstraints extension - Specifications per certificate type"}

(1) control plane CAs can only issue end-entity certificates.



# Trust Root Configuration Specification {#trc-specification}

The Trust Root Configuration (TRC) contains policy information about an ISD and acts as a distribution mechanism for the trust anchors of that ISD. It enables the securing of control plane interactions and is thus an integral part of the SCION infrastructure.

The initial TRC of an ISD is signed during a signing ceremony and then distributed throughout the ISD. This signing ceremony follows specific rules which are described in [](#trc-ceremony).

The TRC is a signed collection of {{X.509}} v3 certificates. Additionally, the TRC contains ISD-specific policies encoded in CMS signed-data ({{RFC5652}} section 5).

The TRC's certificates collection consists of a set of control plane root certificates which build the root of the certification chain for the AS certificates in an ISD. The other certificates in the TRC are solely used for signing the next TRC; a process called "voting". The verification of a new TRC thus depends on the policies and voting certificates defined in the previous TRC.

This section specifies the TRC including format definitions and payload fields. The section uses the ITU-T {{X.680}} syntax.

## TRC Types and States {#trc-states}

The following types of TRCs exist:

- Initial: The very first TRC of an ISD is the initial TRC of that ISD. It is a special case of the base TRC, where the number of the ISD is specified.
- Base: A base TRC is either the initial TRC, or the first TRC after a trust reset (see [](#trust-reset)). Trust for a base TRC cannot be inferred by verifying a TRC update; base TRCs are trusted axiomatically, similarly to how root certificates are trusted by clients in the Web PKI.
- Update: All non-base TRCs are updated TRCs. They are the product of either a regular or a sensitive update.

A TRC can have the following states:

- Valid: The validity period of a TRC is defined in the TRC itself, in the `validity` field (see [](#validity-trc)). A TRC is considered valid if the current time falls within its validity period.
- Active: An active TRC is a valid TRC that can be used for verifying certificate signatures. This is either the latest TRC or the predecessor TRC if it is still in its grace period (as defined in the `gracePeriod` field of the new TRC, see [](#grace)). No more than two TRCs can be active at the same time for any ISD.

{{figure-2}} shows the content of both a base/initial TRC, the changes made with the first regular update to the base TRC. All elements of the TRC is detailed in the following subsections.


## TRC Fields {#trcfields}

The TRC holds the root and voting certificates of the ISD, defining the ISD's trust policy. Its ASN.1 module is described in [](#trc-asn1).
Its fields are contained in a `TRCPayload` sequence. This section describes their syntax and semantics.

### `version`

The `version` field describes the version of the TRC format specification. It MUST be "v1".


### `iD` {#id}

The `iD` field contains an unique identifier for the TRC, constituted by a sequence of:

- ISD number (`iSD` attribute),
- base number (`baseNumber` attribute). It indicates the starting point of the current TRC update chain. This starting point is either the ISD's initial TRC or the currently valid base TRC, if the valid base TRC differs from the initial TRC. The latter is the case after a trust reset.
- TRC serial number (`serialNumber` attribute). It represents the current update cycle, counting from the initial TRC of a specific ISD.

All numbers MUST be positive integers.

A TRC where the base number is equal to the serial number is a base TRC. The initial TRC is a special case of a base TRC and MUST have a serial number of 1 and a base number of 1. With every TRC update, the serial number MUST be incremented by one which facilitates the unique identification of the predecessor and successor TRC in an update chain.

If a trust reset is necessary, a new base TRC is announced in order to start a new and clean TRC update chain. The base number of this new TRC update chain SHOULD be the number following the serial number of the latest TRC that was produced by a non-compromised TRC update for this ISD.

The following example illustrates how to specify the ID of the TRCs in a TRC update chain for *ISD 15*. The IDs are given in a human-readable notation, where Bxx is the base number, and Sxx the serial number.

| Update      | TRC ID              | Remarks                                          |
|-------------+---------------------+--------------------------------------------------|
| Initial     | ISD15-B01-S01       |                                                  |
| Regular     | ISD15-B01-S02       | Only the serial number is incremented.           |
| Regular     | ISD15-B01-S03       | Only the serial number is incremented.           |
| Sensitive   | ISD15-B01-S04       | Only the serial number is incremented.           |
| Trust reset | ISD15-**B05**-S05   | A trust reset includes the creation of a new base TRC. The new base number follows the serial number "04" of the latest TRC resulting from a non-compromised TRC update for this ISD. |
| Regular     | ISD15-B05-S06       | Only the serial number is incremented.           |
| And so on   |                     |                                                  |
{: #table-7 title="ID of TRCs in TRC update chain"}


### `validity` {#validity-trc}

The `validity` field defines the TRC validity period. The `notBefore` and `notAfter` attributes of the `validity` field specify the lower and upper bound of the time interval during which a TRC can be active.

An active TRC is a valid TRC that can be used for verifying certificate signatures. The time period during which a TRC is active can be shorter than the time period during which the TRC is valid. For more information, see [](#trc-states).

The `validity` field consists of a sequence of a `notBefore` and a `notAfter` date, both encoded as `GeneralizedTime`.
All TRCs MUST have a well-defined expiration date. SCION implementations MUST NOT create TRCs that use `GeneralizedTime` value "99991231235959Z", and verifiers MUST reject such a TRC.


### `gracePeriod` {#grace}

The `gracePeriod` field specifies the duration, in seconds, during which the predecessor TRC remains active after a new TRC is issued. This grace period starts at the beginning of the validity period of the new TRC.

A predecessor TRC ceases to be active when the earliest of the following events occurs:

- the grace period expires;
- the predecessor TRC reaches its expiration time (`notAfter`); or
- a subsequent TRC update (i.e., the successor to the new TRC) is announced.

In a base TRC, `gracePeriod` value MUST be zero. In a non-base TRC, `gracePeriod` SHOULD be greater than zero. The defined duration SHOULD provide sufficient overlap between the two TRCs to ensure uninterrupted operations within the ISD. If the grace period is too short, some control plane AS certificates may expire before the corresponding ASes can fetch an updated version from their CA.


### `noTrustReset` {#notrustreset}

The `noTrustReset` Boolean specifies whether a trust reset is forbidden by the ISD. Within a TRC update chain, this value MUST NOT be changed by a regular or sensitive update. However, it is possible to change the `noTrustReset` value in the event of a trust reset where a new base TRC is created.

The `noTrustReset` field is OPTIONAL and defaults to FALSE.

Note that once the `noTrustReset` Boolean is set to TRUE and a trust reset is disallowed, this cannot be reversed. Therefore, ISDs SHOULD always set this value to FALSE, unless they have sufficiently assessed the risks and implications of making a trust reset impossible.

Note that a trust reset represents a special use case where a new base TRC is created. It therefore differs from a TRC update (regular or sensitive) as the signatures in the new base TRC cannot be verified with the certificates contained in the predecessor TRC. Instead, a trust reset base TRC must be axiomatically trusted, similarly to how the initial TRC is trusted.


### `votes` {#votes}

The `votes` field contains a sequence of indices referencing the voting certificates in the predecessor TRC. If index i is part of the `votes` field, then the voting certificate at position i in the `certificates` sequence of the predecessor TRC casted a vote on the successor TRC. The index is 0-based, meaning that 0 represents the first element. For more information on the `certificates` sequence, see [](#cert).

In a base TRC, the `votes` sequence MUST be empty.
Every entry in the `votes` sequence MUST be unique.
Further restrictions on votes are discussed in [](#update).

The `votes` sequence MUST be present to prevent the stripping of voting signatures from the TRC. Without this sequence, an attacker could transform a TRC with more voting signatures than the voting quorum into multiple verifiable TRCs with the same payload, but different voting signature sets, which directly violates the uniqueness requirement of a TRC.


### `votingQuorum` {#quorum}

The `votingQuorum` field defines the number of necessary votes on a successor TRC to make it verifiable.

A voting quorum greater than one will prevent a single entity from creating a malicious TRC update.


### `coreASes` {#core}

The `coreASes` field contains a sequence listing the core AS numbers within the ISD.

Each AS number MUST be unique and encoded as a `PrintableString` using the formatting defined in {{I-D.dekater-scion-controlplane}}, section "Text Representation".

To assign or revoke core status, the target AS number is added to or removed from this sequence. For such modification, a sensitive TRC update is REQUIRED.

### `authoritativeASes` {#auth}

The `authoritativeASes` field contains a sequence listing the authoritative AS numbers in the ISD. Authoritative ASes are those ASes in an ISD that always possess the latest TRCs for the ISD and initiate TRC update announcements.

Every authoritative AS MUST be a core AS (i.e., be listed in the `coreASes` field). The encoding and uniqueness requirements for this sequence are identical to those of the `coreASes` field.

As with core ASes, assigning or revoking authoritative status is performed by adding or removing the target AS number from this sequence. For such modification, a sensitive TRC update is REQUIRED.


### `description` {#description}

The `description` field contains a UTF-8 encoded string that describes the ISD. It SHOULD NOT be empty.
The description MUST be in English and MAY additionally contain information in other languages.


### `certificates` {#cert}

The voting ASes and the certification authorities (CAs) of an ISD are not specified explicitly in the ISD's TRC. Instead, this information is defined by the list of voting and root certificates in the `certificates` field of the TRC payload.

The `certificates` field is a sequence of self-signed X.509 certificates. Each certificate in the certificate sequence MUST be one of the following types:

- a sensitive voting certificate,
- a regular voting certificate, or
- a CP root certificate.

A certificate that is no control plane root or voting certificate MUST NOT be included in the sequence of certificates in the `certificates` field.

A certificate's type (voting or root) is specified in the `extKeyUsage` extension of the certificate, by means of the SCION-specific attributes `id-kp-regular`, `id-kp-sensitive`, and `id-kp-root`, respectively. For more information, see [](#ext-key-usage-ext).

The following constraints must hold for each certificate in the `certificates` field of the TRC payload:

- Each certificate MUST be unique in the sequence of certificates.
- The `issuer` / `serialNumber` pair for each certificate MUST be unique.
- If an ISD-AS number is present in the distinguished name of the certificate, this ISD number MUST be equal to the ISD number of the TRC (which is defined in the `iD` field (see [](#id)).
- Every certificate MUST have a validity period that fully contains the validity period of this TRC. That is, the `notBefore` date of this TRC's validity period MUST be equal to or later than the certificate's `notBefore` date, and the `notAfter` date of this TRC's validity period MUST be before or equal to the certificate's `notAfter` date.
- Per certificate type, every certificate distinguished name MUST be unique.

The following must hold for the entire sequence of certificates in the `certificates` field:

- `votingQuorum` <= count (sensitive voting certificates) <br>
That is, the quorum defined in the TRC's `votingQuorum` field ([](#quorum)) MUST be smaller than or equal to the number of sensitive voting certificates specified in the TRC's `certificates` field.
- `votingQuorum` <= count (regular voting certificates) <br>
That is, the quorum defined in the TRC's `votingQuorum` field ([](#quorum)) MUST be smaller than or equal to the number of regular voting certificates specified in the TRC's `certificates` field.


## TRC Signature Syntax {#signed-format}

To guarantee the integrity and authenticity of the distributed trust anchors, each TRC is digitally signed using the Cryptographic Message Syntax (CMS). The signed TRC payload uses the CMS signed-data content type as specified in Section 5 of {{RFC5652}}, and is encapsulated in a CMS `ContentInfo` element, as defined in Section 3 of {{RFC5652}}.

For signature calculation, the data that is to be signed MUST be encoded using ASN.1 distinguished encoding rules (DER) {{X.690}}.


### SCION-specific rules

SCION implementations MUST fulfill the following additional rules, as well as the general syntax rules specified in {{RFC5652}}:

- `EncapsulatedContentInfo` sequence:
   - The `eContentType` field MUST be set to "id-data".
   - The content of the `eContent` field MUST be the DER-encoded TRC payload. This has the benefit that the format is backwards compatible with PKCS #7, as described in Section 5.2.1 of {{RFC5652}}.
- `SignedData` sequence:
   - The `certificates` field MUST be left empty. The certificate pool used to verify a TRC update is already specified in the `certificates` field of the predecessor TRC's payload (see also [](#cert)).
   - The `version` field MUST be set to "1". This is because SCION uses the "id-data" content type to encapsulate content info and does not specify any certificate in the `SignedData` sequence (see also Section 5.1 of {{RFC5652}}).
- `SignerIdentifier` choice:
   - The type of signer identifier chosen here MUST be `IssuerAndSerialNumber`.
- `SignerInfo` sequence:
   - The `version` field MUST be set to "1". This is because SCION uses the `IssuerAndSerialNumber` type of signer identifier (see also Section 5.3 of {{RFC5652}}).
   - The algorithm specified in the `signatureAlgorithm` field MUST be one of the algorithms supported by SCION. For details, see [signature Field - Additional Information](#certsign).
   - The `digestAlgorithm` is determined by the algorithm specified in the `signatureAlgorithm` field.


### TRC Equality

The signer information in the signed TRC is part of an unordered set, as per {{RFC5652}}. This implies that the signer information can be reordered without affecting verification, although certain operations require TRCs to be equal in accordance with the following definition:

**Two TRCs are equal, if and only if their payloads are byte equal.**

Two TRCs with byte equal payloads can be considered as equal because the TRC payload exactly defines which signatures must be attached in the signed TRC:

- The REQUIRED signatures from voting certificates are explicitly mentioned in the `votes` field of the payload: If index "i" is part of the `votes` field, then the voting certificate at position i in the `certificates` sequence of the predecessor TRC casted a vote on the successor TRC. See also [](#votes).
- The REQUIRED signatures for new certificates are implied by the currently valid TRC payload, and, in case of a TRC update, the predecessor payload.


## Certification Path - Trust Anchor Pool {#trc-selection}

The certification path of a Control Plane AS certificate starts in a Control Plane root certificate. While root certificates for a given ISD are distributed via the TRC, AS and issuing CA certificates are distributed separately. This separation makes it possible to extend the validity period of the root certificate, and to update the corresponding TRC without having to modify the certificate chain.

To validate a certification path, a relying party builds a collection of root certificates known as the trust anchor pool. Because TRC updates can introduce a grace period where multiple TRCs overlap, relying parties MUST execute the following steps to determine the correct trust anchor pool for a given verification time:

1. From the set of all available TRCs for the ISD, keep only TRCs whose validity start time (`notBefore` date) is at or before the verification time. If no such TRC exists, the process terminates unsuccessfully.

2. From the selected TRCs, identify those with the highest base number (`baseNumber`), then select the TRC among them with the highest serial number (`serialNumber`).

3. If the verification time is strictly greater than the selected TRC's `notAfter` date then the process terminates unsuccessfully.

4. If the TRC is valid, add its root certificates to the trust anchor pool.

5. If the TRC is in its grace period, add the preceding TRC's root certificates to the trust anchor pool.

Note that any entity sending information secured by the Control Plane PKI, such as control plane messages, MUST be able to provide all the necessary trust material including certificates to verify said information. If any cryptographic material is missing in the process, the relying party MUST query the originator of the message for the missing material through the control plane API described in {{I-D.dekater-scion-controlplane}}, section "Distribution of Cryptographic Material". If it cannot be resolved, the verification process fails. For more details, see 4.2 "Signing and Verifying Control Plane Messages" [](#signing-verifying-cp-messages).

## TRC Updates {#update}

All non-base TRCs of an ISD are updates of the ISD's base TRC(s) and constitute a chain. Updates are categorized as regular or sensitive, depending on which payload fields are being modified.

This section describes the rules that apply to updating a TRC in regard to the payload information contained in the TRC. Some rules are valid for both update types whilst some only apply to a regular or a sensitive TRC update. Based on the type of update, different sets of voters are needed to create a verifiable TRC update and the corresponding voting (signing) process is also described. Finally, this section describes checks to verify a newly issued TRC.


### Changed or New Certificates {#change-new}

In the context of a TRC update,

- A certificate is changing if the certificate is part of the `certificates` sequence in the predecessor TRC, but no longer part of the `certificates` sequence in the updated TRC. Instead, the `certificates` sequence of the updated TRC holds another certificate of the same type and with the same distinguished name.
- A certificate is new if there is no certificate of the same type and distinguished name at all in the `certificates` sequence of the predecessor TRC.

Every new sensitive or regular voting certificate in a TRC attaches a signature to the TRC. This is done to ensure that the freshly included voting entity agrees with the contents of the TRC it is now part of.


### Update Rules - Overview

The following table gives an overview of the types of TRC update, as well as the rules that must apply in regard to the updated TRC's payload information.

The sections that follow provide more detailed descriptions of each rule.


| Type of TRC Update                 |                 Unchanged Elements  |       Changed Elements |    Other Rules to Hold            |
|------------------------------------+-------------------------------------------+----------------------------------------+-----------------------------------------------------|
| Both Regular AND Sensitive         | - `iD` field: `iSD` and `baseNumber` <br> - `noTrustReset` field | `iD` field: `serialNumber` MUST be incremented by 1 | `votes` field: Number of votes (indices) => number set in the `votingQuorum` field of the predecessor TRC |
| Regular                            | - Quorum in the `votingQuorum` field<br>- Core ASes in the `coreASes` field<br>- ASes in the `authoritativeASes` field<br>- Nr. and distinguished names of root & voting certificates in the `certificates` field<br>- Set of sensitive voting certificates in the `certificates` field | | `votes` field:<br> - All votes MUST only refer to *regular* voting certificates in the predecessor TRC<br>- MUST include votes of each changed regular voting certificate from the predecessor TRC<br> `signatures` field:<br> - MUST include signatures of each changed root certificate from the predecessor TRC |
| Sensitive                          | If the update does not qualify as a regular update, it is a sensitive update |  | `votes` field: <br> - All votes MUST only refer to *sensitive* voting certificates in the predecessor TRC |
{: #table-8 title="Overview of the update types and corresponding rules"}


### General Update Rules

The following rules hold for each updated TRC, independent of the update type:

- The `iSD` and `baseNumber` in the `iD` field MUST NOT change (see also [](#id)).
- The `serialNumber` in the `iD` field MUST be incremented by one.
- The `noTrustReset` field MUST NOT change (see also [](#notrustreset)).
- The `votes` sequence of the updated TRC MUST only contain indices that refer to sensitive or regular voting certificates in the predecessor TRC. This guarantees that the updated TRC only contains valid votes authenticated by sensitive or regular voting certificates in the predecessor TRC. For more information, see [](#votes) and [](#cert).
- The number of votes in the updated TRC MUST be greater than or equal to the number set in the `votingQuorum` field of the predecessor TRC (see [](#quorum)). The number of votes corresponds to the number of indices in the `votes` field of the updated TRC.


### Regular TRC Update

A regular TRC update is a periodic re-issuance of the TRC where the entities and policies listed in the TRC remain unchanged.

A TRC update qualifies as a regular update if the following rules apply in regard to the TRC's payload information.

- The following fields in the updated TRC MUST remain the same compared to the predecessor TRC:
   - The voting quorum set in the `votingQuorum` field.
   - The core ASes specified in the `coreASes` field.
   - The authoritative ASes specified in the `authoritativeASes` field.
   - The number of sensitive and regular voting certificates as well as control plane root certificates included in the `certificates` field and their distinguished names.
   - The set of sensitive voting certificates specified in the `certificates` field.
- For every regular voting certificate that changes, the regular voting certificate in the predecessor TRC is part of the voters on the updated TRC. That is, for each changed regular voting certificate, an index in the `votes` field of the updated TRC MUST refer to the changed regular voting certificate in the predecessor TRC.
- For every control plane root certificate that changes, the updated TRC MUST include a signature created with the private key belonging to the changed control plane root certificate (which is part of the predecessor TRC).
- In order for a regular TRC update to be verifiable, all votes MUST be cast by *regular* voting certificates. That is, each index in the `votes` field of the regularly updated TRC MUST refer to a *regular* voting certificate in the `certificates` field of the predecessor TRC.


### Sensitive TRC Update

If a TRC update does not qualify as a regular update, it is considered a sensitive update.

- In order for a sensitive update to be verifiable, all votes MUST be cast by *sensitive* voting certificates. That is, each index in the `votes` field of the sensitively updated TRC MUST refer to a *sensitive* voting certificate in the `certificates` field of the predecessor TRC.


### Signing a TRC Update

As described above, a set of voters MUST cast votes on the updated TRC to make it verifiable. The `votingQuorum` field of the predecessor TRC (see [](#quorum)) defines the required number of voters, which will represent regular or sensitive voting certificates, respectively.

Furthermore, if one or more *new* certificates are added to the updated TRC, the corresponding voting representatives MUST also sign the updated TRC in order to show that they have access to the private keys listed in these fresh certificates. This is called "showing proof-of-possession" and is done by signing the TRC with the respective private key. For the distinction between changed and new certificates in a TRC update, see [](#change-new).

It is up to the ISD members to decide how the "casting a vote" procedure for updated TRCs will take place. Some ISDs make a distinction between regular and sensitive updates by dividing the regular and sensitive signing keys in different security classes, e.g. they keep the regular key in an online vault while the sensitive key would be stored offline. This way, the regular TRC update would lend itself to being automated (since the keys are accessible online) whereas the sensitive one would require manual actions to access the offline key. Other ISDs keep both regular and sensitive keys online and perform both updates automatically.


### TRC Update Verification {#trc-verification}

To verify a TRC update, the relying party MUST perform the following checks:

- Check that the specified update rules as described above are respected.
- Check that all signatures are verifiable and no superfluous signatures are attached.
- In case of a regular update:
    - check that the signatures for the changing certificates are present and verifiable, and
    - check that all votes are cast by a regular voting certificate.
- In case of a sensitive update:
    - check that all votes are cast by a sensitive voting certificate.

If one or more of the above checks gives a negative result, the updated TRC SHOULD be rejected.


## Initial TRC Signing Ceremony {#trc-ceremony}

The very first base TRC of an ISD - called the initial TRC - is a special case of the base TRC. The initial TRC MUST be signed during a signing ceremony where all voting representatives of the initial TRC take part to sign the TRC and exchange their public keys. Following this, all entities within an ISD can obtain the TRC by means of a secure offline or online mechanism.

[](#initial-ceremony) describes a possible procedure for the signing ceremony of an ISD's initial TRC. Whilst it is up to the initial members of an ISD how to organize the signing ceremony, it recommended to implement a process in line with the ceremony described in the Appendix.


# CP-PKI Operations {#operations-cp-pki}

This section details the procedures for deploying the CP-PKI and securing control plane communications.

## Distribution of TRCs


### Base TRC

Base TRCs are trust anchors and thus axiomatically trusted. All ASes within an ISD MUST be pre-loaded with the currently valid base-version TRC of their own ISD. For all specifications regarding the creation and distribution of initial/base TRCs, see [](#trc-ceremony).


### TRC Update Discovery

All non-base TRCs of an ISD are updates of the ISD's base TRC(s). The TRC update chain consists of regular and sensitive TRC updates. The specifications and rules that apply to updating a TRC are described in [](#update).

Relying parties MUST have at least one valid TRC available. Relying parties MUST discover TRC updates within the grace period defined in the updated TRC, and SHOULD discover TRC updates in a matter of minutes to hours. Additionally, any entity sending information that is secured by the Control Plane PKI MUST be able to provide all the necessary trust material to verify said information.

SCION provides the following mechanisms for discovering TRC updates and fulfilling the above requirement:

- *Beaconing Process*<br>
The TRC version is announced in the beaconing process. Each AS MUST announce what it considers to be the latest TRC, and MUST include the hash value of the TRC contents to facilitate the discovery of discrepancies. Therefore, relying parties that are part of the beaconing process discover TRC updates passively, i.e. a core AS notices TRC updates for remote ISDs that are on the beaconing path. A non-core AS only notices TRC updates for the local ISD through the beaconing process. The creation of a new TRC SHOULD trigger the generation of new control plane messages, as the propagation of control plane messages will help other ASes rapidly discover the new TRC.

- *Path Lookup*<br>
In every path segment, all ASes MUST reference the latest TRC of their ISD. Therefore, when resolving paths, every relying party will notice TRC updates, even remote ones.<br>

- *Active Discovery*<br>
Any TRC can be obtained at any time from the sender of the information it secures; either in a specific version or in its latest available version. The necessary query and response is described in {{I-D.dekater-scion-controlplane}}, section "Distribution of Cryptographic Material".

**Note:** The first two mechanisms above only work when there is active communication between the relying party and the ISD in question.



## Signing and Verifying Control Plane Messages {#signing-verifying-cp-messages}

The main purpose of the Control Plane PKI is providing a mechanism to distribute and authenticate public keys that are used to verify control plane messages and information, e.g. each hop information in a path segment is signed by the respective AS. Consequently, all relying parties MUST be able to verify signatures with the help of the Control Plane PKI.

The following sections specify the requirements that apply to the signing and verification of control plane messages.

### Signing a Control Plane Message

An AS signs control plane messages with the private key that corresponds to the (valid) AS' certificate.

The AS MUST attach the following information as signature metadata to ensure that a relying party can identify which certificate to use to verify the signed message:

- ISD-AS number: The ISD-AS number of the signing entity. For specification details, see [](#isd-as-nr).
- Subject key identifier: The identifier of the public key to be used to verify the message. For specification details, see [](#subject-key-id-ext).

Additionally, the signer SHOULD include the following information:

- Serial and base number of the latest TRC: Including this information allows relying parties to discover TRC updates and trust resets. For specification details, see [](#id).
- Timestamp: For many messages, the time at which it was signed is useful information to ensure freshness.


### Verifying a Control Plane Message

To verify a received control plane message, the relying party first needs to identify the certificate needed to validate the corresponding signature on the message.

AS certificates are bundled together with the corresponding issuing CA certificate into certificate chains. For efficiency, SCION distributes these certificate chains separately from the signed messages.

A certificate chain is verified against the Control Plane root certificate, although the root certificate is bundled with the TRC and **not** in the chain. This makes it possible to extend the validity period of the root certificate and update the corresponding TRC without having to modify the certificate chain.

To verify a control plane message, the relying party MUST perform the following steps:

1. Build a collection of root certificates from the latest TRC of the relevant ISD (that is, the ISD referenced in the signature metadata of the message). If the grace period (see [](#grace)) introduced by the latest TRC is still on-going, the root certificates in the second-to-latest TRC MUST also be included. For a description on how to build the correct collection of certificates, see [](#trc-selection).
2. If the signature metadata of the message contains the serial and base number of the latest TRC, the relying party MUST check that they have this latest TRC. If not, the relying party MUST request the latest TRC.
3. After constructing the pool of root certificates, the relying party MUST select the certificate chain used to verify the message. The AS certificate included in this certificate chain MUST have the following properties:
   - The ISD-AS number in the subject of the AS certificate MUST match the ISD-AS number in the signature metadata. See also [](#isd-as-nr).
   - The subject key identifier of the AS certificate MUST match the subject key identifier in the signature metadata. See also [](#subject-key-id-ext).
   - The AS certificate MUST be valid at verification time, which will normally be the current time. In special cases, e.g. auditing, the time can be set to the past to check if the message was verifiable at the given time.
4. After selecting a certificate chain to verify the control plane messages, the relying party MUST verify the certificate chain by:
   - Executing the regular X.509 verification procedure. For details, see {{X.509}}.
   - Checking that
      - all subjects of the certificates in the chain carry the same ISD number (see also [](#isd-as-nr),
      - each certificate is of the correct type (see also [](#cert-specs)), and
      - the CA certificate validity period covers the AS certificate validity period.
5. If the verification of the certificate chain was successful, the relying party can now verify the control plane messages with the root certificates from the certificate chain.

If any cryptographic material is missing in the process, the relying party MUST query the originator of the message for the missing material. If it cannot be resolved, the verification process fails.

**Important:** An implication of the above procedure is that path segments SHOULD be verifiable at time of use. It is not enough to rely on path segments being verified on insert since TRC updates that change the root key can invalidate a certificate chain.


## Issuing Control Plane AS Certificates

The steps REQUIRED to issue a new AS certificate are the following:

1. The AS creates a new key pair and a certificate signing request (CSR) using that key pair.
2. The AS sends the certificate signing request to the relevant CA within the ISD.
3. The CA uses its CA key and the CSR to create the new AS certificate.
4. The CA sends the AS certificate back to the AS.

When an AS joins an ISD, it sends the first CSR out of band to one of the CAs as part of the formalities to join the ISD. Subsequent certificate renewals MAY be automated and can leverage the control plane communication infrastructure (see {{I-D.dekater-scion-controlplane}}, section "Renewal of Cryptographic Material").

# Deployment Considerations

## PKI Availability

The Control Plane PKI relies on short-lived certificates as an alternative to revocation, as described in [](#substitutes-to-revocation). AS certificates typically have a validity of days (see {{table-3}}), except for the first issued AS certificate. Should an AS not be able to renew certificates, it would be cut off from the network.

It is therefore recommended to deploy multiple, independent CAs within an ISD that can issue certificates to all member ASes and sustain the appropriate certificate renewal load.
ASes should then be able to quickly switch over to a backup CA to renew their certificates in time.

Furthermore, PKI operators need to ensure that the CAs maintain accurate time. Further considerations related to this aspect are discussed in {{I-D.dekater-scion-controlplane}}, section "Attacks on Time Sources".

## Operational Processes for ISD Governance

An ISD is governed by voting ASes. In existing deployments, voting ASes may produce a regulations document to facilitate operations. Such document typically describes:

  - governance structure
  - roles and responsibilities
  - admission criteria
  - processes
  - protection measures for keys (e.g. use of HSMs)
  - actions in case of compromise or regulations breach

This document describes a typical TRC signing ceremony in [](#initial-ceremony), but further processes are out-of-scope.

# Security Considerations

SCION fundamentally differs from a global monopolistic trust model as each ISD manages its own trust roots instead of a single global entity providing those roots. This structure gives each ISD autonomy in terms of key management and in terms of trust, and prevents the occurrence of a global kill switch affecting all ISDs at once. However, each ISD is still susceptible to compromises that could affect or halt other components (control plane and forwarding).

This section discussed implication of such trust architecture, covering *inter*-AS security considerations. All *intra*-AS trust- and security aspects are out of scope.


## Compromise of an ISD

In SCION there is no central authority that could "switch off" an ISD as each relies on its own independent trust roots. Each AS within an ISD is therefore dependent on its ISD's PKI for its functioning, although the following compromises are potentially possible:

- At TRC level: The private root keys of the root certificates contained in a TRC are used to sign issuing CA certificates. If one of these private root keys is compromised, the adversary could issue illegitimate issuing CA certificates which may be used in further attacks. To maliciously perform a TRC update, an attacker would need to compromise multiple voting keys, the number of which is dependent on the voting quorum set in the TRC. The higher the quorum, the more unlikely a malicious update will be.
- At CA level: The private keys of an ISD's issuing CA certificates are used to sign the AS certificates and all ASes within an ISD obtain certificates directly from the CAs. If one of the CA's keys is compromised, an adversary could issue illegitimate AS certificates which may be used to impersonate ASes in further attacks. A compromised or misbehaving CA could also refuse to issue certificates to legitimate ASes, cutting them off the network if no alternative redundant CA is available.
- At AS level: Each AS within an ISD signs control plane messages with their AS private key. If the keys of an AS are compromised by an adversary, this adversary can illegitimately sign control plane messages including Path Construction Beacons (PCBs). This means that the adversary can manipulate the PCBs and propagate them to neighboring ASes or register/store them as path segments.


### Recovery from Compromise
This section deals with possible recovery from the compromises discussed in the previous paragraph.
As described in [](#substitutes-to-revocation), there is no revocation in the Control Plane PKI.

- At TRC level: If any of the root keys or voting keys contained in the TRC are compromised, the TRC MUST be updated as described in [](#update). A trust reset is only required in the case the number of compromised keys at the same time is greater or equal than the TRC's quorum (see [](#quorum)), and an invalid update has been produced and distributed in the network.
- At CA level: If the private key related to an issuing CA certificate is compromised, the impacted CA AS MUST obtain a new CA certificate from the corresponding root AS. Issuing CA certificates are generally short lived to limit the impact of compromise. Alternatively, with a TRC update, a new root keys can also be forced, invalidating the compromised CA.
- At AS level: In the event of a key compromise of a (non-core) AS, the impacted AS needs to obtain a new certificate from its CA. This process will vary depending on internal issuance processes.


## Denial of Service Attacks

The Control Plane PKI lays the foundation for the authentication procedures in SCION by providing each AS within a specific ISD with a certified key pair. These keys enable the authentication of all control plane messages - every AS and endpoint can verify all control plane messages by following the certificate chain.

The relying party needs to be able to discover and obtain new or updated cryptographic material. For the control plane messages, this is simplified by the observation that the sender of a message (e.g. of a Path Construction Beacon during path exploration or a path segment during a path lookup) always has all the cryptographic material to verify it. Thus, the receiver can always immediately obtain all the cryptographic material from the message originator.

As the corresponding PKI messaging only occurs when the control plane is already communicating, these requests to obtain cryptographic material are not prone to additional denial of service attacks. We refer to the security considerations of {{I-D.dekater-scion-controlplane}} for a more detailed description of DoS vulnerabilities of control plane messages.

On the other hand, this does not apply for certificate renewal. Denial of Service on the CA infrastructure or on the communication links from the individual ASes to the CA could be used by an attacker to prevent victim ASes from renewing their certificates and halting the path discovery process. This risk can be mitigated in multiple ways:

- CAs only need to be accessible from ASes within the ISD, reducing the potential DoS attack surface
- relying on multiple CAs within an ISD
- creating policies and processes to renew certificates out-of-band

## TRC Distribution and Trust on First Use

Base TRCs act as an ISD root of trust (see [](#trust-relations)).

If an endpoint retrieves the initial TRC in-band (e.g. from a local control service or a resolution server) without prior validation, it effectively operates under a "Trust on First Use" (TOFU) assumption. Care should therefore be taken in trusting the TRC source.

Should an AS be provisioned with a malicious TRC, it would not be able to communicate to other ASes in the affected ISD, thereby limiting impact of a malicious TRC.


# IANA Considerations

This document has no IANA actions.

The ISD and SCION AS number are SCION-specific numbers. They are currently allocated by the SCION Association (see {{ISD-AS-assignments}}).


--- back

# Certificate Extensions in ASN.1 Syntax {#cert-asn1}

~~~~
SCION-CP-PKI-CERT-EXTENSIONS {
    iso(1) identified-organization(3) dod(6) internet(1) private(4)
    enterprise(1) scion(55324) module(0) id-scion-pki-cert-ext(101)
}

DEFINITIONS EXPLICIT TAGS ::=
BEGIN

-- Root SCION object identifier (IANA Private Enterprise Number 55324)
id-scion OBJECT IDENTIFIER ::= { 1 3 6 1 4 1 55324 }

-- SCION Control Plane PKI
id-cppki OBJECT IDENTIFIER ::= { id-scion 1 }

-- SCION Attributes
id-at OBJECT IDENTIFIER ::= { id-cppki 2 }

-- SCION ISD-AS Attribute
id-at-ia OBJECT IDENTIFIER ::= { id-at 1 }

-- SCION Key Purposes
id-scion-kp OBJECT IDENTIFIER ::= { id-cppki 3 }

-- Identifies a Sensitive voting certificate
id-kp-sensitive OBJECT IDENTIFIER ::= { id-scion-kp 1 }

-- Identifies a Regular voting certificate
id-kp-regular   OBJECT IDENTIFIER ::= { id-scion-kp 2 }

-- Identifies a Root certificate
id-kp-root      OBJECT IDENTIFIER ::= { id-scion-kp 3 }

END
~~~~

# TRC in ASN.1 Syntax {#trc-asn1}

~~~~
SCION-CP-PKI-TRC {
    iso(1) identified-organization(3) dod(6) internet(1) private(4)
    enterprise(1) scion(55324) module(0) trc(1)
}

DEFINITIONS EXPLICIT TAGS ::=
BEGIN

IMPORTS
    Certificate
        FROM PKIX1Explicit88 {
            iso(1) identified-organization(3) dod(6) internet(1)
            security(5) mechanisms(5) pkix(7) id-mod(0)
            id-pkix1-explicit(18)
        };

TRCValidity ::= SEQUENCE {
    notBefore          GeneralizedTime,
    notAfter           GeneralizedTime
}

TRCPayload ::= SEQUENCE {
    version            TRCFormatVersion,
    iD                 TRCID,
    validity           TRCValidity,
    gracePeriod        INTEGER,
    noTrustReset       BOOLEAN,
    votes              SEQUENCE SIZE (0..2047) OF INTEGER (0..4095),
    votingQuorum       INTEGER (1..2047),
    coreASes           SEQUENCE OF ASN,
    authoritativeASes  SEQUENCE OF ASN,
    description        UTF8String (SIZE (0..8192)),
    certificates       SEQUENCE SIZE (0..4095) OF Certificate
}

TRCFormatVersion ::= INTEGER { v1(0) }

TRCID ::= SEQUENCE {
    iSD                ISD,
    serialNumber       INTEGER (1..MAX),
    baseNumber         INTEGER (1..MAX)
}

ISD ::= INTEGER (1..65535)

ASN ::= PrintableString

END
~~~~


# Signing Ceremony Initial TRC {#initial-ceremony}

A signing ceremony is used to create the initial (first) Trust Root Configuration of an ISD. Each ISD may decide how to conduct this ceremony, but it is RECOMMENDED to establish a procedure similar to the one described below:


## Ceremony Participants

The signing ceremony SHOULD include the following participants:

- **Ceremony administrator** - an individual in charge of moderating the signing process, guiding the participants through the steps, and acting as an intermediary for sharing information. The ceremony administrator is typically appointed by the ISD Manager or by resolution of the voting ASes.

- **Voting AS representatives** - individuals representing each voting AS who are able to create voting signatures on the TRC. They are in possession of a device with the private keys of their respective certificates in the TRC.

- **Witness(es)** - individual(s) who have no active role in the signing ceremony but may stop the process and request more information if they feel its integrity may have been compromised. The witness(es) are typically appointed by resolution of the voting ASes.

**Note:** The ISD members must decide on the roles of the signing ceremony participants in advance of signing ceremony, and must have reached agreement about the Certificate Authority (CA) ASes (that will also issue the root certificates). It is assumed that all parties are trustworthy and issues encountered during the signing ceremony may be assumed to be caused by honest mistakes and not by malicious intent. Hash comparison checks are included to counter mistakes and so that every participant can ensure they are operating on the same data, and the private keys of each participant never leave their machine. The ceremony administrator does not have to be entrusted with private keys.

## Ceremony Preparations {#ceremonyprep}

The participants agree in advance on the physical location of the signing ceremony, the devices that will be used, and MUST decide the ISD policy as follows:

- ISD number - usually obtained from the SCION registry, see [](#id);
- The description of the TRC, see [](#description);
- Validity period of the TRC, see [](#validity-trc);
- Grace period of the TRC (except for base TRCs);
- Voting quorum for the TRC, see [](#quorum);
- AS numbers of the core ASes, see [](#core);
- AS numbers of the authoritative ASes, see [](#auth);
- The list of control plane root certificates.

Each representative of a voting AS MUST also create the following before the ceremony:

- A sensitive voting private key and a self-signed certificate containing the corresponding public key.
- A regular voting private key and a self-signed certificate containing the corresponding public key.

In addition, each Certificate Authority MUST create a control plane root private key and a self-signed certificate containing the corresponding public key. A representative of the Certificate Authority need not be present at the ceremony as they do not need to sign the TRC, but they MUST provide their root certificate to be shared at the ceremony. The validity period of the certificates generated in advance MUST cover the full TRC validity period.

The location should provide electricity and power sockets for each participant, and should provide a monitor or projector that allows the ceremony administrator to display proceedings.

The ceremony administrator and voting ASes MUST each bring to the signing ceremony a secure machine capable of signing and verifying TRCs and computing the SHA-512 digest of the files. For voting ASes, the machine requires access to their own sensitive and regular voting private keys.

The ceremony administrator MUST provide or be provided with a device to exchange data between the ceremony participants.

The signing ceremony SHOULD include a procedure to verify that all devices are secure.


## Ceremony Phases {#ceremonyprocess}

The number of voting ASes present at the signing ceremony must be equal to or larger than the specified voting quorum.

The signing process has four phases of data sharing, led by the ceremony administrator who provides instructions to the other participants:


### Certificate Exchange {#phase1}

All parties share the certificates that must be part of the TRC with the ceremony administrator. For the voting ASes, these are the sensitive and the regular voting certificates, and for the Certificate Authority these are the control plane root certificates.

Each representative copies the requested certificates from their machine onto a data exchange device provided by the ceremony administrator that is passed between all representatives, before being returned to the ceremony administrator. Representatives MUST NOT copy the corresponding private keys onto the data exchange device as this invalidates the security of the ceremony.

The ceremony administrator then checks that the validity period of each provided certificate covers the previously agreed upon TRC validity, that the signature algorithms are correct, and that the certificate type is valid (root, sensitive voting or regular voting certificate). If these parameters are correct, the ceremony administrator computes the SHA-512 hash value for each certificate, aggregates and bundles all the provided certificates, and finally calculates the SHA-512 hash value for the entire bundle. All hash values must be displayed to the participants.

The ceremony administrator MUST then share the bundle with the representatives of the voting ASes who MUST validate on their machine that the hash value of their certificates and that of the bundled certificates is the same as displayed by the ceremony administrator.

This phase concludes when every representative has confirmed the SHA-512 sums are correct. If there is any mismatch then this phase MUST be repeated.


### Generation of the TRC Payload {#phase2}

The ceremony administrator generates the TRC payload based on the bundled certificates and the [](#trcfields) completed in accordance with ISD policy, see [](#ceremonyprep).

For each bundled certificate, the voting representatives MUST then verify the certificate type and that the following fields contain the correct information:

- `issuer`
- `subject`
- `validity`
- `signature`

Once the voting representatives have verified the TRC data, the ceremony administrator computes the DER encoding of the data according to [](#trc-asn1) and the SHA-512 hash value of the TRC payload file. The TRC payload file is then shared with the voting representatives via the data exchange device who verify the TRC payload hash value by computing this on their machine and checking it matches the one displayed by the ceremony administrator.

This phase concludes when all voting representatives confirm that the contents of the TRC payload are correct.


### TRC Signing {#phase3}

Each voting representative attaches a signature created with their own private voting key to the TRC (payload file), using their own machine. This serves to prove possession of the private keys.

This phase concludes when all voting representatives have attached their signatures to the TRC.


### TRC Validation {#phase4}

All voting representatives copy the TRC payload signed with their private voting keys to the data exchange device and return this to the ceremony administrator. The ceremony administrator assembles the final TRC by aggregating the payload data and verifying the signatures based on the certificates exchanged during phase [](#phase1). The ceremony administrator then shares the assembled TRC with all participants who MUST again inspect the signatures and verify them based on the certificates exchanged in phase [](#phase1).

The signing ceremony is completed once when every voting representative confirms that the signatures match. All participants can then use the TRC to distribute trust anchors for the ISD.


# Change Log
{:numbered="false"}

Changes made to drafts since ISE submission. This section is to be removed before publication.

## draft-dekater-scion-pki-13
{:numbered="false"}

- Draftforge review
- remove trust Hierarchy subsection and redundant code block
- Certificate validity recommendations: align to current practice
- `authorityKeyIdentifier` Extension: clarify support for `authorityCertIssuer` and `authorityCertSerialNumber` attributes

## draft-dekater-scion-pki-12
{:numbered="false"}

- Overall review and wording polish
- Introduction: shorten and refer to -controlplane
- Consistently use "Issuing CA certificate" / root certificate"
- Sections 2 and 3 (Certificate, TRC specification): reduce number of subheadings, reword TRC field descriptions. - Clarify that TRC validity uses GeneralizedTime
- Add ASN.1 modules in the appendix for Certificate extensions and TRCs
- Tables 3-7: sharpen normative language use

## draft-dekater-scion-pki-11
{:numbered="false"}

- Signing ceremony: minor updates to align with current process
- Signature field: clarify implications of using other algorithms or curves and mention mti set may be updated in future protocol iterations
- Clarify distinction between SCION ASes and BGP ASes through the text.
- Intro: remove duplicated motivation and component description and add a reference to the same text in -controlplane
- Clarify that initial AS certificates may have a longer validity to allow enough time for deployment
- "SCION-Specific Constraints and Conditions" section: drop requirement to use "UTF8String" for all fields, allow use of GeneralizedTime to align with RFC5280
- Security considerations: move and reword section "Dependency on Certificates" to new section "Deployment Considerations"
- Security considerations: new section on TRC Distribution
- Remove informative reference to I-D.dekater-panrg-scion-overview and to Anapaya's ISD assignments, since they are taken over by SCION Association in 2026. Remove unused references to RFC5398 and RFC6996.


## draft-dekater-scion-pki-10
{:numbered="false"}

- removed ISD assignment table and replaced to reference in Control Plane draft
- Updated number assignment reference
- Signatures: mention that other algorithms that ECDSA may be used in the future
- Figures: add SVG version

## draft-dekater-scion-pki-09
{:numbered="false"}

- Signing ceremony and introduction - improved text
- Clarified why a CA must have an ISD-AS number assigned
- Mention Active Discovery as a TRC discovery mechanism
- Abstract: mention goal and that document is not an Internet Standard

## draft-dekater-scion-pki-08
{:numbered="false"}

- Fix some oversized diagrams
- Introduction text rewording

## draft-dekater-scion-pki-07
{:numbered="false"}

Minor changes:

- Clarified relationship with RPKI.
- Added this changelog
- General text editing
- References: fixed ITU, ANSI, Assigned ISD-AS, fixed cross-reference to text formatting in the CP draft

## draft-dekater-scion-pki-06
{:numbered="false"}

Major changes:

- Added overview of SCION components to Introduction section.

Minor changes:

- General edits to make terminology consistent, remove duplication and rationalize text.
- Removed forward references.
- Added RFC2119 compliant terminology.


# Acknowledgments
{:numbered="false"}

Many thanks go to Fritz Steinmann (SIX Group AG), Juan A. Garcia Prado (ETH Zurich), Russ Housley (IETF), Brian Trammell (Google), Ramon Keller (LibC Technologies), Patrick Ambord (independent), Dominik Roos (Anapaya), and Kevin Meynell (SCION Association) for reviewing this document. We are also very grateful to Adrian Perrig (ETH Zurich), for providing guidance and feedback about each aspect of SCION. Finally, we are indebted to the SCION development teams of Anapaya and ETH Zurich, for their practical knowledge and for the documentation about the CP-PKI, as well as to the authors of {{CHUAT22}} - the book is an important source of input and inspiration for this draft.

