# Delegated Domain Control Validation

A survey of several different methods deployed today for DNS based Delegated Domain Control Validation. These are primarily Content Delivery Networks (CDNs) performing validation for SSL certificates. 


## Content Delivery Networks (CDNs)

In order to be issued a TLS cert from a Certification Authority like Let's Encrypt, the requester needs to prove that they control the domain. Often this is done via the {{DNS-01}} challenge. Let's Encrypt only issues certs with a 90 day validity period for security reasons https://letsencrypt.org/2015/11/09/why-90-days.htm. This means that after 90 days, the DNS-01 challenge has to be re-done and the random token has to be replaced with a new one. Doing this manually is error-prone. 

Content Delivery Networks offer to automate this process using a CNAME record in the User's DNS that points to the Validation Record in the CDN's zone. 

### AWS Certificate Manager (ACM)

AWS Certificate Manager {{ACM-CNAME}} allows delegated domain control validation. The record name for the CNAME looks like:

     _<random-token1>.example.com.  IN   CNAME _<random-token2>.acm-validations.aws.

The CNAME points to:

     _<random-token2>.acm-validations.aws.  IN   TXT "<random-token3>"

Here, the random tokens are used for the following:

* `<random-token1>`: Unique sub-domain, so there's no clashes when looking up the Validation Record.
* `<random-token2>`: Proves to ACM that the requester controls the DNS for the requested domain at the time the CNAME is created.
* `<random-token3>`: The actual token being verified.

Note that if there are more than 5 CNAMEs being chained, then this method does not work.

### Akamai

    _acme-challenge.domain.com. IN CNAME validation.validate-akdv.net.

https://techdocs.akamai.com/property-mgr/docs/add-hn-with-default-cert-la

### Fastly

    _acme-challenge.domain.com. IN CNAME token.fastly-validations.com.

https://docs.fastly.com/en/guides/setting-up-tls-with-certificates-fastly-manages

### Cloudflare

    _acme-challenge.domain.com. IN CNAME 

    Cloudflare's validation URL (for partial DNS setups)

https://developers.cloudflare.com/ssl/edge-certificates/changing-dcv-method/methods/delegated-dcv/

### F5 Distributed Cloud

Customers create a CNAME record for 

    _acme-challenge.demo.f5lab.com 

with the value provided by F5 Distributed Cloud

https://f5cloud.zendesk.com/hc/en-us/articles/24624288087959-What-is-an-ACME-challenge-and-how-should-I-enter-these-records-in-my-DNS-zone

### Google Cloud Certificate Manager

Google's Certificate Manager uses DNS authorization where customers add CNAME records like 

    _acme-challenge.myorg.example.com 

pointing to Google's validation domains for certificate provisioning

https://cloud.google.com/certificate-manager/docs/deploy-google-managed-dns-auth

### Certify DNS

A cloud hosted version of the acme-dns protocol that uses CNAME delegation of ACME challenge TXT records to a dedicated challenge response service
https://docs.certifytheweb.com/docs/dns/providers/certifydns/

### acme-dns (open source)


An open source limited DNS server with RESTful HTTP API where customers create CNAME records like 

    _acme-challenge.domainiwantcertfor.tld. IN  CNAME a097455b-52cc-4569-90c8-7a4b97c6eba8.auth.example.org

https://github.com/joohoi/acme-dns


