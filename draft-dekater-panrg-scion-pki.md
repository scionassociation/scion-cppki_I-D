---
title: "SCION Control-Plane PKI"
abbrev: "SCION CPPKI"
category: info
submissiontype: IRTF

docname: draft-dekater-panrg-scion-pki-latest
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
  latest: "https://scionassociation.github.io/scion-cppki_I-D/draft-dekater-panrg-scion-pki.html"

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

## Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
