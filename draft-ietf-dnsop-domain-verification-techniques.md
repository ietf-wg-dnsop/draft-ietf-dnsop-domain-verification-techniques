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

 -
    ins: T. Wicinski
    name: Tim Wicinski
    organization: Cox Communications
    email: tjw.ietf@gmail.com

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
    RFC9499:
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

    ACME-DNS-ACCOUNT-ID:
        title: "ACME DNS Labeled with Account ID Challenge"
        date: 2024
        author:
          - ins: A. A. Chariton
          - ins: A. A. Omidi
          - ins: J. Kasten
          - ins: F. Loukos
          - ins: S. A. Janikowski
        target: https://datatracker.ietf.org/doc/draft-ietf-acme-dns-account-label/

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

Many Application Service Providers of internet services need domain owners to prove that they control a particular DNS domain before the Application Service Provider can operate services for or grant some privilege to that domain. For instance, Certification Authorities (CAs) ask requesters of TLS certificates to prove that they operate the domain they are requesting the certificate for. Application Service Providers generally allow for several different ways of proving control of a domain. In practice, DNS-based methods take the form of the Application Service Provider generating a random token and asking the requester to publish a DNS record containing this random token at a specified name within the domain. The Application Service Provider can verify operational control of the domain by resolving this DNS record.

This document recommends using TXT based domain control validation in a way that is time-bounded and targeted to the specific application service.

# Conventions and Definitions

{::boilerplate bcp14}

* `Application Service Provider`: an internet-based provider of a service, for e.g., a Certification Authority or a service that allows for user-controlled websites. These services often require a User to verify that they control a domain. The Application Service Provider may be implementing a standard protocol for domain validation (such as {{RFC8555}}) or they may have their own specification.

* `Intermediary`: an internet-based service that leverages the services of other providers on behalf of a User. For example, an Intermediary might be a service that allows for User-controlled websites and in-turn needs to use a Certification Authority provider to get TLS certificates for the User on behalf of the website.

* `Validation Record`: the DNS record that is used to prove ownership of a domain name ({{RFC9499}}). It typically contains an unguessable value generated by the Application Service Provider which serves as a challenge. The Application Service Provider looks for the Validation Record in the zone of the domain name being verified and checks if it contains the unguessable value.

* `DNS Administrator`: the owner or responsible party for the contents of a domain in the DNS.

* `User`: the owner or operator of a domain in the DNS who needs to prove ownership of that domain to an Application Service Provider, working in coordination with their DNS Administrator.

* `Random Token`: a random value that uniquely identifies the DNS domain control validation challenge, defined in {{random-token}}.

# Purpose of Domain Control Validation {#purpose}

Domain Control Validation is a challenge-response procedure that allows the User to prove operational control of the contents of a domain to an Application Service Provider. This proof applies only to some point in time after the challenge is issued and before the record is published. This procedure can be appropriate when the Application Service Provider is about to take a time-bounded action related to this domain name.

Domain Control Validation is not suitable for ongoing authorization. Any ongoing authorization using DNS SHOULD be performed with a separate DNS record. For example, in the ACME protocol, issuance of a time-limited certificate can use Domain Control Validation with an ephemeral TXT record via the DNS-01 challenge mechanism ({{RFC8555, Section 8.4}}), but ongoing authorization of certificate authorities uses a persistent CAA record {{?RFC8569}}.

# Scope of Validation {#scope}

For security reasons, it is crucial to understand the scope of the domain name being validated. Both Application Service Providers and the User need to clearly specify and understand whether the validation request is for a single hostname, a wildcard (all hostnames immediately under that domain), or for the entire domain and subdomains rooted at that name. This is particularly important in large multi-tenant enterprises, where an individual deployer of a service may not necessarily have operational authority of an entire domain.

In the case of X.509 certificate issuance, the certificate signing request and associated challenge are clear about whether they are for a single host or a wildcard domain. Unfortunately, the ACME protocol's DNS-01 challenge mechanism ({{RFC8555, Section 8.4}}) does not differentiate these cases in the DNS Validation Record. In the absence of this distinction, the DNS administrator tasked with deploying the Validation Record may need to explicitly confirm the details of the certificate issuance request to make sure the certificate is not given broader authority than the User intended.

In the more general case of an Internet application service granting authority to a domain owner, again no existing DNS challenge scheme makes this distinction today. New applications should consider having different application names for different scopes, as described. Regardless, services should very clearly indicate the scope of the validation in their public documentation so that the domain administrator can use this information to assess whether the Validation Record is granting the appropriately scoped authority.


# Recommendations {#recommendations}

All Domain Control Validation mechanisms are implemented by a DNS resource record with at least the following information:

1. A record name related to the domain name being validated, usually constructed by prepending an application specific label.

2. One or more random tokens.

## TXT Record based Validation {#txt-record}

The RECOMMENDED method of doing DNS-based domain control validation is to use DNS TXT records as the Validation Record. The name is constructed as described in {{name}}, and RDATA MUST contain at least a Random Token (constructed as in {{random-token}}). If there are multiple RDATA strings for a record, the Application Service Provider MUST treat them as a concatenated string. If metadata (see {{metadata}}) is not used, then the unique token generated as-above can be placed as the only contents of the RDATA. For example:

    _service-challenge.example.com.  IN   TXT  "3419...3d206c4"

This again allows the Application Service Provider to query only for application-specific records it needs, while giving flexibility to the User adding the DNS record (i.e., they can be given permission to only add records under a specific prefix by the DNS administrator).

Application Service Providers MUST validate that a random token in the TXT record matches the one that they gave to the User for that specific domain name. Whether or not multiple Validation Records can exist for the same domain is up to the Application Service Provider's application specification. In case there are multiple TXT records for the specific domain name, the Application Service Provider MUST confirm at least one record matches.

### Random Token {#random-token}

A unique token should be used in the challenge. It should be a random value issued between parties (Application Service Provider to User, Application Service Provider to Intermediary, or Intermediary to User) with the following properties:

1. MUST have at least 128 bits of entropy.
2. base64url ({{!RFC4648, Section 5}}) encoded, base32 ({{!RFC4648, Section 6}}) encoded, or base16 ({{!RFC4648, Section 8}}) encoded.

See {{RFC4086}} for additional information on randomness requirements.

Base32 encoding or hexadecimal base16 encoding are RECOMMENDED to be specified when the random token would exist in a DNS label such as in a CNAME target.  This is because base64 relies on mixed case (and DNS is case-insensitive as clarified in {{RFC4343}}) and because some base64 characters ("/", "+", and "=") may not be permitted by implementations that limit allowed characters to those allowed in hostnames.  If base32 is used, it SHOULD be specified in way that safely omits the trailing padding ("=").  Note that DNS labels are limited to 63 octets which limits how large such a token may be.

This random token is placed in either the RDATA or an owner name, as described in the rest of this section.  Some methods of validation may involve multiple independent random tokens.


### Token Metadata {#metadata}

It may be desirable to associate metadata with the token in a Validation Record. When specified, metadata SHOULD be encoded in the RDATA via space-separated ASCII key-value pairs {{RFC1464}}, with the key "token" prefixing the random token. For example:

    _service-challenge.example.com.  IN   TXT  "token=3419...3d206c4"

If there are multiple tokens required, each one MUST be in a separate RR to allow them to match up with any additional attributes.  For example:

    _service-challenge.example.com.  IN   TXT  "token=3419...3d206c4 attr=bar"
                                 IN   TXT  "token=5454...45dc45a attr=quux"

The token MUST be the first element in the key-value list. If the TXT record RDATA is not prefixed with `token=` then {{RFC1464}} encoding MUST NOT be assumed (as this might split the trailing "==" or "=" at the end of base64 encoding).

If an alternate syntax is used by the Application Service Provider for token metadata, they MUST specify a grammar for it.

## Validation Record Owner Name {#name}

The RECOMMENDED format for a Validation Record's owner name is application-specific underscore prefix labels. Domain Control Validation Records are constructed by the Application Service Provider by prepending the label "`_<PROVIDER_RELEVANT_NAME>-challenge`" to the domain name being validated (e.g. "\_service-challenge.example.com"). The prefix "_" is used to avoid collisions with existing hostnames and to prevent the owner name from being a valid hostname.

If an Application Service Provider has an application-specific need to have multiple validations for the same label, multiple prefixes can be used, such as "`_<FEATURE>._<PROVIDER_RELEVANT_NAME>-challenge`".

Application owners SHOULD utilize the IANA "Underscored and Globally Scoped DNS Node Names" registry {{UNDERSCORE-REGISTRY}} and avoid using underscore labels that already exist in the registry.

As a simplification, some applications may decide to omit the "-challenge" suffix and use just "`_<PROVIDER_RELEVANT_NAME>`" as the label.

## Time-bound checking and Expiration

After domain control validation is completed, there is typically no need for the TXT or CNAME record to continue to exist as the presence of the domain validation DNS record for a service only implies that a User with access to the service also has DNS control of the domain at the time the code was generated. It should be safe to remove the validation DNS record once the validation is done and the Application Service Provider doing the validation should specify how long the validation will take (i.e., after how much time can the validation DNS record be deleted).

Some Application Service Providers currently require the Validation Record to remain in the zone indefinitely for periodic revalidation purposes. This practice should be discouraged. Subsequent validation actions using an already disclosed token are no guarantee that the original owner is still in control of the domain, and a new challenge needs to be issued (see {{purpose}}).

One exception is if the record is being used as part of a delegated domain control validation setup ({{delegated}}); in that case, the CNAME record that points to the actual validation TXT record cannot be removed as long as the User is still relying on the Intermediary.

Application Service Providers MUST provide clear instructions on how long the challenge token is valid for, and thus when a Validation Record can be removed.

These instructions MAY be encoded in the RDATA as token metadata ({{metadata}} using the key "expiry" to hold a time after which it is safe to remove the Validation Record. For example:

    _service-challenge.example.com.  IN   TXT  "token=3419...3d206c4 expiry=2023-02-08T02:03:19+00:00"

When an expiry time is specified, the value of "expiry" SHALL be in ISO 8601 format as specified in {{!RFC3339, Section 5.6}}.

A simpler variation of the expiry time is also ISO 8601 valid and can also be specified, using the "full-date" format. For example:

    _service-challenge.example.com.  IN   TXT  "token=3419...3d206c4 expiry=2023-02-08"

Alternatively, if the record should never expire (for instance, if it may be checked periodically by the Application Service Provider) and should not be removed, the key "expiry" SHALL be set to have value "never".

    _service-challenge.example.com.  IN   TXT  "token=3419...3d206c4 expiry=never"

The "expiry" key MAY be omitted in cases where the Application Service Provider has clarified the record expiry policy out-of-band.

    _service-challenge.example.com.  IN   TXT  "token=3419...3d206c4"

Note that this is semantically the same as:

    _service-challenge.example.com.  IN   TXT  "3419...3d206c4"

The User SHOULD de-provision the resource record provisioned for DNS-based domain control validation once it is no longer required.


## TTL Considerations

The TTL {{RFC1034}} for Validation Records SHOULD be short to allow recovering from potential misconfigurations. These records will not be polled frequently so caching or resolver load will not be an issue.

The Application Service Provider looking up a Validation Record may have to wait for up to the SOA minimum TTL (negative caching TTL) of the enclosing zone for the record to become visible, if it has been previously queried. If the application User wants to make the Validation Record visible more quickly they may need to work with the DNS administrator to see if they are willing to lower the SOA minimum TTL (which has implications across the entire zone).

Application Service Providers' verifiers MAY wish to use dedicated DNS resolvers configured with a low maximum negative caching TTL, flush Validation Records from resolver caches prior to issuing queries or just directly query authoritative name servers to avoid caching.

# Delegated Domain Control Validation {#delegated}

Delegated domain control validation lets a User delegate the domain control validation process for their domain to an Intermediary without granting the Intermediary the ability to make changes to their domain or zone configuration.  It is a variation of TXT record validation ({{txt-record}}) that inserts a CNAME record prior to the TXT record.

The Intermediary gives the User a CNAME record to add for the domain and Application Service Provider being validated that points to the Intermediary's domain, where the actual validation TXT record is placed.  The Intermediary's domain MUST be unique to the User, domain, and Application Service Provider. For example, if the User is Example Inc, then the CNAME record could be:

    _service-challenge.example.com.  IN   CNAME  example.com.service.example-inc.dcv.intermediary.example.

The Intermediary then adds the actual Validation Record in a domain they control:

    example.com.service.example-inc.dcv.intermediary.example.  IN   TXT  "<provider-random-token>"

Such a setup is especially useful when the Application Service Provider wants to periodically re-issue the challenge with a new provider random token. CNAMEs allow automating the renewal process by letting the Intermediary place the random token in their DNS zone instead of needing continuous write access to the User's DNS.

During provisioning and renewal, or whenever ownership of a domain changes, there is the risk that an attacker could leverage a "dangling CNAME" to perform a "subdomain takeover" attack ({{SUBDOMAIN-TAKEOVER}}).  Using a subdomain tied to the User prevents these attacks.  This can be accomplished by choosing a name that includes the User's account name (as shown above), a counter, or a random token linked to the User (as in {{random-token}}).

When a User stops using the Intermediary they should remove the domain control validation CNAME in addition to any other records they have associated with the Intermediary.

# Supporting Multiple Intermediaries {#multiple}

There are use-cases where a User may wish to simultaneously use multiple intermediaries or multiple independent accounts with an Application Service Provider. For example, a hostname may be using a "multi-CDN" where the hostname simultaneously uses multiple Content Delivery Network (CDN) providers.

To support this, Application Service Providers may support prefixing the challenge with a label containing an unique account identifier of the form `_<identifier-token>`. The identifier token is encoded in base32 or base16, and if the identifier is sensitive in nature, it should be run through a truncated hashing algorithm first. The identifier token should be stable over time and would be provided to the User by the Application Service Provider, or by an Intermediary in the case where domain validation is delegated ({{delegated}}).

The resulting record could either directly contain a TXT record or a CNAME (as in {{delegated}}).  For example:

    _<identifier-token>._service-challenge.example.com.  IN   TXT  "3419...3d206c4"

or

    _<identifier-token>._service-challenge.example.com.  IN   CNAME  <intermediary-random-token>.dcv.intermediary.example.


When performing validation, the Application Service Provider would resolve the DNS name containing the appropriate identifier token.

The ACME protocol has incorporated this method to specify DNS account specific challenages in {{ACME-DNS-ACCOUNT-ID}}.

Application Service Providers may wish to always prepend the `_<identifier-token>` to make it harder for third parties to scan, even absent supporting multiple intermediaries.  The `_<identifier-token>` MUST start with an underscore so as to not be a valid hostname.

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

## Application Usage Enumeration

The presence of a Validation Record with a predictable domain name (either as a TXT record for the exact domain name where control is being validated or with a well-known label) can allow attackers to enumerate the utilized set of Application Service Providers.

## Public Suffixes {#public-suffixes}

As discussed above in {{domain-boundaries}}, there are risks in allowing control to be demonstrated over domains which are "public suffixes" (such as ".co.uk" or ".com"). The volunteer-managed Public Suffix List ({{PSL}}) is one mechanism that can be used. It includes two "divisions" ({{PSL-DIVISIONS}}) covering both registry-owned public suffixes (the "ICANN" division) and a "PRIVATE" division covering domains submitted by the domain owner.

Operators of domains which are in the "PRIVATE" public suffix division often provide multi-tenant services such as dynamic DNS, web hosting, and CDN services. As such, they sometimes allow their sub-tenants to provision names as subdomains of their public suffix. There are use-cases that require operators of domains in the public suffix list to demonstrate control over their domain, such as to be added to the Public Suffix List, or to provision a wildcard certificate. At the same time, if an operator of such a domain allows its customers or tenants to create names starting with an underscore ("_") then it opens up substantial risk to the domain operator for attackers to provision services on their domain.

Whether or not it is appropriate to allow domain verification on a public suffix will depend on the application.  In the general case:

* Application Service Providers SHOULD NOT allow verification of ownership for domains which are public suffixes in the "ICANN" division. For example, "\_service-challenge.co.uk" would not be allowed.
* Application Service Providers MAY allow verification of ownership for domains which are public suffixes in the "PRIVATE" division, although it would be preferable to apply additional safety checks in this case.




# IANA Considerations

This document has no IANA actions.


--- back

# Appendix {#appendix}

Placeholder for things to put into appendix.

## Common Pitfalls {#pitfalls}

A very common but unfortunate technique in use today is to employ a DNS TXT record and placing it at the exact domain name whose control is being validated (e.g., often the zone apex). This has a number of known operational issues. If the User has multiple application services employing this technique, it will end up with multiple DNS TXT records having the same owner name; one record for each of the services.

Since DNS resource record sets are treated atomically, a query for the Validation Record will return all TXT records in the response. There is no way for the verifier to specifically query only the TXT record that is pertinent to their application service. The verifier must obtain the aggregate response and search through it to find the specific record it is interested in.

Additionally, placing many such TXT records at the same name increases the size of the DNS response. If the size of the UDP response (UDP being the most common DNS transport today) is large enough that it does not fit into the Path MTU of the network path, this may result in IP fragmentation, which can be unreliable due to firewalls and middleboxes is vulnerable to various attacks ([AVOID-FRAGMENTATION]). Depending on message size limits configured or being negotiated, it may alternatively cause the DNS server to "truncate" the UDP response and force the DNS client to re-try the query over TCP in order to get the full response. Not all networks properly transport DNS over TCP and some DNS software mistakenly believe TCP support is optional ([RFC9210]).

Other possible issues may occur. If a TXT record (or any other record type) is designed to be placed at the same domain name that is being validated, it may not be possible to do so if that name already has a CNAME record. This is because CNAME records cannot co-exist with other (non-DNSSEC) records at the same name. This situation cannot occur at the apex of a DNS zone, but can at a name deeper within the zone.

When multiple distinct services specify placing Validation Records at the same owner name, there is no way to delegate an application specific domain Validation Record to a third party. Furthermore, even without delegation, an organization may have a shared DNS zone where they need to provide record level permissions to the specific division within the organization that is responsible for the application in question. This can't be done if all applications expect to find validation records at the same name.


## Domain Boundaries {#domain-boundaries}

The hierarchical structure of domain names do not necessarily define boundaries of ownership and administrative control (e.g., as discussed in {{I-D.draft-tjw-dbound2-problem-statement}}). Some domain names are "public suffixes" ({{RFC9499}}) where care may need to be taken when validating control. For example, there are security risks if an Application Service Provider can be tricked into believing that an attacker has control over ".co.uk" or ".com". The volunteer-managed Public Suffix List {{PSL}} is one mechanism available today that can be useful for identifying public suffixes.

Future specifications may provide better mechanisms or recommendations for defining domain boundaries or for enabling organizational administrators to place constraints on domains and subdomains.

## Interactions with DNAME

Domain control validation in the presence of a DNAME {{RFC6672}} is possible with caveats. Since a DNAME record redirects the entire subtree of names underneath the owner of the DNAME, it is not possible to place a Validation Record under the DNAME owner itself. It would have to be placed under the DNAME target name, since any lookups for a name under the DNAME owner will be redirected to the corresponding name under the DNAME target.


# Acknowledgments

Thank you to Tim Wicinski, John Levine, Daniel Kahn Gillmor, Amir Omidi, Tuomo Soini, Ben Kaduk and many others for their feedback and suggestions on this document.
