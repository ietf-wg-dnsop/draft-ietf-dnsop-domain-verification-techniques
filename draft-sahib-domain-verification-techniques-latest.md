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
  RFC1034:
  RFC1035:
  RFC2119:
  RFC1464:
  RFC4033:

informative:


    RFC8555:

    LETSENCRYPT:
        title: "Challenge Types: DNS-01 challenge"
        date: 2020
        author:
          - ins: Let's Encrypt
        target: https://letsencrypt.org/docs/challenge-types/#dns-01-challenge

    GOOGLE-WORKSPACE-TXT:
        title: "TXT record values"
        author:
          - ins: Google
        target: https://support.google.com/a/answer/2716802

    GOOGLE-WORKSPACE-CNAME:
        title: "CNAME record values"
        author:
          - ins: Google
        target: https://support.google.com/a/answer/112038

    ACM-CNAME:
        title: "Option 1: DNS Validation"
        author:
          - ins: AWS
        target: https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html

    GITHUB-TXT:
        title: "Verifying your organization's domain"
        author:
          - ins: GitHub
        target: https://docs.github.com/en/github/setting-up-and-managing-organizations-and-teams/verifying-your-organizations-domain











--- abstract

Verification of ownership of domains in the Domain Name System (DNS) {{RFC1034}} {{RFC1035}} often relies on adding or editing DNS records within the domain. This document lays out the various techniques and the pros and cons of each.

--- middle

# Introduction

Many providers on the internet need users to prove that they control a particular domain before granting them some sort of privilege associated with that domain. For instance, certificate authorities like Let's Encrypt {{LETSENCRYPT}} ask requesters of TLS certificates to prove that they operate the domain they're requesting the certificate for. Providers generally allow for several different ways of proving domain control, some of which include manipulating DNS records. This document focuses on DNS techniques for domain verification; other techniques (such as email or HTML verification) are out-of-scope.

In practice, DNS-based verification often looks like the provider generating a random value and asking the requester to create a DNS record containing this random value and placing it at a location that the provider can query for. Generally only one temporary DNS record is sufficient for proving domain ownership.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Verification Techniques

## TXT based

Although the original DNS protocol specifications did not associate any semantics with the DNS TXT record, {{RFC1464}} describes how to use them to store attributes in the form of ASCII text key-value pairs for a particular domain.

       host.widgets.com   IN   TXT   "printer=lpr5"

In practice, there is wide variation in the content of DNS TXT records used for domain verification, and they often do not follow the key-value pair model.

The same domain name can have multiple distinct TXT records (a TXT Record Set).

TXT record-based DNS domain verification is usually the default option for DNS verification. The service provider asks the user to add a DNS TXT record (perhaps through their domain host or DNS provider) at the domain with a certain value. Then, the service provider does a DNS TXT query for the domain being verified and checks that the value exists. For example, this is what a DNS TXT verification record could look like:

       example.com.   IN   TXT   "foo-verification=bar"

Here, the value "bar" for the attribute "foo-verification" serves as the randomly-generated TXT value being added to prove ownership of the domain to Foo provider. The value is usually a randomly-generated token in order to guarantee that the entity who requested that the domain be verified (i.e. the person managing the account at Foo provider) is the one who has (direct or delegated) access to DNS records for the domain. The generated token typically expires in a few days. The TXT record is usually placed at the domain being verified ("example.com" in the example above). After a TXT record has been added, the service provider will usually take some time to verify that the DNS TXT record with the expected token exists for the domain.

One drawback of this method is that the TXT record is typically placed at the domain name being verified. If many services are attempting to verify the domain name, many distinct TXT records end up being placed at that name. Since DNS Resource Record sets are treated atomically, all TXT records must be returned to the querier, increasing the size of the response. There is no way to surgically query only the TXT record for a specific service.


### Examples

#### Let's Encrypt

Let's Encrypt {{LETSENCRYPT}} has a challenge type  `DNS-01` that lets a user prove domain ownership in accordance with the ACME protocol {{RFC8555}}. In this challenge, Let's Encrypt asks you to create a TXT record with a randomly-generated token at `_acme-challenge.<YOUR_DOMAIN>`. For example, if you wanted to prove domain ownership of `example.com`, Let's Encrypt could ask you to create the DNS record:

        _acme-challenge.example.com.  IN  TXT "cE3A8qQpEzAIYq-T9DWNdLJ1_YRXamdxcjGTbzrOH5L"

{{RFC8555}} (section 8.4) places requirements on the random value.


#### Google Workspace

{{GOOGLE-WORKSPACE-TXT}} asks the user to sign in with their administrative account and obtain their verification token as part of the setup process for Google Workspace. The verification token is a 68-character string that begins with "google-site-verification=", followed by 43 characters. Google recommends a TTL of 3600 seconds. The owner name of the TXT record is the domain or subdomain neme being verified.


#### GitHub

GitHub asks you to create a DNS TXT record under `_github-challenge-ORGANIZATION-<your-domain>`, where ORGANIZATION stands for the GitHub organization name {{GITHUB-TXT}}. The code is a numeric code that expires in 7 days.

<!-- #### DigiCert

#### Facebook Business Manager

#### Amazon SES -->



## CNAME based

Less commonly than TXT record verification, service providers also provide the ability to verify domain ownership via CNAME records. This is used in case the user cannot create TXT records. One common reason is that the domain name may already have CNAME record that aliases it to a 3rd-party target domain. CNAMEs have a technical restriction that no other record types can be placed along side them at the same domain name ({{RFC1034}}, Section 3.6.2).. The CNAME based domain verification method teypically uses a randomized label prepended to the domain name being verified.


### Examples

#### Google
{{GOOGLE-WORKSPACE-CNAME}} lets you specify a CNAME record for verifying domain ownership. The user gets a unique 12-character string that is added as "Host", with TTL 3600 (or default) and Destination an 86-character string beginning with "gv-" and ending with ".domainverify.googlehosted.com.".

To verify a subdomain, the unique 12-character string is appended with the subdomain name for "Host" field for e.g. JLKDER712AFP.subdomain where subdomain is the subdomain being verified.


#### AWS Certificate Manager (ACM)

To get issued a certificate by AWS Certificate Manager (ACM), you can create a CNAME record to verify domain ownership {{ACM-CNAME}}. The record name for the CNAME looks like `_<random-token1>.example.com`, which would point to `_<random-token2>.<random-token3>.acm-validations.aws.`

Note that if there are more than 5 CNAMEs being chained, then this method does not work.

# Recommendations

## TXT vs CNAME

## TXT recommendations

## CNAME recommendations


# Security Considerations

DNSSEC {{RFC4033}} should be employed by the domain owner to protect against domain name spoofing.


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO
