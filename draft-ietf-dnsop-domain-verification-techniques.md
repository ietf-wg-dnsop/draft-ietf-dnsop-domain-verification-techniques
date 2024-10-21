---
title: "Domain Control Validation using DNS"
abbrev: "Domain Control Validation using DNS"
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

 -
    ins: E. Nygren
    name: Erik Nygren
    organization: Akamai Technologies
    email: erik+ietf@nygren.org

normative:
  RFC1034:
  RFC2119:
  RFC8174:
  RFC9364:

informative:
    RFC1464:
    RFC4086:
    RFC4343:
    RFC8555:
    RFC9210:
    RFC6672:
    RFC8659:
    RFC9499:
    RFC3339:
    I-D.draft-tjw-dbound2-problem-statement:

    PSL:
        title: "Public Suffix List"
        date: 2022
        author:
          - ins: Mozilla Foundation
        target: https://publicsuffix.org/

    PSL-DIVISIONS:
        title: "Public Suffix List format"
        date: 2022
        author:
          - ins: J. Frakes
        target: "https://github.com/publicsuffix/list/wiki/Format#divisions"

    AVOID-FRAGMENTATION:
        title: "Fragmentation Avoidance in DNS"
        date: 2023
        author:
          - ins: K. Fujiwara
          - ins: P. Vixie
        target: https://datatracker.ietf.org/doc/draft-ietf-dnsop-avoid-fragmentation/

    DNS-01:
        title: "Challenge Types: DNS-01 challenge"
        date: 2020
        author:
          - ins: Let's Encrypt
        target: https://letsencrypt.org/docs/challenge-types/#dns-01-challenge

    ACME-SCOPED-CHALLENGE:
        title: "ACME Scoped DNS Challenges"
        date: 2024
        author:
          - ins: A. A. Chariton
          - ins: A. A. Omidi
          - ins: J. Kasten
          - ins: F. Loukos
          - ins: S. A. Janikowski
        target: https://datatracker.ietf.org/doc/draft-ietf-acme-scoped-dns-challenges/

    LETSENCRYPT-90-DAYS-RENEWAL:
        title: "Why ninety-day lifetimes for certificates?"
        date: 2015
        author:
          - ins: Let's Encrypt
        target: https://letsencrypt.org/2015/11/09/why-90-days.html

    GOOGLE-WORKSPACE-TXT:
        title: "TXT record values"
        author:
          - ins: Google
        target: https://support.google.com/a/answer/2716802

    ATPROTO-TXT:
        title: "DNS TXT Method"
        author:
          - ins: Bluesky
        target: https://atproto.com/specs/handle#dns-txt-method

    CLOUDFLARE-DELEGATED:
        title: "Auto-renew TLS certificates with DCV Delegation"
        date: 2023
        author:
          - ins: Cloudflare
        target: https://blog.cloudflare.com/introducing-dcv-delegation/

    AKAMAI-DELEGATED:
        title: "Onboard a secure by default property"
        date: 2023
        author:
          - ins: Akamai Technologies
        target: https://techdocs.akamai.com/property-mgr/reference/onboard-a-secure-by-default-property

    GOOGLE-WORKSPACE-CNAME:
        title: "CNAME record values"
        author:
          - ins: Google
        target: https://support.google.com/a/answer/112038

    DOCUSIGN-CNAME:
        title: "Claim a Domain"
        author:
          - ins: DocuSign Admin for Organization Management
        target: https://support.docusign.com/s/document-item?rsc_301=&bundleId=rrf1583359212854&topicId=gso1583359141256_1.html

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

    SUBDOMAIN-TAKEOVER:
        title: "Subdomain takeovers"
        author:
          - ins: Mozilla
        target: https://developer.mozilla.org/en-US/docs/Web/Security/Subdomain_takeovers

    UNDERSCORE-REGISTRY:
        title: "Underscored and Globally Scoped DNS Node Name"
        author:
          - ins: IANA
        target: https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#underscored-globally-scoped-dns-node-names


--- abstract

Many application services on the Internet need to verify ownership or control of a domain in the Domain Name System (DNS). The general term for this process is "Domain Control Validation", and can be done using a variety of methods such as email, HTTP/HTTPS, or the DNS itself. This document focuses only on DNS-based methods, which typically involve the Application Service Provider requesting a DNS record with a specific format and content to be visible in the domain to be verified. There is wide variation in the details of these methods today. This document provides some best practices to avoid known problems.

--- middle



# Introduction

Many Application Service Providers of internet services need domain owners to prove that they control a particular DNS domain before the Application Service Provider can operate services for or grant some privilege to that domain. For instance, Certification Authorities (CAs) ask requesters of TLS certificates to prove that they operate the domain they are requesting the certificate for. Application Service Providers generally allow for several different ways of proving control of a domain. In practice, DNS-based methods take the form of the Application Service Provider generating a random token and asking the requester to create a DNS record containing this random token and placing it at a location within the domain that the Application Service Provider can query for. Generally only one time-bound DNS record is sufficient for proving domain ownership.

This document describes pitfalls associated with some common practices using DNS-based techniques deployed today, and recommends using TXT based domain control validation in a way that is time-bounded and targeted to the service. The {{appendix}} includes a more detailed survey of different methods used by a set of Application Service Providers.

Other techniques such as email or HTTP(S) based validation are out-of-scope.

# Conventions and Definitions

{::boilerplate bcp14}

* `Application Service Provider`: an internet-based provider of a service, for e.g., a Certification Authority or a service that allows for user-controlled websites. These services often require a User to verify that they control a domain. The Application Service Provider may be implementing a standard protocol for domain validation (such as {{RFC8555}}) or they may have their own specification.

* `Intermediary`: an internet-based service that leverages the services of other providers on behalf of a User. For example, an Intermediary might be a service that allows for User-controlled websites and in-turn needs to use a Certification Authority provider to get TLS certificates for the User on behalf of the website.

* `Validation Record`: the DNS record that is used to prove ownership of a domain name ({{RFC9499}}). It typically contains an unguessable value generated by the Application Service Provider which serves as a challenge. The Application Service Provider looks for the Validation Record in the zone of the domain name being verified and checks if it contains the unguessable value.

* `User`: the owner or operator of a domain in the DNS who needs to prove ownership of that domain to an Application Service Provider.

* `Random Token`: a random value that uniquely identifies the DNS domain control validation challenge, defined in {{random-token}}.

# Common Pitfalls {#pitfalls}

A very common but unfortunate technique in use today is to employ a DNS TXT record and placing it at the exact domain name whose control is being validated (e.g., often the zone apex). This has a number of known operational issues. If the User has multiple application services employing this technique, it will end up with multiple DNS TXT records having the same owner name; one record for each of the services.

Since DNS resource record sets are treated atomically, a query for the Validation Record will return all TXT records in the response. There is no way for the verifier to specifically query only the TXT record that is pertinent to their application service. The verifier must obtain the aggregate response and search through it to find the specific record it is interested in.

Additionally, placing many such TXT records at the same name increases the size of the DNS response. If the size of the UDP response (UDP being the most common DNS transport today) is large enough that it does not fit into the Path MTU of the network path, this may result in IP fragmentation, which often does not work reliably on the Internet today due to firewalls and middleboxes, and also is vulnerable to various attacks ([AVOID-FRAGMENTATION]). Depending on message size limits configured or being negotiated, it may alternatively cause the DNS server to "truncate" the UDP response and force the DNS client to re-try the query over TCP in order to get the full response. Not all networks properly transport DNS over TCP and some DNS software mistakenly believe TCP support is optional ([RFC9210]).

Other possible issues may occur. If a TXT record (or any other record type) is designed to be placed at the same domain name that is being validated, it may not be possible to do so if that name already has a CNAME record. This is because CNAME records cannot co-exist with other (non-DNSSEC) records at the same name. This situation cannot occur at the apex of a DNS zone, but can at a name deeper within the zone.

When multiple distinct services create domain Validation Records with the same owner name, there is no way to delegate an application specific domain Validation Record to a third party. Furthermore, even without delegation, an organization may have a shared DNS zone where they need to provide record level permissions to the specific division within the organization that is responsible for the application in question. This can't be done if all applications expect to find validation records at the same name.

The presence of a Validation Record with a predictable domain name (either as a TXT record for the exact domain name where control is being validated or with a well-known label) can allow attackers to enumerate the utilized set of Application Service Providers.

This specification proposes the use of application-specific labels in the owner name of a Validation Record to address these issues.


# Scope of Validation {#scope}

For security reasons, it is crucial to understand the scope of the domain name being validated. Both Application Service Providers and the User need to clearly specify and understand whether the validation request is for a single hostname, a wildcard (all hostnames immediately under that domain), or for the entire domain and subdomains rooted at that name. This is particularly important in large multi-tenant enterprises, where an individual deployer of a service may not necessarily have operational authority of an entire domain.

In the case of X.509 certificate issuance, the certificate signing request and associated challenge are clear about whether they are for a single host or a wildcard domain. Unfortunately, the ACME protocol's DNS-01 challenge mechanism ({{RFC8555, Section 8.4}}) does not differentiate these cases in the DNS Validation Record. In the absence of this distinction, the DNS administrator tasked with deploying the Validation Record may need to explicitly confirm the details of the certificate issuance request to make sure the certificate is not given broader authority than the User intended.  (The ACME protocol is addressing this in {{ACME-SCOPED-CHALLENGE}}.)

In the more general case of an Internet application service granting authority to a domain owner, again no existing DNS challenge scheme makes this distinction today. New applications should consider having different application names for different scopes, as described below in {{scope-indication}}. Regardless, services should very clearly indicate the scope of the validation in their public documentation so that the domain administrator can use this information to assess whether the Validation Record is granting the appropriately scoped authority.

## Domain Boundaries {#domain-boundaries}

The hierarchical structure of domain names do not necessarily define boundaries of ownership and administrative control (e.g., as discussed in {{I-D.draft-tjw-dbound2-problem-statement}}). Some domain names are "public suffixes" ({{RFC9499}}) where care may need to be taken when validating control. For example, there are security risks if an Application Service Provider can be tricked into believing that an attacker has control over ".co.uk" or ".com". The volunteer-managed Public Suffix List {{PSL}} is one mechanism available today that can be useful for identifying public suffixes.

Future specifications may provide better mechanisms or recommendations for defining domain boundaries or for enabling organizational administrators to place constraints on domains and subdomains. See {{constraint-examples}} for cases where DNS records can be used as constraints complementary to domain verification.


# Recommendations {#recommendations}

All Domain Control Validation mechanisms are implemented by a resource record with:

1) An owner name related to the domain name being validated
2) One or more random tokens, to be placed in either the validation record's RDATA, in the target of a CNAME (or chain of CNAMEs), or as a label of the owner name.

Both of these are issued to the User by either an Application Service Provider or an Intermediary. An issued random token then needs to exist in at least one of these to demonstrate the User has control over the domain name being validated. Variations on this approach exist to meet different uses.

## Random Token {#random-token}

A unique token used in the challenge. It should be a random value issued between parties (Application Service Provider to User, Application Service Provider to Intermediary, or Intermediary to User) with the following properties:

1. MUST have at least 128 bits of entropy.
2. base64url ({{!RFC4648, Section 5}}) encoded, base32 ({{!RFC4648, Section 6}}) encoded, or base16 ({{!RFC4648, Section 8}}) encoded.

See {{RFC4086}} for additional information on randomness requirements.

Base32 encoding or hexadecimal base16 encoding are RECOMMENDED to be specified when the random token would exist in a DNS label such as in a CNAME target.  This is because base64 relies on mixed case (and DNS is case-insensitive as clarified in {{RFC4343}}) and because some base64 characters ("/", "+", and "=") may not be permitted by implementations that limit allowed characters to those allowed in hostnames.  If base32 is used, it SHOULD be specified in way that safely omits the trailing padding ("=").  Note that DNS labels are limited to 63 octets which limits how large such a token may be.

This random token is placed in either the RDATA or an owner name, as described in the rest of this section.  Some methods of validation may involve multiple independent random tokens.

## Validation Record Owner Name {#name}

The RECOMMENDED format for a Validation Record's owner name is as application-specific underscore prefix labels. Domain Control Validation Records are constructed by the Application Service Provider by prepending the label "`_<PROVIDER_RELEVANT_NAME>-challenge`" to the domain name being validated (e.g. "\_foo-challenge.example.com"). The prefix "_" is used to avoid collisions with existing hostnames.

If an Application Service Provider has an application-specific need to have multiple validations for the same label, multiple prefixes can be used, such as "`_<FEATURE>._<PROVIDER_RELEVANT_NAME>-challenge`".

An Application Service Provider may also specify prepending a random token to the owner name of a validation record, such as "`<RANDOM_TOKEN>._<PROVIDER_RELEVANT_NAME>-challenge`". This can be done either as part of the challenge itself ({{cname-dcv}}, to support multiple Intermediaries ({{multiple}}), or to make it harder for a third party to scan what Application Service Providers are being used by a given domain name.

### Scope Indication {#scope-indication}

For applications that may apply more broadly than to a single hostname, the RECOMMENDED approach is to differentiate the application-specific underscore prefix labels to also include the scope (see {{scope}}). In particular:

* "`_<PROVIDER_RELEVANT_NAME>-host-challenge.example.com`" applies only to the specific hostname of "example.com" and not to anything underneath it.
* "`_<PROVIDER_RELEVANT_NAME>-wildcard-challenge.example.com`" applies to all hostnames at the level immediately underneath "example.com". For example, it would apply to "foo.example.com" but not "example.com" nor "quux.bar.example.com"
* "`_<PROVIDER_RELEVANT_NAME>-domain-challenge.example.com`" applies to the entire domain "example.com" as well as its subdomains. For example, it would apply to all of "example.com", "foo.example.com", and "quux.bar.example.com"

The Application Service Provider will normally know which of these scoped DNS records to query based on the User's requested configuration, so this does not typically result in multiple queries for different possible scopes. If discovery of scope is needed for a specific application as part of the domain control validation process, then the scope could alternatively be encoded in a key value pair in the record data.

Note that the ACME DNS challenge specification {{ACME-SCOPED-CHALLENGE}} has incorporated this scope indication format.

Application owners SHOULD utilize the IANA "Underscored and Globally Scoped DNS Node Names" registry {{UNDERSCORE-REGISTRY}} to ensure that there are no collisions with existing entries.

### CNAME Considerations {#cname-considerations}

Any Validation Records that might include a CNAME MUST have a name that is distinct from the domain name being validated, as a CNAME MUST NOT be placed at the same domain name that is being validated.  The recommended format in {{name}} as well as others below all have this property.

This is for the same reason already cited in {{pitfalls}}. CNAME records cannot co-exist with other (non-DNSSEC) data, and there may already be other record types that exist at the domain name. Instead, as with the TXT record recommendation, an Application Service Provider specific label should be added as a subdomain of the domain to be verified. This ensures that the CNAME does not collide with other record types.

Note that some DNS implementations permit the deployment of CNAME records co-existing with other record types. These implementations are in violation of the DNS protocol. Furthermore, they can cause resolution failures in unpredictable ways depending on the behavior of DNS resolvers, the order in which query types for the name are processed, etc. In short, they cannot work reliably and these implementations should be fixed.

## TXT Record {#txt-record}

The RECOMMENDED method of doing DNS-based domain control validation is to use DNS TXT records as the Validation Record. The name is constructed as described in {{name}}, and RDATA MUST contain at least a Random Token (constructed as in {{random-token}}). If metadata (see {{metadata}}) is not used, then the unique token generated as-above can be placed as the only contents of the RDATA. For example:

    _foo-challenge.example.com.  IN   TXT  "3419...3d206c4"

This again allows the Application Service Provider to query only for application-specific records it needs, while giving flexibility to the User adding the DNS record (i.e., they can be given permission to only add records under a specific prefix by the DNS administrator). Whether or not multiple Validation Records can exist for the same domain is up to the Application Service Provider's application specification.

Application Service Providers MUST validate that a random token in the TXT record matches the one that they gave to the User for that specific domain name.

### Token Metadata {#metadata}

It may be desirable to associate metadata with the token in a Validation Record. When specified, metadata SHOULD be encoded in the RDATA via space-separated ASCII key-value pairs {{RFC1464}}, with the key "token" prefixing the random token. For example:

    _foo-challenge.example.com.  IN   TXT  "token=3419...3d206c4"

If there are multiple tokens required, each one MUST be in a separate RR to allow them to match up with any additional attributes.  For example:

    _foo-challenge.example.com.  IN   TXT  "token=3419...3d206c4 attr=bar"
                                 IN   TXT  "token=5454...45dc45a attr=quux"

The token MUST be the first element in the key-value list. If the TXT record RDATA is not prefixed with `token=` then {{RFC1464}} encoding MUST NOT be assumed (as this might split the trailing "==" or "=" at the end of base64 encoding).

If an alternate syntax is used by the Application Service Provider for token metadata, they MUST specify a grammar for it.

### Metadata For Expiry {#expiry-metadata}

Application Service Providers MUST provide clear instructions on when a Validation Record can be removed.

These instructions SHOULD be encoded in the RDATA as token metadata ({{metadata}} using the key "expiry" to hold a time after which it is safe to remove the Validation Record. For example:

    _foo-challenge.example.com.  IN   TXT  "token=3419...3d206c4 expiry=2023-02-08T02:03:19+00:00"

When an expiry time is specified, the value of "expiry" SHALL be in ISO 8601 format as specified in {{!RFC3339, Section 5.6}}.

A simpler variation of the expiry time is also ISO 8601 valid and can also be specified, using the "full-date" format. For example:

    _foo-challenge.example.com.  IN   TXT  "token=3419...3d206c4 expiry=2023-02-08"

Alternatively, if the record should never expire (for instance, if it may be checked periodically by the Application Service Provider) and should not be removed, the key "expiry" SHALL be set to have value "never".

    _foo-challenge.example.com.  IN   TXT  "token=3419...3d206c4 expiry=never"

The "expiry" key MAY be omitted in cases where the Application Service Provider has clarified the record expiry policy out-of-band ({{github}}).

    _foo-challenge.example.com.  IN   TXT  "token=3419...3d206c4"

Note that this is semantically the same as:

    _foo-challenge.example.com.  IN   TXT  "3419...3d206c4"

The User SHOULD de-provision the resource record provisioned for DNS-based domain control validation once it is no longer required.

## Delegated Domain Control Validation {#delegated}

Delegated domain control validation lets a User delegate the domain control validation process for their domain to an Intermediary without having to hand over full DNS access.  It is a variation of the above TXT record validation ({{txt-record}}) that indirectly inserts a CNAME record prior to the TXT record.

The Intermediary gives the User a CNAME record to add for the domain and Application Service Provider being validated that points to the Intermediary's domain, where the actual validation TXT record is placed. The record name and base16-encoded (or base32-encoded) random tokens are generated as in {{random-token}}. For example:

    _foo-challenge.example.com.  IN   CNAME  <intermediary-random-token>.dcv.intermediary.example.

The Intermediary then adds the actual Validation Record in a domain they control:

    <intermediary-random-token>.dcv.intermediary.example.  IN   TXT "<provider-random-token>"

Such a setup is especially useful when the Application Service Provider wants to periodically re-issue the challenge with a new provider random token. CNAMEs allow automating the renewal process by letting the Intermediary place the random token in their DNS instead of needing continuous write access to the User's DNS.

Importantly, the CNAME record target also contains a random token issued by the Intermediary to the User (preferably over a secure channel) which proves to the Intermediary that example.com is controlled by the User. The Intermediary must keep an association of Users and domain names to the associated Intermediary-random-tokens. Without a linkage validated by the Intermediary during provisioning and renewal there is the risk that an attacker could leverage a "dangling CNAME" to perform a "subdomain takeover" attack ({{SUBDOMAIN-TAKEOVER}}).

When a User stops using the Intermediary they should remove the domain control validation CNAME in addition to any other records they have associated with the Intermediary.

See {{delegated-examples}} for examples.

## Domain Control Validation Supporting Multiple Intermediaries {#multiple}

There are use-cases where a User may wish to simultaneously use multiple intermediaries or multiple independent accounts with an Application Service Provider. For example, a hostname may be using a "multi-CDN" where the hostname simultaneously uses multiple Content Delivery Network (CDN) providers.

To support this, Application Service Providers may support prefixing the challenge with a label containing an unique account identifier of the form `_<identifier-token>` and following the requirements of {{random-token}}, specified as either base32 or base16 encoded. This identifier token should be stable over time and would be provided to the User by the Application Service Provider, or by an Intermediary in the case where domain validation is delegated ({{delegated}}).

The resulting record could either directly contain a TXT record or a CNAME (as in {{delegated}}).  For example:

    _<identifier-token>._foo-challenge.example.com.  IN   TXT  "3419...3d206c4"

or

    _<identifier-token>._foo-challenge.example.com.  IN   CNAME  <intermediary-random-token>.dcv.intermediary.example.

When performing validation, the Application Service Provider would resolve the DNS name containing the appropriate identifier token.

Application Service Providers may wish to always prepend the `_<identifier-token>` to make it harder for third parties to scan, even absent supporting multiple intermediaries.

## Specification of Validation Records

Validation Records need to be securely relayed from an Application Service Provider to a DNS administrator. Application Service Providers and intermediaries SHOULD offer detailed and easily-accessible help pages, keeping in mind that the DNS administrator might not have a login account on the website of the Application Service Provider or Intermediary. Similarly, for clarity, the entire DNS resource record (RR) using the Fully Qualified Domain Name to be added SHOULD be provided along with help instructions.  Where possible, APIs SHOULD be used to relay instructions.

## Time-bound checking

After domain control validation is completed, there is typically no need for the TXT or CNAME record to continue to exist as the presence of the domain validation DNS record for a service only implies that a User with access to the service also has DNS control of the domain at the time the code was generated. It should be safe to remove the validation DNS record once the validation is done and the Application Service Provider doing the validation should specify how long the validation will take (i.e., after how much time can the validation DNS record be deleted).

Some Application Service Providers currently require the Validation Record to remain in the zone indefinitely for periodic revalidation purposes. This practice should be discouraged. Subsequent validation actions using an already disclosed token are no guarantee that the original owner is still in control of the domain, and a new challenge needs to be issued.

One exception is if the record is being used as part of a delegated domain control validation setup ({{delegated}}); in that case, the CNAME record that points to the actual validation TXT record cannot be removed as long as the User is still relying on the Intermediary.

## TTL Considerations

The TTL {{RFC1034}} for Validation Records SHOULD be short to allow recovering from potential misconfigurations. These records will not be polled frequently so caching or resolver load will not be an issue.

The Application Service Provider looking up a Validation Record may have to wait for up to the SOA minimum TTL (negative caching TTL) of the enclosing zone for the record to become visible, if it has been previously queried. If the application User wants to make the Validation Record visible more quickly they may need to work with the DNS administrator to see if they are willing to lower the SOA minimum TTL (which has implications across the entire zone).

Application Service Provider's verifiers MAY wish to either use dedicated DNS resolvers configured with a low maximum negative caching TTL or flush Validation Records from resolver caches prior to issuing queries.

## CNAME Records for Domain Control Validation {#cname-dcv}

CNAME records MAY be used instead of TXT records where specified by Application Service Providers to support Users who are unable to create TXT records. Two forms of this are common: including the challenge token in the owner name of a validation record, or including the challenge token as a part of the CNAME target. This approach has a number of limitatations relative to using TXT records.

### Random Token in Domain Names

Application Service Providers MAY specify that a random token be included in the owner name of a validation record.  In this case an underscore-prefixed label MUST be used (e.g., `_<token>._foo` or `_foo-<token>`). The resource record is then a CNAME to a domain name specified by the Application Service Provider. The Application Service Provider uses the presence of a resource record at the CNAME target to perform the validation, validating the both presence of the record as well as the CNAME target. For example:

    _<random-token>._foo-challenge.example.com.  IN   CNAME dcv.provider.example.

In practice, many Application Service Providers that employ CNAMEs for domain control validation today use an entirely random subdomain label which works to avoid accidential collisions, but which could allow for a malicious Application Service Provider to smuggle instructions from some other Application Service Provider. Adding an provider-specific component in addition (such as `_<token>._foo-challenge` or `_foo-<token>-challenge`) make it easier for the domain owner to keep track of why and for what service a Validation Record has been deployed.

Since the random token exists entirely in the challenge, it is not possible to delegate Domain Control Validation challenges of this form to Intermediaries in a way that allows the Intermediary to refresh the challenge over time.

### Random Token in CNAME Targets

An Application Service Provider MAY specify using CNAME records instead of TXT records for Domain Control Validation. In this case, the target of the CNAME would contain the base16-encoded (or base32-encoded) random token followed by a suffix specified by the Application Service Provider. For example:

    _foo-challenge.example.com.  IN   CNAME <random-token>.dcv.provider.example.

The Application Service Provider then validates that the target of the CNAME matches the token provided. This approach has similar properties to TXT records ({{txt-record}}) but does not allow for additional attributes such as expiry to be added.

As mentioned in {{cname-considerations}}, the owner name of the Validation Record MUST be distinct from the domain name being validated.


## Interactions with DNAME

Domain control validation in the presence of a DNAME {{RFC6672}} is theoretically possible. Since a DNAME record redirects the entire subtree of names underneath the owner of the DNAME, it is not possible to place a Validation Record under the DNAME owner itself. It would have to be placed under the DNAME target name, since any lookups for a name under the DNAME owner will be redirected to the corresponding name under the DNAME target.


# Security Considerations

## Token Guessing

If token values aren't long enough or lack adequate entropy there's a risk that a malicious actor could produce a token that could be confused with an application-specific underscore prefix label.

## Service Confusion {#service-confusion}

A malicious Application Service Provider that promises to deliver something after domain control validation could surreptitiously ask another Application Service Provider to start processing or sending mail for the target domain and then present the victim User with this DNS TXT record pretending to be for their service. Once the User has added the DNS TXT record, instead of getting their service, their domain is now certifying another service of which they are not aware they are now a consumer. If services use a clear description and name attribution in the required DNS TXT record, this can be avoided. For example, by requiring a DNS TXT record at \_vendorname.example.com instead of at example.com, a malicious service could no longer forward a challenge from a different service without the User noticing. Both the Application Service Provider and the service being authenticated and authorized should be unambiguous from the Validation Record to prevent malicious services from misleading the domain owner into certifying a different provider or service.

## Service Collision

As a corollary to {{service-confusion}}, if the Validation Record is not well-scoped and unambiguous with respect to the Application Service Provider, it could be used to authorize use of another Application Service Provider or service in addition to the original Application Service Provider or service.

## Scope Confusion

Ambiguity of scope introduces risks, as described in {{scope}}. Distinguishing the scope in the application-specific label, along with good documentation, should help make it clear to DNS administrators whether the record applies to a single hostname, a wildcard, or an entire domain. Always using this indication rather than having a default scope reduces ambiguity, especially for protocols that may have used a shared application-specific label for different scopes in the past. While it would also have been possible to include the scope in as an attribute in the TXT record, that has more potential for ambiguity and misleading an operator, such as if an implementation ignores attribute it doesn't recognize but an attacker includes the attribute to mislead the DNS administrator.

## Authenticated Channels

Application Service Providers and intermediaries should use authenticated channels to convey instructions and random tokens to Users. Otherwise, an attacker in the middle could alter the instructions, potentially allowing the attacker to provision the service instead of the User.

## DNS Spoofing

A domain owner SHOULD sign their DNS zone using DNSSEC {{RFC9364}} to protect Validation Records against DNS spoofing attacks.

## DNSSEC Validation

DNSSEC validation SHOULD be performed by Application Service Providers that verify Validation Records they have requested to be deployed.  If no DNSSEC support is detected for the domain being validated, or if DNSSEC validation cannot be performed, Application Service Providers SHOULD attempt to query and confirm the Validation Record by matching responses from multiple DNS resolvers on unpredictable geographically diverse IP addresses to reduce an attacker's ability to complete a challenge by spoofing DNS. Alternatively, Application Service Providers MAY perform multiple queries spread out over a longer time period to reduce the chance of receiving spoofed DNS answers.

## Public Suffixes {#public-suffixes}

As discussed above in {{domain-boundaries}}, there are risks in allowing control to be demonstrated over domains which are "public suffixes" (such as ".co.uk" or ".com"). The volunteer-managed Public Suffix List ({{PSL}}) is one mechanism that can be used. It includes two "divisions" ({{PSL-DIVISIONS}}) covering both registry-owned public suffixes (the "ICANN" division) and a "PRIVATE" division covering domains submitted by the domain owner.

Operators of domains which are in the "PRIVATE" public suffix division often provide multi-tenant services such as dynamic DNS, web hosting, and CDN services. As such, they sometimes allow their sub-tenants to provision names as subdomains of their public suffix. There are use-cases that require operators of domains in the public suffix list to demonstrate control over their domain, such as to be added to the Public Suffix List ({{psl-example}}) or to provision a wildcard certificate. At the same time, if an operator of such a domain allows its customers or tenants to create names starting with an underscore ("_") then it opens up substantial risk to the domain operator for attackers to provision services on their domain.

Whether or not it is appropriate to allow domain verification on a public suffix will depend on the application.  In the general case:

* Application Service Providers SHOULD NOT allow verification of ownership for domains which are public suffixes in the "ICANN" division. For example, "\_foo-challenge.co.uk" would not be allowed.
* Application Service Providers MAY allow verification of ownership for domains which are public suffixes in the "PRIVATE" division, although it would be preferable to apply additional safety checks in this case.




# IANA Considerations

This document has no IANA actions.


--- back

# Appendix {#appendix}

A survey of several different methods deployed today for DNS based domain control validation follows.

## Survey of Techniques

### TXT based {#txt-based}

A TXT record is usually the default option for domain control validation. The Application Service Provider asks the User to add a DNS TXT record (perhaps through their domain host or DNS provider) at the domain with a certain value. Then the Application Service Provider does a DNS TXT query for the domain being verified and checks that the correct value is present. For example, this is what a DNS TXT record could look like for an Application Service Provider Foo:

    example.com.   IN   TXT   "237943648324687364"

Here, the value "237943648324687364" serves as the randomly-generated TXT value being added to prove ownership of the domain to Foo Application Service Provider. Note that in this construction Application Service Provider Foo would have to query for all TXT records at "example.com" to get the Validation Record. Although the original DNS protocol specifications did not associate any semantics with the DNS TXT record, {{RFC1464}} describes how to use them to store attributes in the form of ASCII text key-value pairs for a particular domain. In practice, there is wide variation in the content of DNS TXT records used for domain control validation, and they often do not follow the key-value pair model. Even so, the RDATA {{RFC1034}} portion of the DNS TXT record has to contain the value being used to verify the domain. The value is usually a Random Token in order to guarantee that the entity who requested that the domain be verified (i.e., the person managing the account at Application Service Provider Foo) is the one who has (direct or delegated) access to DNS records for the domain. After a TXT record has been added, the Application Service Provider will usually take some time to verify that the DNS TXT record with the expected token exists for the domain. The generated token typically expires in a few days.

Some Application Service Providers use a prefix of `_PROVIDER_NAME-challenge` in the Name field of the TXT record challenge. For ACME, the full Host is `_acme-challenge.<YOUR_DOMAIN>`. Such patterns are useful for doing targeted domain control validation. The ACME protocol ([RFC8555]) has a challenge type `DNS-01` that lets a User prove domain ownership. In this challenge, an implementing CA asks you to create a TXT record with a randomly-generated token at `_acme-challenge.<YOUR_DOMAIN>`:

    _acme-challenge.example.com.  IN  TXT "cE3A8qQpEzAIYq-T9DWNdLJ1_YRXamdxcjGTbzrOH5L"

{{RFC8555}} (section 8.4) places requirements on the Random Token.

#### Let's Encrypt

The ACME example in {{txt-based}} is implemented by Let's Encrypt {{DNS-01}}.

#### Google Workspace

{{GOOGLE-WORKSPACE-TXT}} asks the User to sign in with their administrative account and obtain their token as part of the setup process for Google Workspace. The verification token is a 68-character string that begins with "google-site-verification=", followed by 43 characters. Google recommends a TTL of 3600 seconds. The owner name of the TXT record is the domain or subdomain name being verified.

#### The AT Protocol

The Authenticated Transfer (AT) Protocol supports DNS TXT records for resolving social media "handles" (human-readable identifiers) to the User's persistent account identifier {{ATPROTO-TXT}}. For example, this is how the handle `bsky.app` would be resolved:

    _atproto.bsky.app.  IN  TXT "did=did:plc:z72i7hdynmk6r22z27h6tvur"

#### GitHub {#github}

To verify domains for organizations, GitHub asks the user to create a DNS TXT record under `_github-challenge-ORGANIZATION.<YOUR_DOMAIN>`, where ORGANIZATION stands for the GitHub organization name. The RDATA value for the provided TXT record is a string that expires in 7 days {{GITHUB-TXT}}.

#### Public Suffix List {#psl-example}

The Public Suffix List ({{PSL}}) asks for owners of private domains to authenticate by creating a TXT record containing the pull request URL for adding the domain to the Public Suffix List.  For example, to authenticate "example.com" submitted under pull request 100, a requestor would add:

    _psl.example.com.  IN TXT "https://github.com/publicsuffix/list/pull/100"

### CNAME based {#cname-examples}

#### CNAME for Domain Control Validation {#cname-dcv-examples}

##### DocuSign

{{DOCUSIGN-CNAME}} asks the User to add a CNAME record with the "Host Name" set to be a 32-digit random value pointing to `verifydomain.docusign.net.`.

##### Google Workspace

{{GOOGLE-WORKSPACE-CNAME}} lets you specify a CNAME record for verifying domain ownership. The User gets a unique 12-character string that is added as "Host", with TTL 3600 (or default) and Destination an 86-character string beginning with "gv-" and ending with ".domainverify.googlehosted.com.".

#### Delegated Domain Control Validation {#delegated-examples}

##### Content Delivery Networks (CDNs): Akamai and Cloudflare

In order to be issued a TLS cert from a Certification Authority like Let's Encrypt, the requester needs to prove that they control the domain. Often this is done via the {{DNS-01}} challenge. Let's Encrypt only issues certs with a 90 day validity period for security reasons {{LETSENCRYPT-90-DAYS-RENEWAL}}. This means that after 90 days, the DNS-01 challenge has to be re-done and the random token has to be replaced with a new one. Doing this manually is error-prone. Content Delivery Networks like Akamai and Cloudflare offer to automate this process using a CNAME record in the User's DNS that points to the Validation Record in the CDN's zone ({{AKAMAI-DELEGATED}} and {{CLOUDFLARE-DELEGATED}}).

##### AWS Certificate Manager (ACM)

AWS Certificate Manager {{ACM-CNAME}} allows delegated domain control validation {{delegated}}. The record name for the CNAME looks like:

     _<random-token1>.example.com.  IN   CNAME _<random-token2>.acm-validations.aws.

The CNAME points to:

     _<random-token2>.acm-validations.aws.  IN   TXT "<random-token3>"

Here, the random tokens are used for the following:

* `<random-token1>`: Unique sub-domain, so there's no clashes when looking up the Validation Record.
* `<random-token2>`: Proves to ACM that the requester controls the DNS for the requested domain at the time the CNAME is created.
* `<random-token3>`: The actual token being verified.

Note that if there are more than 5 CNAMEs being chained, then this method does not work.

#### Atlassian

Some services ask the DNS record to exist in perpetuity {{ATLASSIAN-VERIFY}}. If the record is removed, the User gets a limited amount of time to re-add it before they lose domain validation status.

#### Constraints on Domains and Subdomains {#constraint-examples}

##### CAA records

While the ACME protocol ([RFC8555]) specifies a way to demonstrate ownership over a given domain, Certification Authorities are required to use it in-conjunction with {{RFC8659}} that specifies CAA records. CAA allows a domain owner to apply policy across a domain and its subdomains to limit which Certification Authorities may issue certificates.

# Acknowledgments

Thank you to Tim Wicinski, John Levine, Daniel Kahn Gillmor, Amir Omidi, Tuomo Soini, Ben Kaduk and many others for their feedback and suggestions on this document.
