---
title: "Survey of Domain Verification Techniques using DNS"
abbrev: "Domain Verification Techniques"
docname: draft-ietf-dnsop-domain-verification-techniques-latest
category: info

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

informative:


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

Many services on the Internet need to verify ownership or control of a domain in the Domain Name System (DNS) {{RFC1034}} {{RFC1035}}. This verification is often done by requesting a specific DNS record to be visible in the domain. This document surveys various techniques in wide use today, the pros and cons of each, and proposes some practices to avoid known problems.

--- middle



# Introduction

Many providers of internet services need domain owners to prove that they control a particular domain before they can operate a services or grant some privilege to the associated domain. For instance, certificate authorities (CAs) ask requesters of TLS certificates to prove that they operate the domain they are requesting the certificate for. Providers generally allow for several different ways of proving domain control. This document describes common practices and pitfalls associated with using DNS based techniques for domain verification. Other techniques such as email or HTTP(S) based verification are out-of-scope.

In practice, DNS-based verification takes the form of the provider generating a random value visible only to the requester, and then asking the requester to create a DNS record containing this random value and placing it at a location within the domain that the provider can query for. Generally only one temporary DNS record is sufficient for proving domain ownership, although sometimes the DNS record must be kept in the zone to prove continued ownership of the domain.

Based on the survey, this document also recommends using TXT-based domain verification which is time-bound and targeted to the service.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

Provider: an internet-based provider of a service, for e.g., a Certificate Authority or a service that allows for user-controlled websites. These services often require a user to verify that they control a domain.


# Verification Techniques

## TXT based {#txt-based}

TXT record-based DNS domain verification is usually the default option for DNS verification. The service provider asks the user to add a DNS TXT record (perhaps through their domain host or DNS provider) at the domain with a certain value. Then, the service provider does a DNS TXT query for the domain being verified and checks that the value exists. For example, this is what a DNS TXT verification record could look like:

    example.com.   IN   TXT   "foo-verification=bar-237943648324687364"

Here, the value "bar-237943648324687364" for the attribute "foo-verification" serves as the randomly-generated TXT value being added to prove ownership of the domain to Foo provider. Although the original DNS protocol specifications did not associate any semantics with the DNS TXT record, {{RFC1464}} describes how to use them to store attributes in the form of ASCII text key-value pairs for a particular domain. In practice, there is wide variation in the content of DNS TXT records used for domain verification, and they often do not follow the key-value pair model. Even so, the rdata portion of the DNS TXT record has to contain the value being used to verify the domain. The value is usually a randomly-generated token in order to guarantee that the entity who requested that the domain be verified (i.e. the person managing the account at Foo provider) is the one who has (direct or delegated) access to DNS records for the domain. The generated token typically expires in a few days. The TXT record is placed at the domain being verified ("example.com" in the example above). After a TXT record has been added, the service provider will usually take some time to verify that the DNS TXT record with the expected token exists for the domain. Some providers expire the code after a set amount of time.

As an example, the ACME protocol {{RFC8555}} has a challenge type  `DNS-01` that lets a user prove domain ownership. In this challenge, an implementing CA asks you to create a TXT record with a randomly-generated token at `_acme-challenge.<YOUR_DOMAIN>`:

    _acme-challenge.example.com.  IN  TXT "cE3A8qQpEzAIYq-T9DWNdLJ1_YRXamdxcjGTbzrOH5L"

{{RFC8555}} (section 8.4) places requirements on the random value.


## CNAME based

Less commonly than TXT record verification, service providers also provide the ability to verify domain ownership via CNAME records. One reason for using CNAME is for the case where the user cannot create TXT records. One common reason is that the domain name may already have CNAME record that aliases it to a 3rd-party target domain. CNAMEs have a technical restriction that no other record types can be placed along side them at the same domain name ({{RFC1034}}, Section 3.6.2). The CNAME based domain verification method typically uses a randomized label prepended to the domain name being verified. For example:

    _random-token1.example.com.   IN   CNAME _random-token2.validation.com.`


## Common Patterns

### Name

Some providers use a suffix of `_PROVIDER_NAME-challenge` in the Name field of the TXT record challenge. For ACME, the full Host is `_acme-challenge.<YOUR_DOMAIN>`. Such patterns are useful for doing targeted domain verification, as discussed in {{targeted-domain-verification}} because if the provider knows what it is looking for (domain in the case of ACME) it can specifically do a DNS query for that TXT record, as opposed to having to do a TXT query for the apex.

ACME does the same name construction for CNAME records.

### RDATA

One pattern that quite a few providers follow is constructing the rdata of the TXT DNS record in the form of `PROVIDER-SERVICE-domain-verification=` followed by the random value being checked for. This is in accordance with {{RFC1464}} which mandates that attributes must be stored as key-value pairs.

# Recommendations

## Targeted Domain Verification {#targeted-domain-verification}

The TXT record being used for domain verification is most commonly placed at the domain name being verified. For example, if `example.com` is being verified, then the DNS TXT record will have `example.com` in the Name section. Unfortunately, this practice does not scale very well. Many services are now attempting to verify domain names, causing many of these TXT records to be placed at that same location at the top of the domain (the APEX).

When a DNS administrator sees 15 DNS TXT records for their domain based on only random letters, they can no longer determine which service or vendor the DNS TXT records were added for. This causes administrators to leave all DNS TXT records in there, as they want to avoid breaking a service. Over time, the domain ends up with a lot of unnecessary, unknown and untraceable DNS TXT records.

It is recommended that providers use a prefix (eg "\_foo.example.com") instead of using the top of the domain ("APEX") directly, such as:

    _foo.example.com.  IN   TXT    "bar-237943648324687364"

An operational issue arises from the DNS protocol only being able to query for "all TXT records" at a single location: if multiple services all require TXT records, this can cause the DNS answer for TXT records to become very large. It has been observed that some well known domains had so many services deployed that their DNS TXT answer did not fit in a single UDP DNS packet. This results in fragmentation which is known to be vulnerable to various attacks {{!AVOID-FRAGMENTATION=I-D.ietf-dnsop-avoid-fragmentation}}. It can also lead to UDP packet truncation, causing a retry over TCP. Not all networks properly transport DNS over TCP and some DNS software mistakenly believe TCP support is optional {{RFC9210}}.

A malicious service that promises to deliver something after domain verification could surreptitiously ask another service provider to start processing or sending mail for the target domain and then present the victim domain administrator with this DNS TXT record pretending to be for their service. Once the administrator has added the DNS TXT record, instead of getting their service, their domain is now certifying another service of which they are not aware they are now a consumer. If services use a clear description and name attribution in the required DNS TXT record, this can be avoided. For example by requiring a DNS TXT record at \_vendorname.example.com instead of at example.com, a malicious service could no longer replay this without the DNS administrator noticing this.


## TXT vs CNAME

CNAME records cannot co-exist with any other data. What happens when both a CNAME and other data such as a TXT record or NS record exist depends on the DNS implementation. But most likely, either the CNAME or the other records will be silently ignored. The user interface for adding a record might not check for this. It might also break in unexpected ways: if a CNAME is added for continuous authorization, and for another service a TXT record is added, the TXT record might work but the CNAME record might break.

Another issue with CNAME records is that they MUST NOT point to another CNAME. But where this might be true in an initial deployment, if the target that the CNAME points to is changed from a non-CNAME record to a CNAME record, some DNS software might no longer resolve this as expected.

Early web-based DNS administration tools did not always have the TXT record available in the menu for DNS record types, while CNAME would be available. However as many anti-spam measures now require TXT records, they are now widely supported. The CNAME method should only be used for delegating authorization to an actual subdomain, for example:

    recruitment.example.com.   IN   CNAME   example.recruitment-vendor.com.

## Time-bound checking

After domain verification is done, there is typically no need for the TXT or CNAME record to continue to exist as the presence of the domain-verifying DNS record for a service only implies that a user with access to the service also has DNS control of the domain at the time the code was generated. It should be safe to remove the verifying DNS record once the verification is done and the service provider doing the verification should specify how long the verification will take (i.e. after how much time can the verifying DNS record be deleted).

If a provider will use the DNS TXT record only for a one-time verification, they should clearly indicate this in the RDATA of the TXT record, so a DNS administrator at the target domain can easily spot an obsolete record in the future. For example:

    _provider-token.example.com.   IN   TXT "type=activation_only expiry=2023-10-12 token=TOKENDATA"

If a provider requires the continued presence of the TXT record as proof that the domain owner is still authorizing the service, this should also be clear from the TXT record RDATA. For example:

    _provider-service.example.com.   IN   TXT "type=continued_service expiry=never token=TOKENDATA"

# Email sending authorization

Some vendors use a hosted service that wants to generate emails that appear to be from the customer. When a customer has deployed anti-spam meassures such as DKIM {{RFC6376}}, DMARC {{RFC7489}} or SPF {{RFC7208}}, the vendor's mail service needs to be added to the list of allowed mail servers. However, some customers might not want to give permission for a vendor to send emails from their entire domain. It is recommended that a vendor uses a subdomain. If the vendor's domain is example-vendor.com, and the customer domain is example-customer.com, the vendor could use the subdomain example-customer.example-vendor.com to send emails. Alternatively, the customer could delegate a subdomain example-vendor.example-customer.com to the vendoer for email sending, as those email addresses would have a stronger origin appearance of being emails send by the customer to their clients.

 Besides requiring proof of ownership of the domain, the customer needs to authorize the hosted service to send email on their behalf.

# Security Considerations

Both the provider and the service being authenticated and authorized should be obvious from the TXT content to prevent malicious services from misleading the domain owner into certifying a different provider or service.

DNSSEC {{RFC4033}} can be employed by the domain owner to protect against domain name spoofing.

# Operational Considerations

Consumers of the provider services need to relay information from a provider's website to their local DNS administrators. The exact DNS record type, content and location is often not clear when the DNS administrator receives the information, especially to consumers who are not DNS experts. Providers should offer extremely detailed help pages, that are accessible without needing a login on the provider website, as the DNS adminstrator often has no login account on the provider service website. Similarly, for clarity, the exact and full DNS record (including a Fully Qualified Domain Name) to be added should be provided along with help instructions.

# IANA Considerations

This document has no IANA actions.



--- back

# Appendix

The survey done in this document found several varying methods for DNS domain verification techniques across providers. This Appendix lists them, for completeness.

## Let's Encrypt

The ACME example in {{txt-based}} is implemented by Let's Encrypt {{LETSENCRYPT}}.

## Google Workspace

{{GOOGLE-WORKSPACE-TXT}} asks the user to sign in with their administrative account and obtain their verification token as part of the setup process for Google Workspace. The verification token is a 68-character string that begins with "google-site-verification=", followed by 43 characters. Google recommends a TTL of 3600 seconds. The owner name of the TXT record is the domain or subdomain neme being verified.

{{GOOGLE-WORKSPACE-CNAME}} lets you specify a CNAME record for verifying domain ownership. The user gets a unique 12-character string that is added as "Host", with TTL 3600 (or default) and Destination an 86-character string beginning with "gv-" and ending with ".domainverify.googlehosted.com.".

## GitHub

GitHub asks you to create a DNS TXT record under `_github-challenge-ORGANIZATION-<YOUR_DOMAIN>`, where ORGANIZATION stands for the GitHub organization name {{GITHUB-TXT}}. The code is a numeric code that expires in 7 days. This fits under {{targeted-domain-verification}}.

## AWS Certificate Manager (ACM)

To get issued a certificate by AWS Certificate Manager (ACM), you can create a CNAME record to verify domain ownership {{ACM-CNAME}}. The record name for the CNAME looks like:

     `_<random-token1>.example.com.   IN   CNAME _RANDOM-TOKEN.acm-validations.aws.`

Note that if there are more than 5 CNAMEs being chained, then this method does not work.

## Atlassian

Some services ask the DNS record to exist in perpetuity {{ATLASSIAN-VERIFY}}. If the record is removed, the user gets a limited amount of time to re-add it before they lose domain verification status.
