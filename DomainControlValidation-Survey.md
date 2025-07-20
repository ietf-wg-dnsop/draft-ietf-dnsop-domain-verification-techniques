# Survey of Domain Control Validation Techniques


## TXT based

A TXT record is usually the default option for domain control validation. The Application Service Provider asks the User to add a DNS TXT record (perhaps through their domain host or DNS provider) at the domain with a certain value. Then the Application Service Provider does a DNS TXT query for the domain being verified and checks that the correct value is present. For example, this is what a DNS TXT record could look like for an Application Service Provider `Service`:

    example.com.   IN   TXT   "237943648324687364"

Here, the value "237943648324687364" serves as the randomly-generated TXT value being added to prove ownership of the domain to `Service` Application Service Provider. Note that in this construction Application Service Provider `Service` would have to query for all TXT records at "example.com" to get the Validation Record. Although the original DNS protocol specifications did not associate any semantics with the DNS TXT record, {{RFC1464}} describes how to use them to store attributes in the form of ASCII text key-value pairs for a particular domain. In practice, there is wide variation in the content of DNS TXT records used for domain control validation, and they often do not follow the key-value pair model. Even so, the RDATA {{RFC1034}} portion of the DNS TXT record has to contain the value being used to verify the domain. The value is usually a Random Token in order to guarantee that the entity who requested that the domain be verified (i.e., the person managing the account at Application Service Provider `Service`) is the one who has (direct or delegated) access to DNS records for the domain. After a TXT record has been added, the Application Service Provider will usually take some time to verify that the DNS TXT record with the expected token exists for the domain. The generated token typically expires in a few days.

Some Application Service Providers use a prefix of `_PROVIDER_NAME-challenge` in the Name field of the TXT record challenge. For ACME, the full Host is `_acme-challenge.<YOUR_DOMAIN>`. Such patterns are useful for doing targeted domain control validation. The ACME protocol ([RFC8555]) has a challenge type `DNS-01` that lets a User prove domain ownership. In this challenge, an implementing CA asks you to create a TXT record with a randomly-generated token at `_acme-challenge.<YOUR_DOMAIN>`:

    _acme-challenge.example.com.  IN  TXT "cE3A8qQpEzAIYq-T9DWNdLJ1_YRXamdxcjGTbzrOH5L"

{{RFC8555}} (section 8.4) places requirements on the Random Token.

### Let's Encrypt

The ACME example is implemented by Let's Encrypt https://letsencrypt.org/docs/challenge-types/#dns-01-challenge.

### Google Workspace

Google Workspace (https://support.google.com/a/answer/2716802) asks the User to sign in with their administrative account and obtain their token as part of the setup process for Google Workspace. The verification token is a 68-character string that begins with "google-site-verification=", followed by 43 characters. Google recommends a TTL of 3600 seconds. The owner name of the TXT record is the domain or subdomain name being verified. 

### The AT Protocol

The Authenticated Transfer (AT) Protocol supports DNS TXT records for resolving social media "handles" (human-readable identifiers) to the User's persistent account identifier https://atproto.com/specs/handle#dns-txt-method. For example, this is how the handle `bsky.app` would be resolved:

    _atproto.bsky.app.  IN  TXT "did=did:plc:z72i7hdynmk6r22z27h6tvur"

### GitHub

To verify domains for organizations, GitHub asks the user to create a DNS TXT record under `_github-challenge-ORGANIZATION.<YOUR_DOMAIN>`, where ORGANIZATION stands for the GitHub organization name. The RDATA value for the provided TXT record is a string that expires in 7 days https://docs.github.com/en/github/setting-up-and-managing-organizations-and-teams/verifying-your-organizations-domain.

### Public Suffix List

The Public Suffix List (https://publicsuffix.org/) asks for owners of private domains to authenticate by creating a TXT record containing the pull request URL for adding the domain to the Public Suffix List.  For example, to authenticate "example.com" submitted under pull request 100, a requestor would add:

    _psl.example.com.  IN TXT "https://github.com/publicsuffix/list/pull/100"

## CNAME based

### CNAME for Domain Control Validation

#### DocuSign

Docusign CNAME (https://support.docusign.com/s/document-item?rsc_301=&bundleId=rrf1583359212854&topicId=gso1583359141256_1.html) asks the User to add a CNAME record with the "Host Name" set to be a 32-digit random value pointing to `verifydomain.docusign.net.`.

#### Google Workspace CNAME

Google Workspace CNAME (https://support.google.com/a/answer/112038) lets you specify a CNAME record for verifying domain ownership. The User gets a unique 12-character string that is added as "Host", with TTL 3600 (or default) and Destination an 86-character string beginning with "gv-" and ending with ".domainverify.googlehosted.com.".

### Delegated Domain Control Validation

#### Content Delivery Networks (CDNs)

In order to be issued a TLS cert from a Certification Authority like Let's Encrypt, the requester needs to prove that they control the domain. Often this is done via the  https://letsencrypt.org/docs/challenge-types/#dns-01-challenge  challenge. Let's Encrypt only issues certs with a 90 day validity period for security reasons https://letsencrypt.org/2015/11/09/why-90-days.htm. This means that after 90 days, the DNS-01 challenge has to be re-done and the random token has to be replaced with a new one. Doing this manually is error-prone. 

Content Delivery Networks offer to automate this process using a CNAME record in the User's DNS that points to the Validation Record in the CDN's zone. 


#### AWS Certificate Manager (ACM)
https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html{ACM-CNAME  allows delegated domain control validation. The record name for the CNAME looks like:

     _<random-token1>.example.com.  IN   CNAME _<random-token2>.acm-validations.aws.

The CNAME points to:

     _<random-token2>.acm-validations.aws.  IN   TXT "<random-token3>"

Here, the random tokens are used for the following:

* `<random-token1>`: Unique sub-domain, so there's no clashes when looking up the Validation Record.
* `<random-token2>`: Proves to ACM that the requester controls the DNS for the requested domain at the time the CNAME is created.
* `<random-token3>`: The actual token being verified.

Note that if there are more than 5 CNAMEs being chained, then this method does not work.


#### Akamai

    _acme-challenge.domain.com. IN CNAME validation.validate-akdv.net.

https://techdocs.akamai.com/property-mgr/docs/add-hn-with-default-cert-la

#### Fastly

    _acme-challenge.domain.com. IN CNAME token.fastly-validations.com.

https://docs.fastly.com/en/guides/setting-up-tls-with-certificates-fastly-manages

#### Cloudflare

    _acme-challenge.domain.com. IN CNAME 

    Cloudflare's validation URL (for partial DNS setups)

https://developers.cloudflare.com/ssl/edge-certificates/changing-dcv-method/methods/delegated-dcv/

#### F5 Distributed Cloud

Customers create a CNAME record for 

    _acme-challenge.demo.f5lab.com 

with the value provided by F5 Distributed Cloud

https://f5cloud.zendesk.com/hc/en-us/articles/24624288087959-What-is-an-ACME-challenge-and-how-should-I-enter-these-records-in-my-DNS-zone

#### Google Cloud Certificate Manager

Google's Certificate Manager uses DNS authorization where customers add CNAME records like 

    _acme-challenge.myorg.example.com 

pointing to Google's validation domains for certificate provisioning

https://cloud.google.com/certificate-manager/docs/deploy-google-managed-dns-auth

#### Certify DNS

A cloud hosted version of the acme-dns protocol that uses CNAME delegation of ACME challenge TXT records to a dedicated challenge response service
https://docs.certifytheweb.com/docs/dns/providers/certifydns/

#### acme-dns (open source)


An open source limited DNS server with RESTful HTTP API where customers create CNAME records like 

    _acme-challenge.domainiwantcertfor.tld. IN  CNAME a097455b-52cc-4569-90c8-7a4b97c6eba8.auth.example.org

https://github.com/joohoi/acme-dns


