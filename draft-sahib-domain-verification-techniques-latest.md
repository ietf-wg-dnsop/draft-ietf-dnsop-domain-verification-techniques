---
title: "Survey of Domain Verification Techniques using DNS"
abbrev: "Domain Verification Techniques"
docname: draft-sahib-domain-verification-techniques
category: info

ipr: trust200902
area: General
workgroup: Network Working Group
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: S. Sahib
    name: Shivan Sahib
    organization: Salesforce
    email: shivankaulsahib@gmail.com

normative:
  RFC2119:

informative:

    LETSENCRYPT:
        title: "Challenge Types: DNS-01 challenge"
        date: 2020
        author:
          - ins: Let's Encrypt
        target: https://letsencrypt.org/docs/challenge-types/#dns-01-challenge



--- abstract

Domain verification on the web is often DNS-based. This document lays out the different techniques and the pros and cons of each.

--- middle

# Introduction

Several providers on the internet need users to prove that they control a particular domain before granting them some sort of privilege associated to that domain. For instance, certificate authorities like Let's Encrypt {{LETSENCRYPT}} ask requesters of TLS certificates to prove that they operate the domain they're requesting the certificate. Providers generally allow for several different ways of proving domain control, some of which include manipulating DNS records.

In practice, what this looks like is the provider generating a random value and asking the requester to create a DNS record containing this random value and placing it at a location that the provider can query for.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# TXT based

## Google

## Let's Encrypt

## GitHub

## DigiCert

# CNAME based


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
