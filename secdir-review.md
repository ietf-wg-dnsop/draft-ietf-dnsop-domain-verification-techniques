# SecDir review of draft-ietf-dnsop-domain-verification-techniques-05
CC @kaduk

Since the changes from the -01 that I previously reviewed are so substantial,
this is mostly a de novo review.  The main comment on the diff is on the
weakening of the guidance to enable DNSSEC validation from MUST to SHOULD,
which I assume had ample WG discussion.  Perhaps the identified exception
cases that keep it from being a MUST could be mentioned in the draft, though?

On the whole, though, this is good stuff, and I'll be happy to see it
published.  The one "discuss" point is mostly about being clear about what the
actual requirements are for domain control validation and what we believe
we're getting from the random token -- while I believe that our recommendation
is secure and the right general recommendation, the PSL example seems to be a
case that does securely perform the validation, but without a random token,
which is a good opportunity for us to think about the underlying requirements.

## Discuss

### Recommendation for random token

In toplevel §5 we say that "an issued random token then needs to exist in at
least one of these to demonstrate the User has control over the domain name
in-question", but this could stand to be more precise.  While I agree that the
random token is needed in general, we should be talking about which of
unguessable, probabilistically collision-free, unique, bound to a given
issuance event, etc. are needed.  (We say just (§5.1) "128 bits of entropy",
which is plenty to give the first three properties but needs a bit of help
(underscore prefix) to get the last, and doesn't say which properties we're
relying on that entropy for.)  In particular the scheme used by the PSL
(§A.1.1.5) does not use a random token but as far as I can tell it is fit for
purpose, so I would say that the core underlying requirement is not just
"random".  I'll include some notes from my analysis below, with suggestion
that we both clarify the specifics of the requirement (and why that translates
to needing to be random in the general case) and include in §A.1.1.5 some
discussion of why this the PSL flow is a secure flow.

The PSL's verification scheme (§A.1.1.5) uses a github pull request URL rather
than a random token.  It seems like this is actually a secure flow, since it
does provide a clear binding to intent to perform the requested operation
and is collision-free (by use of a central authority).  But to assess that, we
really need to enumerage the risks that a DCV scheme needs to protect against.
This list is going to include at least:
- collisions, where an authorizing DNS record intended to authorize use at one
  service provider also authorizes use at another (this is most prominent when
  using a TXT record at the name being verified itself but may show up
  elsewhere))
- forwarding of challenge values from one service provider to another (so
  that a user authorizes an application other than the one they intend to)
- more generally, confusion between the application and user about the scope
  of what is being authorized (e.g., single domain vs wildcard, among others)
- an attacker guessing what the verification token will be and causing the
  corresponding DNS record to exist through means other than the authorization
  flow

By having github generate the token (a URL) in a way that guarantees
uniqueness, we prevent collisions, and the PR content covers what the intended
action being confirmed is.  It is pretty possible to guess what an upcoming
verification token would be, but it seems pretty challenging to get someone to
create a DNS record pointing to a URL that talks about something totally
different.  So while prediction/collision is possible, it seems pretty easy to
detect as malicious and avoid using the collided URL.

The PR also gets a nice binding of intent, since the literal operation being
authorized is right there in the page referenced by the URL; for the scheme we
recommend we need to combine both the underscore prefix and the random token
to get an indication of intent (as provider plus unique token for specific
authorization event), and even then rely on the provider's docs/UI to be clear
about what they're doing on their end.

But for all I'm lauding the PSL method, it's not suitable for general use
since there's not a central authority to assign numbers/URLs, nor is there a
generic way to have the token/URL clearly refer to the operation being
authorized.  And in many cases the parties involved don't want the operation
being authorized to be particularly public -- the PSL is a rare case in that
the whole point of it is to be public!

## Comments

    I remain a little nervous about allocating a new BCP number for a topic of
    fairly narrow scope such as this, but the guidance is good and worth
    publishing, and I have not better alternative to offer, so I will continue to
    stifle any objection I might have raised.

    On the whole the content is reasonable, but read on for some suggestions on
    how to tighten things up and some questions about why all the pieces are
    needed.

    I also created a pull request with some editorial suggestions, at
    https://github.com/ietf-wg-dnsop/draft-ietf-dnsop-domain-verification-techniques/pull/149

Already Merged with thanks

### Use of "foo" as example content

    A bunch of examples use challenge names like _[foo-challenge.example.com](http://foo-challenge.example.com/), but
    of course neither _foo nore _foo-challenge is present in the Underscored and
    Globally Scoped DNS Node Names registry.  There is an entry for _example, but
    even an _[example-challenge.example.com](http://example-challenge.example.com/) entry would probably be confusing since
    the two "example"s serve different purposes.  Do we want to register something
    less "foo"-like and use it for documentation purposes?

I like your idea of something better than. I generallty think of these records as used by a service

\_service-challenge

would be acceptable to me. But I suck at naming.


### DONE Registry for validation record RDATA metadata

    It looks like we specify the "token" and "expiry" metadata keys for the RDATA
    format.  Do we want a registry or some other guidance to application service
    providers about avoiding conflicts within their own usage and with any future
    evolution of this BCP that adds more metadata keys?

So the guidance is to use _underscore records (conflicted guidance below), which would mean
_kaduk.example.com TXT "tokenk=value" would be unique to your space.  If I created a record for my service
_tjw.example.com TXT "tokenk=novalue"
They should not conflict.

There is an open issue on underscore guidance:
https://github.com/ietf-wg-dnsop/draft-ietf-dnsop-domain-verification-techniques/issues/138


### DONE Proposal or Actual guidance

    The abstract says that this document "proposes" some best practices, but it
    seems to me that with this level of review we can safely say that it
    "provides" some best practices.  (There is one other instance of the word
    "proposes" in the body of the document but it does not seem to have as broad
    of a scope as this one in the abstract.

s/This document proposes some best practices/This document provides some best practices/


### fragmentation often does not work

In §3 we write:

    > this may result in IP fragmentation, which often does not work reliably on
    > the Internet today due to firewalls and middleboxes, and also is vulnerable
    > to various attacks ([AVOID-FRAGMENTATION])

    But [AVOID-FRAGMENTATION] seems to mostly be a reference for the attacks
    possible, and I don't see much in there to support the "often does not work
    reliably" part.  That, in turn, is a statement that probably does merit a
    reference, so hopefully we have a decent one handy.

    The subsequent "Not all networks properly transmit DNS over TCP" might benefit
    from a reference to supplement RFC 9210 as well, but RFC 9210 does have at
    least some coverage of the topic.

Fragmentation is a touchy topic in the DNS space. I do agree that the phrase "often does not work reliably" needs something to buttress this comment.

We have updated our comments on fragmentation based on other feedback.

for an additional reference to RFC 9210, perhaps RFC7766 "DNS Transport over TCP - Implementation Requirements" would be useful.

### Direction of authority flow

In §3 we say:

> In the more general case of an Internet application service granting
> authority to a domain owner, again no existing DNS challenge scheme makes
> this distinction today.

But I am not sure I understand why the statement is written such that
authority is flowing from the application service to the domain owner; I would
have expected the authority to flow the other direction.

### Hiding provider use

In §5.2 we say:

    > An Application Service Provider may also specify prepending a random token to
    the name, such as "<RANDOM_TOKEN>._<PROVIDER_RELEVANT_NAME>-challenge". This
    can be done either as part of the challenge itself (Section 5.9, to support
    multiple Intermediaries (Section 5.5), or to make it harder for a third party
    to scan what Application Service Providers are being used by a given domain
    name.

but I am not sure how difficult this actually makes the scanning process.
Per RFC 8020 shouldn't a query for _<PROVIDER_RELEVANT_NAME>-[challenge.name](http://challenge.name/)
give NODATA if there is a record for a child name with a random token, which
would be distinguishable from the NXDOMAIN that is expected if the provider is
not being used?  (Similarly for the last paragraph of §5.5.)

### Challenge scoping status

In §5.2.1 we note that [ACME-SCOPED-CHALLENGE] has incorporated the scope
indication format proposed here, but up in §4 we say that "no existing DNS
challenge scheme makes this distinction today".  Since the ACME work is just
an I-D and thus work in progress, these two statements may not be technically
in conflict, but we might want to wordsmith slightly to clarify how they are
aligned with each other.

### Equivalent forms of specifying RDATA

In §5.3.2 we note that RDATA of "token=3419...3d206c4" is semantically
equivalent to that of just "3419...3d206c4".  This raises two questions: (1)
do we actually want to recommend two equivalent options for doing something vs
just always having a single right way to do things?  (2) If the answer to (1)
is 'yes', shouldn't we say that the application service provider needs to
specify which format expects (or require them to accept either form)?

### DONE Additional safety checks

    The closing note of the security considerations (§6.1) is that "it would be
    preferable to apply additional safety checks in this case" (the case of
    allowing verification of ownership for domains which are public suffixes in
    the "PRIVATE" division).  Is there any additional direction on what form such
    additional safety checks might take or what goals they would need to serve?  I
    do not think we need to have detailed directions in this document, but a sense
    for what risks should get protected against would be useful.

Shivan covered this in https://github.com/ietf-wg-dnsop/draft-ietf-dnsop-domain-verification-techniques/pull/158

### DONE Reference classification

    I think that [RFC1464] and [RFC3339] should move to the normative references
    section.

So I normally look over the references as chair/shepherd, and I use the guidance oh how previous documents have treated references as some guidance.

RFC1464 - more informative

RFC3339 - Yes Normative,.

Good catch

## Nits

### DONE temporary

    In §1 we talk about only one "temporary DNS record" being sufficient to prove
    control, but that's the only instance of "temporary" in the document (we do
    say "time-bound", but only after this instance of "temporary")... it's
    unclear if we want to have some other mention of it being temporary, or remove
    that mention, or something else.

one use of "temporary" ? yes it can be time-bound

### DNS Administrator

    We use the term "DNS Administrator" a few times, but it's not in the
    definitions section.  Should it be (or those usages changed to "User")?

The 'User' we view to be a service owner, someone who has to prove validation

The attempt with DNS Administrator was to be the manages the data in a DNS Zone.
We can go with "zone operator" but now I feel we do need to express that.

### RR implementation list

In §5 we list two things that the RR used to implement DCV includes.  In the
HTML rendering, this is rendered as "1) stuff 2) more stuff" with no line
break or conjunction.  Probably a line break would help, and I might also go
with "with both:" to clarify that it is an "and" statement rather than an "or"
one.

### DONE Github code

    §A.1.1.4 covers GitHub's _github-challenge-ORGANIZATION.DOMAIN, and says that
    "[t]he code is a numeric code that expires in 7 days."  Is this "code" the
    record RDATA or something else?

Github uses the word "code" in their documentation, such as "Use this code for the value of the TXT record

How about we replace that last sentence with:

"Their systems will provide a string for the value of the TXT record, which will expire in 7 days."

### DONE Typical Let's Encrypt Usage

    In §A.1.2.2.1 we say that "[t]ypically, [DCV by Let's Encrypt or other CA] is
    done via the [DNS-01] challenge."  I would probably weaken this to "often",
    rather than "typically", since I know of a number of sites that use the
    http-01 challenge via certbot or similar.

s/Typically,/Often/ works for me

