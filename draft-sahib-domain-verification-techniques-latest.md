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

 -
    ins: S. Huque
    name: Shumon Huque
    organization: Salesforce
    email: shuque@gmail.com

normative:
  RFC2119:
  RFC1464:

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

Several providers on the internet need users to prove that they control a particular domain before granting them some sort of privilege associated with that domain. For instance, certificate authorities like Let's Encrypt {{LETSENCRYPT}} ask requesters of TLS certificates to prove that they operate the domain they're requesting the certificate for. Providers generally allow for several different ways of proving domain control, some of which include manipulating DNS records. This document focuses on DNS techniques for domain verification; other techniques (such as email verification) are out-of-scope.

In practice, DNS-based verification often looks like the provider generating a random value and asking the requester to create a DNS record containing this random value and placing it at a location that the provider can query for. Generally only one DNS record is sufficient for proving domain ownership.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Verification Techniques

## TXT based

{{RFC1464}} describes how to use DNS TXT records to store attributes in the form of ASCII text key-value pairs for a particular domain.

       host.widgets.com   IN   TXT   "printer=lpr5"

One domain can have multiple TXT records.

TXT record-based DNS domain verification is usually the default option for DNS verification. The provider asks the user to add a DNS TXT record at the domain with a certain value. Then, the provider does a DNS TXT query for the domain being verified and checks that the value exists. In practice, DNS TXT records used for domain verification often do not follow the practice of key=value pairs, as evidenced in the examples.

### Examples

#### Google Workspace

#### Let's Encrypt

#### GitHub

#### DigiCert

#### Facebook Business Manager

#### Amazon SES



## CNAME based

### Examples

#### Google

#### AWS Certificate Manager (ACM)

# Recommendations

## TXT vs CNAME

## TXT recommendations

## CNAME recommendations


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
