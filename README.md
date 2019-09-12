Strict Navigation Security
==========================

Mike West, September 2019

(_Note: This isn't a proposal that's well thought out, and stamped solidly with the Google Seal of
Approval. It's a collection of interesting ideas for discussion, nothing more, nothing less._)

A Problem
---------

[Strict Transport Security](https://tools.ietf.org/html/rfc6797) (HSTS) aims to give developers the
ability to ensure that a given site's users will _always_ communicate with the site over secure
channels. This, of course, is wonderful, and has been a substantial part of the security story for
many years.

Unfortunately, the fact is that this mechanism creates a unique source of global, web-visible state.
As noted in [Section 14.9 of the spec](https://tools.ietf.org/html/rfc6797#section-14.9), persisting
HSTS state creates an implicit identifier. As user agents generally shift away from wide
availability of explicit identifiers, mechanisms like this one have been used in the wild to rebuild
tracking capabilities that users have little insight into or control over.

Early last year, Apple provided a good explanation of this mechanism in
["Protecting Against HSTS Abuse"](https://webkit.org/blog/8146/protecting-against-hsts-abuse/), and
put forward a set of concrete mitigations that shipped in Safari 10. These mitigations seem quite
reasonable, but do place limits on the user agent's ability to protect users' future navigations by
collecting HSTS state from subresources, and also rely on a categorization of sites into "trackers"
and "non-trackers" when determining whether a given subresource is upgraded.

This document suggests that, in combination with a few other proposals floating around at the
moment, a simpler solution might be possible.

A Proposal
----------

At the moment, some user agents are considering one or more of the following proposals:

1.  Require cookies that can be delivered cross-site (e.g. `SameSite=None`) to
    [_also_ assert the `Secure` attribute](https://mikewest.github.io/cookie-incrementalism/draft-west-cookie-incrementalism.html#rfc.section.3.2).
    This has the effect of dropping cookies entirely from cross-site requests to non-secure
    endpoints.

2.  [Shifting the notion of "same-site" to exclude HTTP <-> HTTPS transitions](https://github.com/whatwg/url/issues/448),
    which (among other things) has the effect of hardening the boundary created by the `SameSite`
    cookie attribute against network attackers.

3.  [Forcibly upgrading mixed content](https://groups.google.com/a/chromium.org/forum/#!msg/blink-dev/XI1otbsuvMw/X_65dN7qAgAJ).

It seems to me that this set of changes substantially mitigates the same problems that HSTS solves
with respect to subresource requests. The first two prevent cookies from being delivered along with
non-secure subresource requests, and the last will reduce the number of non-secure subresource
requests over time.

With these changes in place, I'd suggest that we could relatively safely:

1.  Apply HSTS assertions _only_ to top-level navigations, and not to subresource requests.

2.  Continue to collect HSTS assertions from subresource requests.

This would allow broad collection of HSTS assertions, and would apply those assertions broadly to
top-level navigations, protecting users against downgrade attacks. Subresource requests, however,
would not be upgraded by HSTS. If they're blocked/upgraded as part of mixed content (#3 above),
then HSTS is superfluous. Even if they're not blocked/upgraded, explicit session state will be
stripped by #1 & #2, substantially reducing the risk associated with the non-secure request.


FAQ
---

### Is it really safe to collect HSTS assertions from _all_ subresource requests?

If we collect HSTS assertions from all subresource requests, then we do create the opportunity for
passing around partial identity via top-level navigations that are redirected through
[~20 hosts](https://fetch.spec.whatwg.org/#concept-http-redirect-fetch) on their way to a page the
user wanted to visit. There's real, visible cost to this tactic, but we should carefully consider
the possibility that that cost isn't high enough to prevent its usage.

If we decide that that risk is something we need to mitigate, we have options. For instance, we
could collect HSTS assertions only from first-party subresources, or double-key the assertions
against the top-level origin. These wouldn't create as broad a protection for users, but would
certainly reduce the potential for abuse.

### Couldn't we safely apply HSTS upgrades to subresource requests to [preloaded HSTS hosts](https://hstspreload.org)?

We could. A quite reasonable variant of the proposal above would not bypass the user agent's
HSTS cache entirely for subresources, but only the dynamic portion collected from a user's browsing
over time. This would still mitigate the tracking risk associated with HSTS (assuming that ~all
users have the same list of preloaded hosts).

It would also require user agents to distinguish preloaded from dynamic HSTS assertions, and would
increase pressure on presence in the HSTS preload list, which is already bursting at the seams. In
a world where we can't ship forcible upgrades, it might be a necessary tradeoff, but if we can avoid
the complexity, I'd like to.

### Does this enable interesting, network-based attacks?

Non-secure sites probably try to load resources like `http://www.google-analytics.com/analytics.js`
all the time, and HSTS currently ensures that these requests don't create a single chokepoint that
[attackers could easily abuse](https://www.netresec.com/?page=Blog&month=2015-03&post=China%27s-Man-on-the-Side-Attack-on-GitHub).
In the absence of forcible upgrades (or outright blocking), this proposal does carry the real risk
of creating more non-secure requests that could be manipulated by the network.

Applying preloaded assertions, as discussed above, could substantially mitigate this concern.

### What about HTTP state beyond cookies?

We should indeed ensure that we're not sending identifiers out along with non-secure subresource
requests. A proposal that I haven't made yet (until now!) is that we should unset non-secure,
cross-site requests' `credentials flag` during
[HTTP-network fetch](https://fetch.spec.whatwg.org/#http-network-fetch). This would more explicitly
and robustly ensure that mechanisms like the
[basic/digest authentication scheme](https://httpwg.org/specs/rfc7235.html) are disabled for these
risky requests, and align request credentials generally with the `SameSite=None;Secure` proposal
discussed above.

Filed [whatwg/fetch#936](https://github.com/whatwg/fetch/issues/936) to discuss that proposal.


Acks
----

This document is my synthesis of a few conversations with very smart people. Thanks to David
Benjamin, Kaustubha Govind, Matt Menke, Chris Palmer, Ryan Sleevi, and Emily Stark for some
internal discussion; and to folks like Brent Fulgham and John Wilander for taking concrete
action in this space that we can learn from.
