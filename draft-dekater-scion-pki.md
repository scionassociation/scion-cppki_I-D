---
title: "SCION Control-Plane PKI"
abbrev: "SCION CPPKI"
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
     org: ETH Zuerich
     email: corine.dekatermuehlhaeuser@inf.ethz.ch

 -   ins: N. Rustignoli
     name: Nicola Rustignoli
     org: ETH Zuerich
     email: nicola.rustignoli@inf.ethz.ch

normative:

informative:
  RFC5280:
  RFC5480:
  RFC5758:
  RFC8410:
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
  I-D.scion-overview:
    title: SCION Overview
    date: 2022
    target: https://datatracker.ietf.org/doc/draft-dekater-panrg-scion-overview/
    author:
      -
        ins: C. de Kater
        name: Corine de Kater
        org: ETH Zuerich
      -
        ins: N. Rustignoli
        name: Nicola Rustignoli
        org: ETH Zuerich
      -
        ins: A. Perrig
        name: Adrian Perrig
        org: ETH Zuerich
  I-D.scion-components:
    title: SCION Components Analysis
    date: 2022
    target: https://datatracker.ietf.org/doc/draft-rustignoli-panrg-scion-components/
    author:
      -
        ins: N. Rustignoli
        name: Nicola Rustignoli
        org: ETH Zuerich
      -
        ins: C. de Kater
        name: Corine de Kater
        org: ETH Zuerich

--- abstract

This document presents the trust concept and design of the SCION *control-plane PKI*, SCION's public key infrastructure model. SCION (Scalability, Control, and Isolation On Next-generation networks) is a path-aware, inter-domain network architecture. The control-plane PKI, or short CP-PKI, handles cryptographic material and lays the foundation for the authentication procedures in SCION. It is used by SCION's control plane to authenticate and verify path information, and builds the basis for SCION's special trust model based on so-called Isolation Domains.

The document first describes the trust model behind SCION's control-plane PKI, and provides a short overview of the certificates, keys, and roles involved. It then continues with detailed specifications of the building blocks of SCION's control-plane PKI. The document concludes with several considerations in regard to deploying the control-plane PKI.


--- middle

# Introduction

The control-plane PKI (CP-PKI) lays the foundation for the authentication procedures in SCION. It handles all cryptographic material used in the public key infrastructure of SCION's control plane. This section first introduces the key concepts of the SCION CP-PKI, including the trust model, its core elements (certificates, keys, and roles), and their relationships. The sections after the Introduction provide detailed specifications of the building blocks of the CP-PKI.

<br>
**Note:** For more detailed information on the SCION next-generation inter-domain architecture, see {{CHUAT22}}, especially Chapter 2, as well as the IETF Internet Drafts {{I-D.scion-overview}} and {{I-D.scion-components}}.
<br>

## Trust Model

Given the diverse nature of the constituents in the current Internet, an important challenge is how to scale authentication of network elements (such as AS ownership, hop-by-hop routing information, name servers for DNS, and domains for TLS) to the global environment. The roots of trust of currently prevalent public key infrastructure (PKI) models do not scale well to a global environment, because (1) mutually distrustful parties cannot agree on a single trust root (monopoly model), and because (2) the security of a plethora of roots of trust is only as strong as its weakest link (oligopoly model) - see also {{BARRERA17}}.

The monopoly model suffers from two main drawbacks: First, all parties must agree on a single root of trust. Secondly, the single root of trust represents a single point of failure, the misuse of which enables the forging of certificates. Also, its revocation can result in a kill-switch for all the entities it certifies. The oligopoly model relies on several roots of trust, all equally and completely trusted. However, this is not automatically better: Whereas the monopoly model has a single point of failure, the oligopoly model has the drawback of exposing more than one point of failure.

Thus, there is a need for a trust architecture that supports meaningful trust roots in a global environment with inherently distrustful parties. This new trust architecture should provide the following properties:

- Trust agility (see further below);
- Resilience to single root of trust compromise;
- Multilateral governance; and
- Support for policy versioning and updates.

Ideally, the trust architecture allows parties that mutually trust each other to form their own trust "union" or "domain", and to freely decide whether to trust other trust unions (domains) outside their own trust bubble.

To fulfill the above requirements, which in fact apply well to inter-domain networking, SCION introduces the concept of **Isolation Domains**. An Isolation Domain (ISD) is a building block for achieving high availability, scalability, and support for heterogeneous trust. It consists of a logical grouping of ASes that share a uniform trust environment (i.e., a common jurisdiction). An ISD is administered by multiple ASes that form the ISD core; these are the **core ASes**. It is governed by a policy called the **Trust Root Configuration** (TRC), which is negotiated by the ISD core. The TRC defines the locally scoped roots of trust used to validate bindings between names and public keys.

Authentication in SCION is based on digital certificates that bind identifiers to public keys and carry digital signatures that are verified by roots of trust. SCION allows each ISD to define its own set of trust roots, along with the policy governing their use. Such scoping of trust roots within an ISD improves security, as compromise of a private key associated with a trust root cannot be used to forge a certificate outside the ISD. An ISD's trust roots and policy are encoded in the TRC, which has a version number, a list of public keys that serves as root of trust for various purposes, and policies governing the number of signatures required for performing different types of actions. The TRC serves as a way to bootstrap all authentication within SCION. Additionally, TRC versioning is used to efficiently revoke compromised roots of trust.

The TRC also provides *trust agility*, that is, it enables users to select the trust roots used to initiate certificate validation. This implies that users are free to choose an ISD they believe maintains a non-compromised set of trust roots. ISD members can also decide whether to trust other ISDs and thus transparently define trust relationships between parts of the network. SCION trust model, therefore, differs from the one provided by other PKI architectures.


## Trust Relations within an Isolation Domain

As already mentioned previously, the control-plane PKI, SCION's concept of trust, is organized on ISD-level. Each ISD can independently specify its own Trust Root Configuration (TRC) and build its own verification chain. Each TRC consists of a collection of signed root certificates, which are used to sign CA certificates, which are in turn used to sign AS certificates. The TRC also includes ISD-policies that specify, for example, the TRC's usage, validity, and future updates. A TRC is a fundamental component of an ISD's control-plane PKI. The so-called **base TRC** constitutes the ISD's trust anchor. It is signed during a signing ceremony and then distributed throughout the ISD.


### Updates and Trust Resets {#trust-reset}

There are two types of TRC updates: regular and sensitive. A **regular TRC update** is a periodic re-issuance of the TRC where the entities and policies listed in the TRC remain unchanged, whereas a **sensitive TRC update** is an update that modifies critical aspects of the TRC, such as the set of core ASes. In both cases, the base TRC remains unchanged. If the ISD's TRC has been compromised, it is necessary for an ISD to re-establish the trust root. This is possible with a process called **trust reset** (if allowed by the ISD's trust policy). In this case, a new base TRC is created.


## Overview of Certificates, Keys, and Roles

The base TRC constitutes the root of trust within an ISD. The next figure provides a first impression of the trust chain within an ISD, based on its TRC. For detailed descriptions, please refer to the chapters [Certificate Specification](#cert-specs) and [Specification of the Trust Root Configuration](#trc-specification).

>>>>>> **Figure 1** - *Chain of trust within an ISD*

All certificates used in SCION's control-plane PKI are in X.509 v3 format {{RFC5280}}. Additionally, the TRC contains self-signed certificates instead of plain public keys. Self-signed certificates have the following advantages over plain public keys: (1) They make the binding between name and public key explicit; and (2) the binding is signed to prove possession of the corresponding private key.

All ASes in SCION have the task to sign and verify control-plane messages. However, certain ASes have additional roles:

- **Core ASes**: Core ASes are a distinct set of ASes in the SCION control plane. For each ISD, the core ASes are listed in the TRC. Each core AS in an ISD has links to other core ASes (in the same or in different ISDs).
- **Certification authorities (CAs)**: CAs are responsible for issuing AS certificates to other ASes and/or themselves.
- **Voting ASes**: Only certain ASes within an ISD may sign TRC updates. The process of appending a signature to a new TRC is called "casting a vote"; the designated ASes that hold the private keys to sign a TRC update are "voting ASes".

All further details of the SCION control-plane PKI are specified in the following sections.


## Trust Concept as a Function

The SCION control-plane PKI can be seen as a function that transforms potential distrust among different parties into a mutually accepted trust contract including a trust update and reset policy as well as certificates used for authentication procedures in SCION's control plane.

For the function to work, it is not necessary that the ASes of the ISD all trust each other. However, all ASes MUST trust the ISD's core ASes, authoritative ASes, voting ASes, as well as its CA(s). These trusted parties negotiate the ISD trust contract in a "bootstrapping of trust" ceremony, where cryptographic material is exchanged, and the ISD's trust anchor, the initial Trust Root Configuration, is created and signed.

### Input

Prior to the ceremony, the trusted parties must decide about the validity period of the TRC as well as the number of votes required to update a TRC. They must also bring the required keys and certificates, the so-called root and voting keys/certificates. During the ceremony, the trusted parties decide about the number of the ISD, which must be an integer in the inclusive range between 1 and 65535.

### Output

The output of the bootstrapping of trust ceremony, or the trust "function", are the ISD's initial Trust Root Configuration as well as mutually trusted and accepted CA and AS certificates--the latter are used to verify SCION's control-plane messages. Together with the ISD's control-plane root certificates, the CA and AS certificates build the ISD's trust and verification chain.


## Conventions and Definitions

{::boilerplate bcp14-tagged}


# Certificate Specification {#cert-specs}

This section provides a detailed specification of all certificates used in SCION's control-plane PKI. It starts with an overview of the main keys and certificates.

## SCION Control-Plane PKI Keys and Certificates - Overview {#overview}

All certificates in SCION's control-plane PKI are in X.509 v3 format {{RFC5280}}. Each certificate has a `subject` (the entity that owns the certificate) and an `issuer` (the entity that signed the certificate, usually a CA). In the case of self-signed certificates, the subject and the issuer are the same entity.

There are three types of control-plane (CP) certificates: root certificates, CA certificates, and AS certificates. Together, they build a chain of trust that is anchored in the trust root configuration (TRC) file of the respective Isolation Domain (ISD). Additionally, there are regular and sensitive voting certificates, which define the keys to cast votes in a regular and a sensitive TRC update, respectively.

The following list summarizes the main certificates and corresponding key pairs of SCION's control-plane PKI as well as the voting certificates and keys:

- **Control-Plane Root Certificates** - Control-plane (CP) root certificates are used to verify control-plane CA certificates. Control-plane root certificates are embedded in TRCs, to facilitate the bootstrapping of trust.
   - *CP root private key*: This private key is used to sign control-plane CA certificates.
   - *CP root certificate*: This is the container for the public key associated with the CP root private key.
   - Section [Control-Plane Root Certificate](#cp-root-cert) provides more details on the CP root certificates.
- **Control-Plane CA Certificates** - Control-plane (CP) CA certificates are used to verify AS certificates.
   - *CP CA private key*: This private key is used by the CA to sign AS certificates.
   - *CP CA certificate*: This is the container for the public key associated with the CP CA private key.
   - Section [Control-Plane CA Certificate](#cp-ca-cert) provides more details on the CP CA certificates.
- **Control-Plane AS Certificates** - Control-plane (CP) AS certificates are used to verify control-plane messages such as path-segment construction beacons (PCB). PCBs explore network paths within an ISD.
   - *CP AS private key*: This private key is used by an AS to sign control-plane messages.
   - *CP AS certificate*: This is the container for the public key associated with the CP AS private key.
   - Section [Control-Plane AS Certificate](#cp-as-cert) provides more details on the CP AS certificates.

<br>
**Note:** The TRC of each ISD contains a trusted set of control-plane root certificates. This set builds the root of each ISD's verification path. For more information on the selection of this trusted set of root certificates, see [Specification of the Trust Root Configuration](#trc-specification).
<br>

- **Voting Certificates** - Regular and sensitive voting certificates are used to verify regular and sensitive TRC updates, respectively.
   - *Regular voting private key*: This private key is used to sign regular TRC updates. The corresponding public key is embedded in TRCs (via the regular voting certificate).
   - *Regular voting certificate*: This is the container for the public key associated with the regular voting private key.
   - *Sensitive voting private key*: This private key is used to sign sensitive TRC updates. The corresponding public key is embedded in TRCs (via the sensitive voting certificate).
   - *Sensitive voting certificate*: This is the container for the public key associated with the sensitive voting private key.
   - Section [Voting Certificates](#cp-voting-cert) provides more details on the voting certificates.

{{table-1}} and {{table-2}} below provide a formal overview of the different types of key pairs and certificates in the control-plane PKI.


| Name                 | Notation (1)   	| Used to verify/sign   	|
|----------------------+------------------+-------------------------|
| Sensitive voting key | K<sub>sens</sub> | TRC updates (sensitive) |
| Regular voting key   | K<sub>reg</sub>  | TRC updates (regular)   |
| CP root key          | K<sub>root</sub> | CP CA certificates      |
| CP CA key            | K<sub>CA</sub>   | CP AS certificates      |
| CP AS key            | K<sub>AS</sub>   | PCBs, path segments     |
{: #table-1 title="Key chain"}

(1) K<sub>x</sub> = PK<sub>x</sub> + SK<sub>x</sub>, where x = certificate type, PK<sub>x</sub> = public key, and SK<sub>x</sub> = private key

| Name                   | Notation       	 | Signed with           	                  | Contains                                                | Validity (2) |
|------------------------+-------------------+------------------------------------------+---------------------------------------------------------+--------------|
| TRC (trust root conf)  | TRC               | SK<sub>sens</sub>, SK<sub>reg</sub> (1)  | C<sub>root</sub>, C<sub>sens</sub>, C<sub>reg</sub> (1) | 1 year       |
| Sensitive voting cert. | C<sub>sens</sub>  | SK<sub>sens</sub>                        | PK<sub>sens</sub>                                       | 5 years      |
| Regular voting cert.   | C<sub>reg</sub>   | SK<sub>reg</sub>                         | PK<sub>reg</sub>                                        | 1 year       |
| CP root certificate    | C<sub>root</sub>  | SK<sub>root</sub>                        | PK<sub>root</sub>                                       | 1 year       |
| CP CA certificate      | C<sub>CA</sub>    | SK<sub>root</sub>                        | PK<sub>CA</sub>                                         | 11 days (3)  |
| CP AS certificate      | C<sub>AS</sub>    | SK<sub>CA</sub>                          | PK<sub>AS</sub>                                         | 3 days       |
{: #table-2 title="Certificates"}

(1) Multiple signatures and certificates of each type may be included in a TRC.<br>
(2) Recommended maximum validity period.<br>
(3) A validity of 11 days with 4 days overlap between two CA certificates is recommended to enable best possible operational procedures when performing a CA certificate rollover.

Figure 2 illustrates, at a high level, the relationship between a TRC and the five types of certificates.

>>>>>> **Figure 2** - *TRC update chain and the different types of associated certificates. Arrows show how signatures are verified; in other words, they indicate that a public key contained in a certificate or TRC can be used to verify the authenticity of another item.*


## Certificate Specification

This section provides an in-depth specification of the SCION certificates. The SCION certificate specification builds on top of {{RFC5280}}, which in turn builds on top of [X.509](https://handle.itu.int/11.1002/1000/13031). However, the SCION specification is more restrictive.

This section defines the additional constraints compared to {{RFC5280}} for each type of SCION control-plane certificate. The recommended settings for optional constraints are based on the SCION open source implementation [scionproto](https://github.com/scionproto/scion/). Adjusting the optional constraints to the requirements of a customer implementation is possible and allowed.

### General Certificate Requirements {#general-cert-req}

SCION control-plane certificates are X.509 v3 certificates. Every certificate has a `subject`, which is the entity that owns the certificate, and an `issuer`, which is the entity that issued the certificate, usually a CA.

The next code block shows the generic format of SCION control-plane certificates. It is followed by a description of the SCION specifics for each certificate field.

<br>
**Note:** For information regarding the full format, see [X.509](https://handle.itu.int/11.1002/1000/13031), clause 7.2.
<br>

~~~~
   TBSCertificate ::= SEQUENCE {
       version               [0]   EXPLICIT Version DEFAULT v1,
       serialNumber                CertificateSerialNumber,
       signature                   AlgorithmIdentifier{{SupportedAlgorithms}},
       issuer                      Name,
       validity                    Validity,
       subject                     Name,
       subjectPublicKeyInfo        SubjectPublicKeyInfo,
       issuerUniqueID        [1]   IMPLICIT UniqueIdentifier OPTIONAL, -- disallowed in SCION
       subjectUniqueID       [2]   IMPLICIT UniqueIdentifier OPTIONAL, -- disallowed in SCION
       extensions            [3]   EXPLICIT Extensions OPTIONAL
   }

   Version ::= INTEGER { v1(0), v2(1), v3(2)}  -- v1, v2 are disallowed in SCION
   CertificateSerialNumber ::= INTEGER

   Validity ::= SEQUENCE {
       notBefore Time,
       notAfter Time
   }

   Time ::= CHOICE {
       utcTime UTCTime,
       generalizedTime GeneralizedTime
   }

   SubjectPublicKeyInfo ::= SEQUENCE {
       algorithm         AlgorithmIdentifier{{SupportedAlgorithms}},
       subjectPublicKey  BIT STRING
   }

   Extensions ::= SEQUENCE SIZE (1..MAX) OF Extension

   Extension ::= SEQUENCE {
       extnId      OBJECT IDENTIFIER,
       critical    BOOLEAN DEFAULT FALSE,
       extnValue   OCTET STRING
                       -- contains DER encoding of ASN.1 value
                       -- corresponding to type identified by extnID
   }
~~~~


#### `version` Field

The `version` field MUST be set to `v3` in SCION, as extensions are mandatory.


#### `serialNumber` Field

The `serialNumber` field is used as specified in {{RFC5280}}.


#### `signature` Field {#certificate-signature}

For security reasons, SCION uses a custom list of acceptable signature algorithms. This list of acceptable signature algorithms is specified in the `signature` field.

The list currently only contains the ECDSA signature algorithm (defined in [X.962](https://standards.globalspec.com/std/1955141/ansi-x9-62)) - see the code block below. However, the list might be extended in the future.

The Object Identifiers (OIDs) for ECDSA are defined as `ecdsa-with-SHA256`, `ecdsa-with-SHA384`, and `ecdsa-with-SHA512` in {{RFC5758}}. They are included as follows:

~~~~
   sigAlg-ecdsa-SHA256      ALGORITHM         ::= { OID ecdsa-with-SHA256 }
   sigAlg-ecdsa-SHA384      ALGORITHM         ::= { OID ecdsa-with-SHA384 }
   sigAlg-ecdsa-SHA512      ALGORITHM         ::= { OID ecdsa-with-SHA512 }

   ecdsa-with-SHA256 OBJECT IDENTIFIER ::= { iso(1) member-body(2)
       us(840) ansi-X9-62(10045) signatures(4) ecdsa-with-SHA2(3) 2 }
   ecdsa-with-SHA384 OBJECT IDENTIFIER ::= { iso(1) member-body(2)
       us(840) ansi-X9-62(10045) signatures(4) ecdsa-with-SHA2(3) 3 }
   ecdsa-with-SHA512 OBJECT IDENTIFIER ::= { iso(1) member-body(2)
       us(840) ansi-X9-62(10045) signatures(4) ecdsa-with-SHA2(3) 4 }
~~~~

<br>
**Important:** The accepted cryptographic algorithms listed in this document are the only currently accepted cryptographic algorithms. SCION implementations MUST reject cryptographic algorithms not found in the list.
<br>

The only accepted curves for ECDSA are:

- NIST P-256 ([NISTFIPS186-4](http://dx.doi.org/10.6028/NIST.FIPS.186-4), section D.1.2.3) (named `secp256r1` in {{RFC5480}})
- NIST P-384 ([NISTFIPS186-4](http://dx.doi.org/10.6028/NIST.FIPS.186-4), section D.1.2.4) (named `secp384r1` in {{RFC5480}})
- NIST P-521 ([NISTFIPS186-4](http://dx.doi.org/10.6028/NIST.FIPS.186-4), section D.1.2.5) (named `secp521r1` in {{RFC5480}})

The OIDs for the above curves are:

~~~~
   secp256r1 OBJECT IDENTIFIER ::= {
       iso(1) member-body(2) us(840) ansi-X9-62(10045) curves(3)
       prime(1) 7 }
   secp384r1 OBJECT IDENTIFIER ::= {
       iso(1) identified-organization(3) certicom(132) curve(0) 34 }
   secp521r1 OBJECT IDENTIFIER ::= {
       iso(1) identified-organization(3) certicom(132) curve(0) 35 }
~~~~

The appropriate hash size to use when producing a signature with an ECDSA key is:

- ECDSA with SHA-256, for a P-256 signing key
- ECDSA with SHA-384, for a P-384 signing key
- ECDSA with SHA-512, for a P-521 signing key

<br>
**Important:** SCION implementations MUST include support for P-256, P-384, and P-521.
<br>

##### `AlgorithmIdentifier` Sequence

[X.509](https://handle.itu.int/11.1002/1000/13031) defines the syntax of the `AlgorithmIdentifier` as follows:

~~~~
   AlgorithmIdentifier  ::=  SEQUENCE  {
       algorithm   OBJECT IDENTIFIER,
       parameters  ANY DEFINED BY algorithm OPTIONAL
   }
~~~~

<br>
**Note:** In SCION implementations, the `parameters` field MUST be absent, as defined in {{RFC8410}}.
<br>

In general, if the `AlgorithmIdentifier` in a specific SCION implementation is not defined as described above, the implementation should stop the validation process entirely and error out.


#### `issuer` Field {#issuer}

The `issuer` field contains the distinguished name (DN) of the CA that created the certificate. The `issuer` field MUST be non-empty.

[X.501](http://handle.itu.int/11.1002/1000/13030) (10/2016), clause 9.2, defines the syntax for `Name` as follows:

~~~~
   Name ::= CHOICE {
       rdnSequence RDNSequence
   }

   RDNSequence ::= SEQUENCE OF RelativeDistinguishedName

   RelativeDistinguishedName ::= SET SIZE (1..MAX) OF AttributeTypeAndValue

   AttributeType ::= OBJECT IDENTIFIER

   AttributeValue ::= ANY -- DEFINED BY AttributeType
~~~~

In most SCION implementations, the type (`AttributeType`) will be a `DirectoryString` type, outlined as follows:

~~~~
   DirectoryString ::= CHOICE {
       teletexStrings TeletexString (SIZE (1..MAX)),
       printableString PrintableString (SIZE (1..MAX)),
       universalString UniversalString (SIZE (1..MAX)),
       utf8String UTF8String (SIZE (1..MAX)),
       bmpString BMPString (SIZE (1..MAX)),
   }
~~~~

All SCION implementations MUST support the following standard attribute types:

- `country`
- `organization`
- `organizational unit`
- `distinguished name qualifier`
- `state or province name`
- `common name`
- `serial number`
- `ISD-AS number`
<br>

Except for the `ISD-AS number` attribute, all the above attributes are defined in {{RFC5280}}.

As an additional constraint compared to {{RFC5280}}, SCION implementations MUST use the `UTF8String` value type for all the above attributes, including the `ISD-AS number` attribute.

<br>
**Note:** Besides the above listed required attributes, SCION implementations may additionally also support other attributes.
<br>


##### `ISD-AS number` Attribute {#isd-as-nr}

The `ISD-AS number` attribute identifies the SCION ISD and AS. In the SCION open source implementation, the attribute type is `id-at-ia`, defined as:

~~~~
   id-at-ia AttributeType ::= {id-ana id-cppki(1) id-at(2) 1}
~~~~

where `id-ana` specifies the root SCION object identifier (OID).

<br>
**Note:** The SCION open source implementation currently uses the Anapaya IANA Private Enterprise Number (55324) as root SCION object identifier (OID):<br>
`id-ana ::= OBJECT IDENTIFIER {1 3 6 1 4 1 55324}`
<br>

The following points apply when setting the attribute value of the `ISD-AS number` attribute:

- The string representation MUST follow the canonical formatting defined in [ISD and AS numbering](https://github.com/scionproto/scion/wiki/ISD-and-AS-numbering).
- The canonical string representation uses a dash separator between the ISD and AS numbers.
- The ISD numbers are formatted as decimal.
- The canonical string formatting of AS numbers in the BGP AS range (0, 2<sup>32-1</sup>) is the decimal form. Larger AS numbers, i.e., from 2<sup>32</sup> to 2<sup>48-1</sup>, use a 16-bit, colon-separated, lower-case, hex encoding with leading zeros omitted: `1:0:0` to `ffff:ffff:ffff`.

<br>
**Example:**<br>
AS `ff00:0:110` in ISD `1` is formatted as `1-ff00:0:110`.
<br>

The `ISD-AS number` attribute MUST be present exactly once in all SCION control-plane certificates. Implementations MUST NOT create nor successfully verify certificates that do not include the ISD-AS number, or include it more than once.


#### `validity` Field {#cert-validity}

Section 4.1.2.5 of {{RFC5280}} defines the `validity` field. In addition to this definition, the following constraints apply to SCION control-plane certificates:

- All certificates MUST have a well-defined expiration date. SCION control-plane certificates that use the *99991231235959Z* generalized time value, instead of a well-defined expiration date, are not valid. SCION implementations MUST NOT create such certificates, and verifiers MUST error out when encountering such a certificate.
- The validity period of a certificate is the period of time in between the values of the `notBefore` and `notAfter` attributes. For each control-plane certificate type, this validity period must have a specific maximum value. For more information, see the below sections describing the control-plane and voting certificates: [Control-Plane Root Certificate](#cp-root-cert), [Control-Plane CA Certificate](#cp-ca-cert), [Control-Plane AS Certificate](#cp-as-cert), and [Voting Certificates](#cp-voting-cert).


#### `subject` Field {#subject}

The `subject` field defines the entity that owns the certificate. It is specified in the same way as the `issuer` field (see [`issuer` Field](#issuer)). All SCION control-plane certificates MUST have the `subject` field defined (with the same requirements as those for the `issuer` field).


#### `subjectPublicKeyInfo` Field

The `subjectPublicKeyInfo` field carries the public key of the subject (the entity that owns the certificate). It identifies which algorithm to use with the key.

The SCION constraints in [`signature` Field](#certificate-signature) also apply here: The key must be a valid key for the selected curve, and the algorithm used must be included in the list of acceptable signature algorithms. The list currently only contains the ECDSA signature algorithm (defined in [X.962](https://standards.globalspec.com/std/1955141/ansi-x9-62)).


#### `issuerUniqueID` and `subjectUniqueID` Fields

The fields `issuerUniqueID` and `subjectUniqueID` are disallowed and thus MUST NOT be used in a SCION implementation.


#### Extensions {#exts}

[X.509](https://handle.itu.int/11.1002/1000/13031), clause 7.2, defines the syntax of the `Extensions` sequence.
This section describes the extensions relevant for SCION.


##### `authorityKeyIdentifier` Extension

The `authorityKeyIdentifier` extension is defined in clause 9.2.2.1 of [X.509](https://handle.itu.int/11.1002/1000/13031).

The `authorityKeyIdentifier` extension identifies the private key used to sign the certificate. It is defined as follows:

~~~~
   authorityKeyIdentifier EXTENSION ::= {
       SYNTAX AuthorityKeyIdentifier
       IDENTIFIED BY id-ce-authorityKeyIdentifier
   }

   AuthorityKeyIdentifier ::= SEQUENCE {
       keyIdentifier             [0]   KeyIdentifier OPTIONAL,
       authorityCertIssuer       [1]   GeneralNames OPTIONAL,
       authorityCertSerialNumber [2]   CertificateSerialNumber OPTIONAL,
       ...
   }
   (WITH COMPONENTS {..., authorityCertIssuer PRESENT,
                          authorityCertSerialNumber PRESENT } |
   WITH COMPONENTS {..., authorityCertIssuer ABSENT,
                         authorityCertSerialNumber ABSENT } )

   KeyIdentifier ::= OCTET STRING
~~~~

Using the `keyIdentifier` attribute is the preferred way to specify the `authorityKeyIdentifier` extension.

<br>
**Important:** SCION implementations may also support the use of the `authorityCertIssuer` and `authorityCertSerialNumber` attributes. However, if these attributes are set and support for them is missing, implementations should error out.
<br>

This extension MUST always be non-critical (which is the default, see the code block displaying the generic format of SCION control-plane certificates in [General Certificate Requirements](#general-cert-req)). However, SCION implementations MUST error out if the extension is not present AND the certificate is not self-signed.


##### `subjectKeyIdentifier` Extension {#subject-key-id-ext}

The `subjectKeyIdentifier` extension is defined in clause 9.2.2.2 of [X.509](https://handle.itu.int/11.1002/1000/13031), (10/2016).

The `subjectKeyIdentifier` extension identifies the public key being certified. It can be used, for example, by control-plane messages to identify which certificate to use for verification. The extension allows for overlapping control-plane CA keys, for example during updates. It is defined as follows:

~~~~
   subjectKeyIdentifier EXTENSION ::= {
       SYNTAX SubjectKeyIdentifier
       IDENTIFIED BY id-ce-subjectKeyIdentifier
   }

   SubjectKeyIdentifier ::= KeyIdentifier
~~~~

This extension MUST always be non-critical (which is the default, see the code block displaying the generic format of SCION control-plane certificates in [General Certificate Requirements](#general-cert-req)). However, SCION implementations must error out if the extension is not present.

##### `keyUsage` Extension

The `keyUsage` extension is defined in clause 9.2.2.3 of [X.509](https://handle.itu.int/11.1002/1000/13031), (10/2016).

The `keyUsage` extension identifies the intended usage of the public key in the corresponding certificate. The ASN.1 definition is as follows:

~~~~
   keyUsage EXTENSION ::= {
       SYNTAX KeyUsage
       IDENTIFIED BY id-ce-keyUsage
   }

   KeyUsage ::= BIT STRING {
       digitalSignature  (0),
       contentCommitment (1),
       keyEncipherment   (2),
       dataEncipherment  (3),
       keyAgreement      (4),
       keyCertSign       (5),
       cRLSign           (6),
       encipherOnly      (7),
       decipherOnly      (8),
   }
~~~~

The attributes of the `keyUsage` extension define the various possible ways of using the public key. The attributes have the following meaning in SCION:

- `digitalSignature`: The public key can be used to verify the digital signature of a control-plane payload.
- `contentCommitment`: Not used.
- `keyEncipherment`: Not used.
- `dataEncipherment`: Not used.
- `keyAgreement`: Not used.
- `keyCertSign`: The public key can be used to verify the CA signature on a control-plane certificate.
- `cRLSign`: Not used.
- `encipherOnly`: Not used.
- `decipherOnly`: Not used.

<br>
**Important:** If the certificate's public key is used to verify the signature of a control-plane payload (`digitalSignature` attribute), it must be possible to trace back the private key used for the signature. This is done by referencing the ISD-AS and the subject key identifier (via the `subjectKeyIdentifier` extension). For more information about the `subjectKeyIdentifier` extension, see [`subjectKeyIdentifier` Extension](#subject-key-id-ext).
<br>

Each control-plane certificate type uses the public key differently, and consequently also specifies the attributes of the `keyUsage` extension differently. For more information, see the below sections describing the control-plane and voting certificates: [Control-Plane Root Certificate](#cp-root-cert), [Control-Plane CA Certificate](#cp-ca-cert), [Control-Plane AS Certificate](#cp-as-cert), and [Voting Certificates](#cp-voting-cert).

If present, the `keyUsage` extension should be marked as "critical". That is, the `critical` Boolean attribute of this extension must be set to TRUE (the default is FALSE, see the code block displaying the generic format of SCION control-plane certificates in [General Certificate Requirements](#general-cert-req)).

<br>
**Note:** If a certificate extension is marked "critical", the public key in the certificate should only be used for the purpose set in the critical extension.
<br>


##### `extKeyUsage` Extension

The `extKeyUsage` extension is defined in clause 9.2.2.4 of [X.509](https://handle.itu.int/11.1002/1000/13031).

The `extKeyUsage` extension specifies additional usages of the public key in the certificate. It is defined as follows:

~~~~
   extKeyUsage EXTENSION ::= {
       SYNTAX             SEQUENCE SIZE (1..MAX) OF KeyPurposeId
       IDENTIFIED BY      id-ce-extKeyUsage
   }

   KeyPurposeId ::= OBJECT IDENTIFIER
~~~~

SCION uses the following attributes of the Extended Key Usage extension, as defined in Section 4.2.1.12 of {{RFC5280}}:

- `id-kp-serverAuth`: If set, the public key can be used for SCION control-plane server authentication.
- `id-kp-clientAuth`: If set, the public key can be used for SCION control-plane client authentication.
- `id-kp-timeStamping`: If set, the public key can be used for the verification of timestamps.

This extension MUST be present in control-plane root-, AS- and voting certificates. It MAY be present in control-plane CA certificates. For the exact settings per certificate type, see the below sections describing the control-plane and voting certificates: [Control-Plane Root Certificate](#cp-root-cert), [Control-Plane CA Certificate](#cp-ca-cert), [Control-Plane AS Certificate](#cp-as-cert), and [Voting Certificates](#cp-voting-cert).


##### `basicConstraints` Extension

The `basicConstraints` extension is defined in clause 9.4.2.1 of [X.509](https://handle.itu.int/11.1002/1000/13031).

The `basicConstraints` extension specifies whether the certificate subject may act as a CA. The ASN.1 definition for the `basicConstraints` extension is as follows:

~~~~
   basicConstraints EXTENSION ::= {
       SYNTAX          BasicConstraintsSyntax
       IDENTIFIED BY   id-ce-basicConstraints
   }

   BasicConstraintsSyntax ::= SEQUENCE {
       cA                BOOLEAN DEFAULT FALSE,
       pathLenConstraint INTEGER(0..MAX) OPTIONAL,
   }
~~~~

- `cA` attribute: Specifies whether the certificate subject may act as a CA. If yes, this attribute MUST be set to TRUE.
- `pathLenConstraint` attribute: This attribute is only relevant if the `cA` attribute is set to TRUE. It specifies the maximum number of CA certificates that may follow this CA certificate in the certification chain. Value "0" means that this CA may only issue end-entity certificates, but no CA certificates. If the attribute is not set, there is no limit to the allowed length of the certification path.

The settings of the `basicConstraints` extension differ for each control-plane certificate type. For more information, see the below sections describing the control-plane and voting certificates: [Control-Plane Root Certificate](#cp-root-cert), [Control-Plane CA Certificate](#cp-ca-cert), [Control-Plane AS Certificate](#cp-as-cert), and [Voting Certificates](#cp-voting-cert).



### Control-Plane Root Certificate {#cp-root-cert}

The control-plane root private key is used to sign control-plane CA certificates. Consequently, the control-plane root certificate with the control-plane root public key is used to verify control-plane CA certificates. So indirectly, CP root certificates determine which ASes act as CA in an ISD.

In X.509 terms, CP root certificates are self-*signed* CA certificates. That is, issuer and subject of the certificate are the same entity, and the public key in the root certificate can be used to verify the root certificate's signature. The CP root public key and proof of ownership of the private key are embedded in the Trust Root Configuration (TRC) of an Isolation Domain (ISD), via the self-signed CP root certificate. This facilitates the bootstrapping of trust within an ISD, and marks the CP root certificates as the starting point of an ISD's certificate verification path.

All constraints described in [General Certificate Requirements](#general-cert-req) also apply to CP root certificates.

The recommended **maximum validity period** of a CP root certificate is: 1 year.


#### Extension Constraints

The extensions of a CP root certificate differ from the general certificate requirements described previously, in the following ways.


##### `keyUsage` Extension

- `digitalSignature` attribute: This attribute MUST NOT be set (because the CP root certificate should not be used to verify control-plane messages).
- `keyCertSign` attribute: This attribute MUST be set.


##### `extKeyUsage` Extension

The `extKeyUsage` extension MUST be present in the CP root certificate.
It must be defined as follows:

- `id-kp-serverAuth` attribute: MUST NOT be set.
- `id-kp-clientAuth` attribute: MUST NOT be set.
- `id-kp-timeStamping` attribute: MUST be set.

Additionally, the `id-kp-root` attribute must be specified, as follows:

~~~~
   id-kp-root AttributeType ::= {id-ana id-cppki(1) id-kp(3) 3}
~~~~

where `id-ana` specifies the root SCION object identifier (OID).

<br>
**Note:** The SCION open source implementation currently uses the Anapaya IANA Private Enterprise Number (55324) as root SCION object identifier (OID):<br>
`id-ana ::= OBJECT IDENTIFIER {1 3 6 1 4 1 55324}`
<br>

##### `basicConstraints` Extension

The `basicConstraints` extension MUST be present in the CP root certificate.<br>
The extension attributes must be set as follows:

- `cA` attribute: MUST be set to TRUE.
- `pathLenConstraint` attribute: Should be set to "1". Additionally, it must be marked as "critical", according to [X.509](https://handle.itu.int/11.1002/1000/13031).
<br>

### Control-Plane CA Certificate {#cp-ca-cert}

The control-plane CA private key is used to sign control-plane AS certificates. Consequently, control-plane CA certificates holding the control-plane CA public key are used to verify control-plane AS certificates.

The public key needed to verify the CA certificate is in a CP root certificate. CA certificates do not bundle the root certificate needed to verify them. In order to verify a CA certificate, a pool of root certificates must first be extracted from one or more active TRCs (see also [Signing and Verifying Control-Plane Messages](#signing-verifying-cp-messages).

All constraints described in [General Certificate Requirements](#general-cert-req) also apply to control-plane CA certificates.

The recommended **maximum validity period** of a CP CA certificate is: 11 days.

#### Extension Constraints

The extensions of a CP CA certificate differ from the general certificate requirements described previously, in the following ways.


##### `keyUsage` Extension

- `digitalSignature` attribute: This attribute MUST NOT be set (because the control-plane CA certificate should not be used to verify control-plane messages).
- `keyCertSign` attribute: This attribute MUST be set.


##### `extKeyUsage` Extension

The `extKeyUsage` extension MAY be present in the CP CA certificate.

If the `extKeyUsage` extension is present in the CP CA certificate, the attributes `id-kp-serverAuth` and `id-kp-clientAuth` MUST NOT be set.


##### `basicConstraints` Extension

The `basicConstraints` extension MUST be present in the CP CA certificate.<br>
The extension attributes must be set as follows:

- `cA` attribute: MUST be set to TRUE.
- `pathLenConstraint` attribute: SHOULD be set to "0". This means that the CP CA certificate can only issue end-entity certificates. Additionally, the attribute must be marked as "critical", according to [X.509](https://handle.itu.int/11.1002/1000/13031).
<br>


### Control-Plane AS Certificate {#cp-as-cert}

SCION ASes sign control-plane messages, such as PCBs, with their AS private key. Consequently, control-plane AS certificates holding the corresponding AS public key are required to verify control-plane messages.

In X.509 terms, control-plane AS certificates are end-entity certificates. That is, they cannot be used to verify other certificates.

All constraints described in [General Certificate Requirements](#general-cert-req) also apply to control-plane AS certificates.

The recommended **maximum validity period** of a CP AS certificate is: 3 days.


#### Extension Constraints

The extensions of a CP AS certificate differ from the general certificate requirements described previously, in the following ways.

##### `keyUsage` Extension

- `digitalSignature` attribute: This attribute MUST be set.
- `keyCertSign` attribute: This attribute MUST NOT be set.


##### `extKeyUsage` Extension

The `extKeyUsage` extension MUST be present in the CP AS certificate.<br>
It must be defined as follows:

- `id-kp-serverAuth` attribute: MUST be set, if the CP AS certificate is used on the server-side of a control-plane TLS session establishment.
- `id-kp-clientAuth` attribute: MUST be set, if the CP AS certificate is used on the client-side of control-plane TLS session establishment.
- `id-kp-timeStamping` attribute: MUST be set.
<br>

##### `basicConstraints` Extension

Control-plane AS certificates should not include the ``basicConstraints`` extension.


### Voting Certificates {#cp-voting-cert}

There are two types of voting certificates: the (1) regular voting certificates and the (2) sensitive voting certificates. They contain the public keys associated with the private keys that are allowed to cast votes in the TRC update process. Voting certificates are X.509-style certificates.

Regular and sensitive voting certificates are used to verify regular and sensitive TRC updates, respectively.


#### Regular Voting Certificate {#reg-vote}

Regular voting certificates state which keys are allowed to cast votes in a regular update. In X.509 terms, regular voting certificates are self-signed end-entity certificates. This means that the issuer and subject of a regular voting certificate are the same entity, and the key within the certificate was used to sign the certificate. However, a regular voting certificate cannot be used to verify other certificates.

The constraints described in [General Certificate Requirements](#general-cert-req) also apply to regular voting certificates. There is one exception: A regular voting certificate is not required to include the `ISD-AS number` attribute in their distinguished name (for more information on this attribute, see [`ISD-AS number` Attribute](#isd-as-nr)).

The recommended **maximum validity period** of a regular voting certificate is: 1 year.


#### Sensitive Voting Certificate {#sens-vote}

Sensitive voting certificates specify which keys are allowed to cast votes in a sensitive update. In X.509 terms, sensitive voting certificates are self-signed end-entity certificates. This means that the issuer and subject of a sensitive voting certificate are the same entity, and the key within the certificate was used to sign the certificate. However, a sensitive voting certificate cannot be used to verify other certificates.

The constraints described in [General Certificate Requirements](#general-cert-req) also apply to sensitive voting certificates. There is one exception: A sensitive voting certificate is not required to include the `ISD-AS number` attribute in their distinguished name (for more information on this attribute, see [`ISD-AS number` Attribute](#isd-as-nr)).

The recommended **maximum validity period** of a sensitive voting certificate is: 5 years.


##### Extension Constraints of Voting Certificates

The extensions of both regular and sensitive voting certificates differ from the general certificate requirements described previously, in the following ways.

###### `keyUsage` Extension

The `keyUsage` extension is not required in a voting certificate.<br>
However, if this extension is present, both the `digitalSignature` and the `keyCertSign` attributes MUST NOT be set.

###### `extKeyUsage` Extension

The `extKeyUsage` extension MUST be present in a voting certificate.<br>
It must be defined as follows:

- `id-kp-serverAuth` attribute: MUST NOT be set.
- `id-kp-clientAuth` attribute: MUST NOT be set.
- `id-kp-timeStamping` attribute: MUST be set.

Additionally, the `id-kp-regular` / `id-kp-sensitive` attribute MUST be set, as follows:

- For a regular voting certificate:<br>
`id-kp-regular AttributeType ::= {id-ana id-cppki(1) id-kp(3) 1}`
- For a sensitive voting certificate:<br>
`id-kp-sensitive AttributeType ::= {id-ana id-cppki(1) id-kp(3) 1}`

where `id-ana` specifies the root SCION object identifier (OID).

<br>
**Note:** The SCION open source implementation currently uses the Anapaya IANA Private Enterprise Number (55324) as root SCION object identifier (OID):<br>
`id-ana ::= OBJECT IDENTIFIER {1 3 6 1 4 1 55324}`
<br>


###### `basicConstraints` Extension

The `basicConstraints` extension SHOULD NOT be part of a voting certificate.<br>
However, if this extension is present in a voting certificate, it MUST be defined as follows:

- `cA` attribute: MUST be set to FALSE.
- `pathLenConstraint` attribute: MUST NOT be present.
<br>
<br>


# Specification of the Trust Root Configuration {#trc-specification}

TODO

# Deploying the CP PKI - Specifications {#deploy-cp-pki}

TODO

## Signing and Verifying Control-Plane Messages {#signing-verifying-cp-messages}

TODO


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

