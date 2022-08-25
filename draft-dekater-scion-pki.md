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
  X.509: 
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

**Note**: For more detailed information on the SCION next-generation inter-domain architecture, see {{CHUAT22}}, especially Chapter 2, as well as the IETF Internet Drafts {{I-D.scion-overview}} and {{I-D.scion-components}}.

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

**Note**: The TRC of each ISD contains a trusted set of control-plane root certificates. This set builds the root of each ISD's verification path. For more information on the selection of this trusted set of root certificates, see [Specification of the Trust Root Configuration](#trc-specification).

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

(1) : K<sub>x</sub> = PK<sub>x</sub> + SK<sub>x</sub>, where x = certificate type, PK<sub>x</sub> = public key, and SK<sub>x</sub> = private key

| Name                   | Notation       	 | Signed with           	                  | Contains                                                | Validity (2) |              
|------------------------+-------------------+------------------------------------------+---------------------------------------------------------+--------------|
| TRC (trust root conf)  | TRC               | SK<sub>sens</sub>, SK<sub>reg</sub> (1)  | C<sub>root</sub>, C<sub>sens</sub>, C<sub>reg</sub> (1) | 1 year       |
| Sensitive voting cert. | C<sub>sens</sub>  | SK<sub>sens</sub>                        | PK<sub>sens</sub>                                       | 5 years      |
| Regular voting cert.   | C<sub>reg</sub>   | SK<sub>reg</sub>                         | PK<sub>reg</sub>                                        | 1 year       |
| CP root certificate    | C<sub>root</sub>  | SK<sub>root</sub>                        | PK<sub>root</sub>                                       | 1 year       |
| CP CA certificate      | C<sub>CA</sub>    | SK<sub>root</sub>                        | PK<sub>CA</sub>                                         | 11 days (3)  |
| CP AS certificate      | C<sub>AS</sub>    | SK<sub>CA</sub>                          | PK<sub>AS</sub>                                         | 3 days       |
{: #table-2 title="Certificates"}

(1) Multiple signatures and certificates of each type may be included in a TRC.
(2) Recommended maximum validity period.
(3) A validity of 11 days with 4 days overlap between two CA certificates is recommended to enable best possible operational procedures when performing a CA certificate rollover.

Figure 2 illustrates, at a high level, the relationship between a TRC and the five types of certificates. 

>>>>>>>>>>>>>>> **figure 2** - *TRC update chain and the different types of associated certificates. Arrows show how signatures are verified; in other words, they indicate that a public key contained in a certificate or TRC can be used to verify the authenticity of another item.*


## Certificate Specification

This section provides an in-depth specification of the SCION certificates. The SCION certificate specification builds on top of {{RFC5280}}, which in turn builds on top of {{X.509}}. However, the SCION specification is more restrictive.

This section defines the additional constraints compared to [RFC5280]_ for each type of SCION control-plane certificate. The recommended settings for optional constraints are based on the SCION open source implementation [scionproto]_. Adjusting the optional constraints to the requirements of a customer implementation is possible and allowed.


### Control-Plane Root Certificate {#cp-root-cert}

TODO

### Control-Plane CA Certificate {#cp-ca-cert}

TODO

### Control-Plane AS Certificate {#cp-as-cert}

TODO

### Voting Certificates {#cp-voting-cert}

TODO



# Specification of the Trust Root Configuration {#trc-specification}

TODO


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

Thanks to Juan, François and Jordi (+ Adrian?)
