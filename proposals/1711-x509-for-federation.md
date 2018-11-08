# MSC1711: X.509 certificate verification for federation connections

TLS connections for server-to-server communication currently rely on an
approach borrowed from the [Perspectives
project](https://web.archive.org/web/20170702024706/https://perspectives-project.org/)
to provide certificate verification, rather than the more normal model using
certificates signed by trusted Certificate Authorities. This document sets out
the reasons that this has not been a success, and suggests that we should
instead revert to the CA model.

## Background: the failure of the Perspectives approach

The Perspectives approach replaces the conventional hierarchy of trust provided
by the Certificate Authority model with a large number of "notary" servers
distributed around the world. The intention is that the notary servers
regularly monitor remote servers and observe the certificates they present;
when making a connection to a new site, a client can correlate the certificate
it presents with that seen by the notary servers. In theory this makes it very
hard to mount a Man-in-the-Middle (MitM) attack, because it would require
intercepting traffic between the target server and a large number of the notary
servers.

It is notable that the Perspectives project itself appears to have largely been
abandoned: its website has largely been repurposed, the [Firefox
extension](https://addons.mozilla.org/en-GB/firefox/addon/perspectives/) does
not work with modern versions of Firefox, the [mailing
list](https://groups.google.com/forum/#!forum/perspectives-dev) is inactive,
and several of the (ten) published notary servers are no longer functional. The
reasons for this are not entirely clear, though clearly it never gained
widespread adoption.

When Matrix was originally designed in 2014, the Perspectives project was
heavily active, and avoiding dependencies on the relatively centralised
Certificate Authorities was attractive, in accordance with Matrix's design as a
decentralised protocol. However, this has not been a success in practice.

Matrix was unable to make use of the existing notary servers (largely because
we wanted to extend the protocol to include signing keys): the intention was
that, as the Matrix ecosystem grew, public Matrix servers would act as notary
servers. However, in practice we have ended up in a situation where almost <sup
id="a1">[1](#f1)</sup> every Matrix homeserver either uses `matrix.org` as the
sole notary, or does no certificate verification at all. Far from avoiding the
centralisation of the Certificate Authorities, the entire protocol is therefore
dependent on a single point of control at `matrix.org` - and because
`matrix.org` only monitors from a single location, the protection against MitM
attacks is weak.

It is also clear that the Perspectives approach is poorly-understood. It is a
common error for homeservers to be deployed behind reverse-proxies which make
the Perspectives-based approach unreliable. The CA model, for all its flaws, is
at least commonly used, which makes it easier for administrators to deploy
(secure) homeservers, and allows server implementations to leverage existing
libraries.

## Proposal

We propose that Matrix homeservers should be required to present valid TLS
certificates, signed by a known Certificate Authority, on their federation
port.

In order to ease transition, we could continue to follow the current,
perspectives-based approach for servers whose TLS certificates fail
validation. However, this should be strictly time-limited (for three months,
say), to give administrators time to switch to a signed certificate. The
`matrix.org` team would proactively attempt to reach out to homeserver
administrators who do not update their certificate.

Once the transition to CA-signed certificates is complete, the
`tls_fingerprints` property of the
[`/_matrix/key/v2`](https://matrix.org/docs/spec/server_server/unstable.html#retrieving-server-keys)
endpoints would be redundant and we should consider removing it.

The process of determining which CAs are trusted to sign certificates would be
implementation-specific, though it should almost certainly make use of existing
operating-system support for maintaining such lists. It might also be useful if
administrators could override this list, for the purpose of setting up a
private federation using their own CA.

### Interaction with SRV records

With the use of `SRV` records, it is possible for the hostname of a homeserver
to be quite different from the matrix domain it is hosting. For example, if
there were an SRV record at `_matrix._tcp.matrix.org` which pointed to
`server.example.com`, then any federation requests for `matrix.org` would be
routed to `server.example.com`. The question arises as to which certificate
`server.example.com` should present.

In short: the server should present a certificate for the matrix domain
(`matrix.org` in the above example). This ensures that traffic cannot be
intercepted by a MitM who can control the DNS response for the `SRV` record
(perhaps via cache-poisoning or falsifying DNS responses).

This will be in line with the current
[requirements](https://matrix.org/docs/spec/server_server/unstable.html#resolving-server-names)
in the Federation API specification for the `Host`, and by implication, the TLS
Server Name Indication <sup id="a2">[2](#f2)</sup>.

### Interaction with `.well-known` files

[MSC1708](https://github.com/matrix-org/matrix-doc/blob/rav/proposal/well-known-for-federation/proposals/1708-well-known-for-federation.md)
proposes an alternative to SRV records, in the form of `.well-known` files. In
this instance, a file at `https://matrix.org/.well-known/matrix/server` might
direct requests to `server.example.com`.

In this case, `server.example.com` would be required to present a valid
certificate for `server.example.com`.

Because the request for the `.well-known` file takes place over a validated TLS
connection, this is not subject to the same DNS-based attacks as the SRV
record, and this mechanism allows the owners of a domain to delegate
responsibility for running their Matrix homeserver without having to hand over
TLS keys for the whole domain.

### Extensions

HTTP-Based Public Key Pinning (HPKP) and
[Certificate transparency](https://www.certificate-transparency.org) are
both HTTP extensions which attempt to work around some of the deficiencies in
the CA model, by making it more obvious if a CA has issued a certificate
incorrectly.

HPKP has not been particularly successful, and is
[deprecated](https://developers.google.com/web/updates/2018/04/chrome-67-deps-rems#deprecate_http-based_public_key_pinning)
in Google Chrome as of April 2018. Certificate transparency, however, is seeing
widespread adoption from Certificate Authories and HTTP clients.

This proposal sees both technologies as optional techniques which could be
provided by homeserver implementations. We encourage but do not mandate the use
of Certificate Transparency.

### Related work

The Perspectives approach is also currently used for exchanging the keys that
are used by homeservers to sign Matrix events and federation requests (the
"signing keys"). Problems similar to those covered here also apply to that
mechanism. A future MSC will propose improvements in that area.

## Tradeoffs

There are well-known problems with the CA model, including a number of
widely-published incidents in which CAs have issued certificates
incorrectly. It is therefore important to consider alternatives to the CA
model.

### Improving support for the Perspectives model

In principle, we could double-down on the Perspectives approach, and make an effort
to get servers other than `matrix.org` used as notary servers. However, there
remain significant problems with such an approach:

* Perspectives remain complex to configure correctly. Ideally, administrators
  need to make conscious choices about which notaries to trust, which is hard
  to do, especially for newcomers to the ecosystem. (In practice, people use
  the out-of-the-box configuration, which is why everyone just uses
  `matrix.org` today).

* A *correct* implementation of Perspectives really needs to take into account
  more than the latest state seen by the notary servers: some level of history
  should be taken into account too.

Essentially, whilst we still believe the Perspectives approach has some merit,
we believe it needs further research before it can be relied upon. We believe
that the resources of the Matrix ecosystem are better spent elsewhere.

### DANE

DNS-Based Authentication of Named Entities (DANE) can be used as an alternative
to the CA model. (It is arguably more appropriately used *together* with the CA
model.)

It is not obvious to the author of this proposal that DANE provides any
material advantages over the CA model. In particular it replaces the
centralised trust of the CAs with the centralised trust of the DNS registries.

## Potential issues

Beyond the problems already discussed with the CA model, requiring signed
certificates comes with a number of downsides.

### More difficult setup

Configuring a working, federating homeserver is a process fraught with
pitfalls. This proposal adds the requirement to obtain a signed certificate to
that process. Even with modern intiatives such as Let's Encrypt, this is
another procedure requiring manual intervention across several moving parts<sup
id="a3">[3](#f3)</sup>.

On the other hand: obtaining an SSL certificate should be a familiar process to
anybody capable of hosting a production homeserver (indeed, they should
probably already have one for the client port). This change also opens the
possibility of putting the federation port behind a reverse-proxy without the
need for additional configuration. Hopefully making the certificate usage more
conventional will offset the overhead of setting up a certificate.

### Inferior support for IP literals

Whilst it is possible to obtain an SSL cert which is valid for a literal IP
address, this typically requires purchase of a premium certificate; in
particular, Let's Encrypt will not issue certificates for IP literals. This may
make it impractical to run a homeserver which uses an IP literal, rather than a
DNS name, as its `server_name`.

It has long been the view of the `matrix.org` administrators that IP literals
are only really suitable for internal testing. Those who wish to use them for
that purpose could either disable certificate checks inside their network, or
use their own CA to issue certificates.

### Inferior support for hidden services (`.onion` addresses)

It is currently possible to correctly route traffic to a homeserver on a
`.onion` domain, provided any remote servers which may need to reach that
server are configured to route to such addresses via the Tor network. However,
it can be difficult to get a certificate for a `.onion` domain (again, Let's
Encrypt do not support them).

The reasons for requiring a signed certificate (or indeed, for using TLS at
all) are weakened when traffic is routed via the Tor network. It may be
reasonable to relax the requirement for a signed certificate for such traffic.

## Conclusion

We believe that requiring homeservers to present an X.509 certificate signed by
a recognised Certificate Authority will improve security, reduce
centralisation, and eliminate some common deployment pitfalls.

<a id="f1"/>[1] It's *possible* to set up homeservers to use servers other than
`matrix.org` as notaries, but we are not aware of any that are set up this
way. [↩](#a1)

<a id="f2"/>[2] I've not been able to find an authoritative source on this, but
most reverse-proxies will reject requests where the SNI and Host headers do not
match. [↩](#a2)

<a id="f3"/>[3] Let's Encrypt will issue ACME challenges via port 80 or DNS
(for the `http-01` or `dns-01` challenge types respectively). It is unlikely
that a homeserver implementation would be able to control either port 80 or DNS
responses, so we will be unable to automate a Let's Encrypt certificate
request. [↩](#a3)