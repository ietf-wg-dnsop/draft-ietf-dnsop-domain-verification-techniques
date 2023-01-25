---
title: "Domain Verification Techniques using DNS"
abbrev: "Domain Verification Techniques"
docname: draft-ietf-dnsop-domain-verification-techniques-latest
category: bcp

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
stream: IETF
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: S. Sahib
    name: Shivan Sahib
    organization: Brave Software
    email: shivankaulsahib@gmail.com

 -
    ins: S. Huque
    name: Shumon Huque
    organization: Salesforce
    email: shuque@gmail.com

 -
    ins: P. Wouters
    name: Paul Wouters
    organization: Aiven
    email: paul.wouters@aiven.io

normative:
  RFC1034:
  RFC1035:
  RFC2119:
  RFC1464:
  RFC4033:
  RFC8174:
  draft-ietf-dnsop-dnssec-bcp:

  SHA256:
      title: "Secure Hash Standard (SHS), NIST FIPS 180-4"
      date: 2015
      author:
        - ins: National Institute of Standards and Technology
      target: https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf

informative:
    RFC4086:
    RFC8555:
    RFC6376:
    RFC7208:
    RFC7489:
    RFC9210:

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

    ATLASSIAN-VERIFY:
        title: "Verify over DNS"
        author:
          - ins: Atlassian
        target: https://support.atlassian.com/user-management/docs/verify-a-domain-to-manage-accounts/#Verify-over-DNS



--- abstract

Many services on the Internet need to verify ownership or control of a domain in the Domain Name System (DNS). This verification is often done by requesting a specific DNS record to be visible in the domain. There are a variety of techniques in use today, with different pros and cons. This document proposes some practices to avoid known problems.

--- middle



# Introduction

Many providers of internet services need domain owners to prove that they control a particular domain before they can operate a services or grant some privilege to the associated domain. For instance, certificate authorities (CAs) ask requesters of TLS certificates to prove that they operate the domain they are requesting the certificate for. Providers generally allow for several different ways of proving domain control. In practice, DNS-based verification takes the form of the provider generating a random value visible only to the requester, and then asking the requester to create a DNS record containing this random value and placing it at a location within the domain that the provider can query for. Generally only one temporary DNS record is sufficient for proving domain ownership, although sometimes the DNS record must be kept in the zone to prove continued ownership of the domain.

This document describes common practices and pitfalls associated with using DNS based techniques for domain verification, and recommends using TXT-based domain verification which is time-bound and targeted to the service. Other techniques such as email or HTTP(S) based verification are out-of-scope.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}}
when, and only when, they appear in all capitals, as shown here.

Provider: an internet-based provider of a service, for e.g., a Certificate Authority or a service that allows for user-controlled websites. These services often require a user to verify that they control a domain.

APEX: the 'top' of the domain name. From the user perspective, the highest level of "their" domain name.

    # this record is at the APEX of the domain example.com.
    example.com.   IN   NS   a.iana-servers.net.
    # this record is NOT at the APEX of the domain example.com.
    something.example.com.   IN   A   192.0.2.1

Random Token: a random value that uniquely identifies the DNS domain verification challenge.


# Survey of Verification Techniques

## TXT based {#txt-based}

TXT record-based DNS domain verification is usually the default option for DNS verification. The service provider asks the user to add a DNS TXT record (perhaps through their domain host or DNS provider) at the domain with a certain value. Then the service provider does a DNS TXT query for the domain being verified and checks that the value exists. For example, this is what a DNS TXT verification record could look like for a provider Foo:

    example.com.   IN   TXT   "237943648324687364"

Here, the value "237943648324687364" serves as the randomly-generated TXT value being added to prove ownership of the domain to Foo provider. Note that in this construction provider Foo would have to query for all TXT records at "example.com" to get the validating record. Although the original DNS protocol specifications did not associate any semantics with the DNS TXT record, {{RFC1464}} describes how to use them to store attributes in the form of ASCII text key-value pairs for a particular domain. In practice, there is wide variation in the content of DNS TXT records used for domain verification, and they often do not follow the key-value pair model. Even so, the RDATA [{{RFC1034}}] portion of the DNS TXT record has to contain the value being used to verify the domain. The value is usually a Random Token in order to guarantee that the entity who requested that the domain be verified (i.e. the person managing the account at Foo provider) is the one who has (direct or delegated) access to DNS records for the domain. After a TXT record has been added, the service provider will usually take some time to verify that the DNS TXT record with the expected token exists for the domain. The generated token typically expires in a few days. See {{appendix}} for a survey of different implementations.

Some providers use a suffix of `_PROVIDER_NAME-challenge` in the Name field of the TXT record challenge. For ACME, the full Host is `_acme-challenge.<YOUR_DOMAIN>`. Such patterns are useful for doing targeted domain verification. The ACME protocol {{RFC8555}} has a challenge type  `DNS-01` that lets a user prove domain ownership. In this challenge, an implementing CA asks you to create a TXT record with a randomly-generated token at `_acme-challenge.<YOUR_DOMAIN>`:

    _acme-challenge.example.com.  IN  TXT "cE3A8qQpEzAIYq-T9DWNdLJ1_YRXamdxcjGTbzrOH5L"

{{RFC8555}} (section 8.4) places requirements on the Random Token.

An operational issue arises from the DNS protocol only being able to query for "all TXT records" at a single location: if multiple services all require TXT records, this can cause the DNS answer for TXT records to become very large. It has been observed that some well known domains had so many services deployed that their DNS TXT answer did not fit in a single UDP DNS packet. This results in fragmentation which is known to be vulnerable to various attacks ({{!AVOID-FRAGMENTATION=I-D.ietf-dnsop-avoid-fragmentation}}). It can also lead to UDP packet truncation, causing a retry over TCP. Not all networks properly transport DNS over TCP and some DNS software mistakenly believe TCP support is optional ({{RFC9210}}).

A malicious service that promises to deliver something after domain verification could surreptitiously ask another service provider to start processing or sending mail for the target domain and then present the victim domain administrator with this DNS TXT record pretending to be for their service. Once the administrator has added the DNS TXT record, instead of getting their service, their domain is now certifying another service of which they are not aware they are now a consumer. If services use a clear description and name attribution in the required DNS TXT record, this can be avoided. For example, by requiring a DNS TXT record at \_vendorname.example.com instead of at example.com, a malicious service could no longer replay this without the DNS administrator noticing this.

## CNAME based

Less commonly than TXT record verification, service providers also provide the ability to verify domain ownership via CNAME records. One reason for using CNAME is for the case where the user cannot create TXT records; for example, when the domain name may already have a CNAME record that aliases it to a 3rd-party target domain. CNAMEs have a technical restriction that no other record types can be placed along side them at the same domain name [{{RFC1034}}, Section 3.6.2]. The CNAME based domain verification method typically uses a randomized label prepended to the domain name being verified. For example:

    _random-token1.example.com.   IN   CNAME _random-token2.validation.com.`


When a third-party validation provider is used, both the client and the service provider need to give the validation provider a random token, so that the validation provider can confirm the client request is unique and bound to the service provider's request.

## Time-bound checking

After domain verification is done, there is typically no need for the TXT or CNAME record to continue to exist as the presence of the domain-verifying DNS record for a service only implies that a user with access to the service also has DNS control of the domain at the time the code was generated. It should be safe to remove the verifying DNS record once the verification is done and the service provider doing the verification should specify how long the verification will take (i.e. after how much time can the verifying DNS record be deleted).


### Recommendations

#### TXT Record

DNS TXT records are the RECOMMENDED method of doing DNS-based domain verification. The provider constructs the validation domain name by prepending a provider-relevant prefix followed by "-challenge" to the domain name being validated (e.g. "\_foo-challenge.example.com"). The RDATA of the TXT resource record MUST be the output of the following:

1. Generate a Random Token with at least 128 bits of entropy.
2. Take the SHA-256 digest output {{SHA256}} of it.
3. base64url encode it.

See {{RFC4086}} for additional information on randomness requirements.

For example:

    _foo-challenge.example.com.  IN   TXT  "3419...3d206c4"

If a provider has an application-specific need to have multiple verifications for the same label, multiple prefixes can be used:

    _feature1._foo-challenge.example.com.  IN   TXT  "3419...3d206c4"

This again allows the provider to query only for application-specific records it needs, while giving flexibility to the user adding the DNS verification record (i.e. they can be given permission to only add records under a specific prefix by the DNS administrator). Whether or not multiple verifying records can exist for the same domain is up to the implementation.

Providers MUST provide clear instructions on when a verifying record can be removed. The user SHOULD de-provision the resource record(s) provisioned for a DNS-based domain verification challenge once the challenge is complete.

Consumers of the provider services need to relay information from a provider's website to their local DNS administrators. The exact DNS record type, content and location is often not clear when the DNS administrator receives the information, especially to consumers who are not DNS experts. Providers SHOULD offer detailed help pages, that are accessible without needing a login on the provider website, as the DNS adminstrator often has no login account on the provider service website. Similarly, for clarity, the exact and full DNS record (including a Fully Qualified Domain Name) to be added SHOULD be provided along with help instructions.

## CNAME Record

CNAME records cannot co-exist with any other data; what happens when both a CNAME and other records exist depends on the DNS implementation, and might break in unexpected ways. If a CNAME is added for continuous authorization, and for another service a TXT record is added, the TXT record might work but the CNAME record might break. Another issue with CNAME records is that they must not point to another CNAME. But while this might be true in an initial deployment, if the target that the CNAME points to is changed from a non-CNAME record to a CNAME record, some DNS software might no longer resolve this as expected. However, when using a properly named prefix, existing CNAME records should never conflict with regular CNAME records.

It is therefore NOT RECOMMENDED to use CNAMEs for DNS domain verification.


# Security Considerations

Both the provider and the service being authenticated and authorized should be obvious from the TXT content to prevent malicious services from misleading the domain owner into certifying a different provider or service.

DNSSEC [{{draft-ietf-dnsop-dnssec-bcp}}] SHOULD be employed by the domain owner to protect their domain verification records against DNS spoofing attacks.

DNSSEC validation MUST be enabled by service providers that verify domain verification records they have issued and when no DNSSEC support is detected for the domain owner zone, SHOULD attempt to query and confirm by matching the validation record using multiple DNS validators on (preferably) unpredictable geographically diverse IP addressses to reduce an attackers capability of DNS spoofing. Alternatively, service providers MAY perform multiple queries spread out over a longer time period to reduce the chance of receiving spoofed DNS answers.


# IANA Considerations

This document has no IANA actions.


--- back

# Appendix {#appendix}

The survey done in this document found several varying methods for DNS domain verification techniques across providers. This Appendix lists them, for completeness.

## Let's Encrypt

The ACME example in {{txt-based}} is implemented by Let's Encrypt {{LETSENCRYPT}}.

## Google Workspace

{{GOOGLE-WORKSPACE-TXT}} asks the user to sign in with their administrative account and obtain their verification token as part of the setup process for Google Workspace. The verification token is a 68-character string that begins with "google-site-verification=", followed by 43 characters. Google recommends a TTL of 3600 seconds. The owner name of the TXT record is the domain or subdomain neme being verified.

{{GOOGLE-WORKSPACE-CNAME}} lets you specify a CNAME record for verifying domain ownership. The user gets a unique 12-character string that is added as "Host", with TTL 3600 (or default) and Destination an 86-character string beginning with "gv-" and ending with ".domainverify.googlehosted.com.".

## GitHub

GitHub asks you to create a DNS TXT record under `_github-challenge-ORGANIZATION-<YOUR_DOMAIN>`, where ORGANIZATION stands for the GitHub organization name {{GITHUB-TXT}}. The code is a numeric code that expires in 7 days.

## AWS Certificate Manager (ACM)

To get issued a certificate by AWS Certificate Manager (ACM), you can create a CNAME record to verify domain ownership {{ACM-CNAME}}. The record name for the CNAME looks like:

     `_<random-token1>.example.com.   IN   CNAME _RANDOM-TOKEN.acm-validations.aws.`

Note that if there are more than 5 CNAMEs being chained, then this method does not work.

## Atlassian

Some services ask the DNS record to exist in perpetuity {{ATLASSIAN-VERIFY}}. If the record is removed, the user gets a limited amount of time to re-add it before they lose domain verification status.
