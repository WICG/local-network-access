# Explainer for Local Network Access

This proposal is an early design sketch by the Chrome Secure Web and Network team to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

**Interested in testing Local Network Access in Chrome?** We have a blog post up with details for interested developers who want to test https://developer.chrome.com/blog/local-network-access

## Proponents

*   Chrome Secure Web and Network team

## Participate

*   https://github.com/explainers-by-googlers/local-network-access/issues

## Introduction

Currently public websites can probe a user's local network, perform CSRF attacks against vulnerable local devices, and generally abuse the user's browser as a "confused deputy" that has access inside the user's local network or software on their local machine. For example, if you visit `evil.com` it can use your browser as a springboard to attack your printer (given an HTTP accessible printer exploit).

Local Network Access aims to prevent these undesired requests to insecure devices on the local network. This is achieved by deprecating direct access to private IP addresses from public websites, and instead requiring that the user grants permission to the initiating website to make connections to their local network.

_Note:_ This proposal builds on top of Chrome's previously paused [Private Network Access (PNA) work](https://github.com/WICG/private-network-access/blob/main/explainer.md), but differs by gating access on a permission rather than via preflight requests. This increases the level of user control (at the expense of new permissions that have to be explained to the user) but removes the explicit "device opt-in" that the preflight design achieved. We believe this simpler design will be easier to ship, in order to mitigate the real risks of local network access today. Unlike the previous Private Network Access proposal, which required changes to devices on local networks, this approach should only require changes to sites that need to access the local network. Sites are much easier to update than devices, and so this approach should be much more straightforward to roll out. 

## Goals

*  **Stop exploitation of vulnerable devices and servers from the drive-by web.**
* Allow public websites to communicate to private network devices when the user expects it and explicitly allows it.

An adjacent goal is that we want a path for browsers to be good stewards of OS-level local network access permissions. These OS-level permissions are increasingly common ([on iOS, and more recently on macOS](https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy)) -- simply because the _browser_ has been granted the permission (for legitimate browser functionality the user may want to use, like mirroring the contents of a tab on a local device) should not expose users' local devices to the risks of the open web.

## Non-goals

* Break existing workflows and services that rely on a public web frontend that can control local network devices.
    * As long as there is _some_ path forward we should be okay with breaking some use cases (e.g., iframe and HTML subresources that aren't explicitly sourced from local hostnames), but overall we want to minimize breakage.
* Solve the local network HTTPS problem.
    * As stated in the [original Private Network Access explainer](https://github.com/WICG/private-network-access/blob/main/explainer.md#non-goals): _Provide a secure mechanism for initiating HTTPS connections to services running on the local network or the user's machine. This piece is missing to allow secure public websites to embed non-public resources without running into mixed content violations, with the exception of http://localhost which is embeddable. While a useful goal, and maybe even a necessary one in order to deploy Private Network Access more widely, it is out of scope of this specification._

## Use cases

### Use case 1

The most common case is for users who don't have any services or devices on their local network that expect connections from websites. Today, browsers freely allow JavaScript and subresource requests to these devices without any indication to the end user. Unless this behavior is expected by the user, users should not be exposed to this risk by default.

### Use case 2

Public web frontend for controlling or setting up local devices (such as an IoT device, home router, etc.).

Device manufacturers want to be able to give users an easy process for setting up a new device, and one method that is used is to have a page hosted on the manufacturer's public website which then communicates with the device via the user's browser, explicitly relying on the browser's vantage point inside the user's network.

This also reduces the complexity needed on the device itself -- for example, a smart toothbrush does not need to support a full webserver. Additionally, by being a public webpage under the control of the manufacturer, the setup page is always up to date.

## Proposed Solution

We propose gating the ability for a site to make requests to the users' local network behind a new "local network access" permission. Any origin that has not been granted this permission would be blocked from making such requests.

### Address spaces

As defined in the [original Private Network Access proposal](https://github.com/WICG/private-network-access/blob/main/explainer.md#address-spaces), we organize an IP network into three layers from the point of view of a node, from most to least private:

* Localhost: accessible only to the node itself, by default
* Private IP addresses: accessible only to the members of the local network (e.g. RFC1918)
* Public IP addresses: accessible to anyone

We call these layers **address spaces**: `loopback`, `local`, and `public`.

_(Note: The original PNA proposal called these `local`, `private`, and `public`. Changing this was considered in https://github.com/WICG/private-network-access/issues/91 but reverted due to already using the "private network access" name and values in headers implemented by sites and device manufacturers.)_

We note that `local` includes [RFC 1918](https://datatracker.ietf.org/doc/html/rfc1918)/[RFC 4193](https://datatracker.ietf.org/doc/html/rfc4193) private/local IP addresses and [RFC 6762](https://datatracker.ietf.org/doc/html/rfc6762) link-local names (`.local` hostnames). (See [discussion on the original PNA proposal repository](https://github.com/WICG/private-network-access/issues/4).)

### Local network requests

We define a **local network request** as a request crossing an address space boundary to a more-private address space. That is, any of the following are considered to be local network requests:

1. `public` -> `local`
2. `public` -> `loopback`
3. `local` -> `loopback`

Note that `local` -> `local` is not a local network request, as well as `loopback` -> anything. (See "cross-origin requests" below for a discussion on potentially expanding this definition in the future.)

A request is considered to be going to a `local` space if:

* The hostname is a private IP address literal (per RFC 1918 etc.), or
* The hostname is a `.local` domain (per RFC 6762), or
* The `fetch()` call is annotated with `targetAddressSpace="local"` (see "Integration with Fetch" below).

In these cases we know _a priori_ that the request is `local`.

Similarly, if a request is to a loopback IP literal (e.g., 127.0.0.1), `localhost`, or the `fetch()` call is annotated with `targetAddressSpace="loopback"`, then the request is `loopback`.

Separately, a request may eventually end up being considered a local network request if the request's hostname resolves to a private or loopback IP address. (We do not know this a priori however, and so cannot exempt these requests from mixed content blocking -- see "Mixed Content" below.)

### Permission prompts

When a site makes a local network request, the UA should check if the origin has already been granted the "local network access" permission. If not, the request should be blocked while the UA displays a prompt to the user asking whether they want to allow the origin to make requests to their local network. If the user denies the permission prompt, the request fails. If the user accepts the permission prompt, the request continues.

To reduce breakage (due to the lack of local network HTTPS), the permission also exempts requests that are known to be `local` or `loopback` from mixed content blocking (see "Mixed Content" below.)

### How this solution would solve the use cases

#### Use case 1: unexpected usage

For a user who is not expecting a site to connect to their local network, when `example.com` tries to call `fetch("http://192.168.0.1/routerstatus")`, the user's browser will ask whether or not to allow `example.com` to make connections to the local network. Since the user is not expecting this behavior, they can deny the permission request, and `example.com` is blocked from making these connections. 

#### Use case 2: controlling local devices

An existing site run by a device manufacturer that talks to a local device by making `fetch()` requests would potentially make minor modifications (for example, ensuring that they either use a private IP address or `.local` name when referring to the device, or adding the `targetAddressSpace="local"` property to their `fetch()` calls). When the site first tries to make a request to the device, the user sees a permission prompt. If the user expects the site to be communicating with devices on their local network, they can choose to grant the permission and the site will continue to function. If the user does not expect this or does not want to grant the permission to the site, they choose to not grant the permission to the site, and no local network requests from the site will be allowed.

## Detailed design discussion

### Integration with Fetch

The Fetch spec does not integrate the details of DNS resolution, only defining an **obtain a connection** algorithm, thus Local Network Access checks are applied to the newly-obtained connection. Given complexities such as Happy Eyeballs ([RFC6555](https://datatracker.ietf.org/doc/html/rfc6555), [RFC8305](https://datatracker.ietf.org/doc/html/rfc8305)), these checks might pass or fail non-deterministically for hosts with multiple IP addresses that straddle IP address space boundaries.

After we have obtained a connection, if we detect a local network request:

* If the client is not in a secure context, block the request.
* Check if the origin has previously been granted the local network access permission; if not, prompt the user.
* If the user grants permission, the request will proceed.

However, the requirement that local network requests be made from secure contexts means that any insecure request will be blocked as mixed content unless we can know ahead of time that the request should be considered a local network request.

To make this easier for developers, we propose adding a new parameter to the `fetch()` options bag to explicitly tag the address space of the request. For example: 

```js
fetch("http://router.com/ping", {
  targetAddressSpace: "local",
});
```

This would instruct the browser to allow the fetch even though the scheme is non-secure and obtain a connection to the target server. The `targetAddressSpace` should be either `local` or `loopback`.

If the remote IP address does not belong to the IP address space specified as the targetAddressSpace option value, then the request is failed. This ensures the feature cannot be abused to bypass mixed content in general. See "Mixed content" below for more details.

### Mixed content

There is a challenge in the combination of (1) requiring secure contexts in order to make PNA requests, (2) trying to load PNA subresources over HTTP (due to the lack of local network HTTPS), and (3) applying mixed content blocking. The Fetch specification applies mixed content upgrading and blocking steps well before we have obtained a connection.

We can know ahead of the mixed content checks whether a request has a local target address if:

* The hostname is a private IP address literal (per RFC1918 etc.), or
* The hostname is a `.local` domain, or
* The `fetch()` call is annotated with the `{ targetAddressSpace: "local" }` option.

If the request meets any of those requirements, we skip steps 6 and 7 of **main fetch** (upgrading mixed content and blocking mixed content) and mark the request as a local network request. Then, after obtaining the connection the local network access checks can fully run, and if the origin is not granted the local network access permission the request will be blocked. Additionally, if the request ends up not resolving to a local network endpoint, we need to block the request (as it should have been blocked as mixed content).

A new parameter `targetAddressSpace` will be added as a `fetch()` API option, allowing a developer to specify that the request should be treated as going to a `local` or `loopback` address space. This allows for HTTP local network requests that are not private IP literals or .local domains as long as they are explicitly tagged with the target address space. This also allows the developer to deterministically request the permission.

### Integration with HTML

`Document`s and `WorkerGlobalScope`s store an additional **address space** value. This is initialized from the IP address the document or worker was sourced from.

### Integration with WebRTC

Local connection attempts that use WebRTC should also be gated behind the Local Network Access permission prompt.

In the WebRTC spec, algorithms that add candidates to the ICE Agent already have steps that ensure [“administratively prohibited”](https://www.w3.org/TR/webrtc/#dfn-administratively-prohibited) addresses are not used. We can modify these algorithms to perform the following steps if the candidate has a loopback or local address:
 * Check if the origin has previously been granted the local network access permission; if not, prompt the user.
 * If the user grants permission, the algorithm will continue.
 * If the user denies the permission, we won’t add the candidate to the ICE Agent and it won’t be used when establishing a connection.

Note that these checks are done asynchronously and don’t block resolving the methods where they are used i.e. setRemoteDescription() and addIceCandidate().

The same checks should also be performed when connecting to STUN/TURN servers with loopback or local addresses.

### Integration with Permissions Policy

By default the ability to make local network requests will be limited to top-level documents that are secure contexts. There are use cases where a site needs to be able to delegate this permission into a subframe. To support these use cases, a new policy-controlled feature ("local-network-access") will be added that will allow top-level documents to delegate access to this feature to subframes.

### Integration with Permissions API

The permission will be integrated with the Permissions API, which will allow sites to query the status of the permission grant.

### HTML subresources

HTML subresource fetches go through the standard Fetch algorithm, but will not have the ability to specify an explicit `targetAddressSpace`. This includes subframe navigations.

For HTTP subresources, only "local names" (i.e., private IP literals or .local hostnames) are allowed for local network requests. This is required for resolving the mixed content problem (see "Mixed content" above, and see "Considered alternatives" for more involved methods that have been discussed).

### Websockets

The **establish a WebSocket connection** algorithm depends on the Fetch algorithm ([in the updated WHATWG spec](https://websockets.spec.whatwg.org/#websocket-protocol)), so Websockets should behave like other Fetch requests and trigger local network access prompts without additional work.

### Service Workers

Requests from a service worker go through the Fetch algorithm, and will be included in local network access restrictions.

## Considered alternatives

### Private Network Access (PNA)

The previously proposed [Private Network Access work](https://github.com/WICG/private-network-access/blob/main/explainer.md) (PNA for short, also previously referred to as CORS-RFC1918) required a secure CORS preflight response from the private subresource server. If the preflight failed, the request would be blocked. If the request was for an insecure resource (e.g., due to the lack of trusted local HTTPS), they [proposed a separate permission prompt](https://github.com/WICG/private-network-access/blob/main/permission_prompt/explainer.md) to allow the connection to a specific endpoint device _and_ relax the mixed content restrictions on the connection.

A lot of effort and developer outreach went into this effort, but it was never able to ship. Chrome currently has an opt-in mode behind an enterprise policy, and for a while Chrome shipped a restriction where private network access was restricted to secure contexts only (with a deprecation trial for developers who could not yet meet this requirement due to having HTTP-only private network endpoints). The previous plan was to build and ship the "mixed content permission prompt" to get these remaining developers out of the reverse origin trial.

PNA met a lot of different developer and user needs, and in the "good case" (secure website talking to a local network device that had a publicly trusted TLS certificate and a "PNA-aware" server) could be quite seamless, since it required no user intervention. In the non-ideal cases, PNA accumulated a lot of workarounds to address use cases that would result in a permission prompt in many cases (a device chooser style prompt for insecure devices). For example:

* Some routers filter out DNS responses mapping public domain names to private IP addresses -- this means that [fallback to direct (insecure) private IP addresses is required](https://github.com/WICG/private-network-access/issues/23) (and thus a permission prompt to bypass mixed content blocking), even for developers who provision publicly trusted certs to local network servers.
* Some devices could not update to supply preflight, so the permission prompt was relaxed to allow this ([but only grant an ephemeral permission](https://github.com/WICG/private-network-access/blob/main/permission_prompt/explainer.md#ephemeral-permission)).

Even if the local device has opted in to connections from a top level site, we believe there is value in user awareness and control over this exchange.

The use of preflights (without any user consent speed bump) also exposed its own risks. For example, timing attacks could be used to determine valid IP addresses on the network ([crbug.com/40051437](https://crbug.com/40051437), https://github.com/WICG/private-network-access/issues/41).

### Block all local network requests

Given the risks of allowing sites to use the browser as an access point into the user's local network, we could simply block all local network requests (or just any requests that cross from a public address space to a more private one). This would be simplest, but would break many existing use cases, or require expensive workarounds.

### Do nothing

As more operating systems are implementing local network access permissions at the application level, we believe it is the duty of user agents to broker access to that privilege in order to be good stewards of it. A user may have legitimate reasons to grant access for their _browser_ (e.g., Chrome's "cast this tab" functionality) but never want a _site_ to be able to access their local network.

### Alternatives for the mixed content problem

In our proposal above we recommend restricting local network HTML subresources to "assumed local" hostnames (such as `.local` domains) as a middle ground that meets developer needs, is relatively easy to deploy, and doesn't require complex technical or specification work to accomplish.

Below are some alternatives that have been considered for addressing the mixed content problem.

#### Require secure local network subresources

In order to restrict local network access to secure contexts (which is necessary in order for a permission prompt to make sense), we need some resolution of the mixed content problem. The Fetch specification orders the "upgrade or block mixed content" steps _before_ we have obtained a connection, and thus before we can know the IP address.

Currently, some developers can work around this by getting publicly trusted certificates for their servers running on local networks (e.g., Plex getting Let's Encrypt certs under a different subdomain for every install) but it is a substantial engineering and maintenance burden. Even for developers that go to the trouble of using publicly trusted certificates, [fallback to HTTP is required in some network circumstances](https://github.com/WICG/private-network-access/issues/23).

#### List subresource address space details in a header or meta tag

This could be a "treat hostname as public" / "treat hostname as local" property that could be specified in a response header or in a meta tag (or a response header meta equiv). 

An initial idea was to add this on to [CSP](https://www.w3.org/TR/CSP3/), however that was rejected due to CSP already being a bit overloaded and this not being a great conceptual fit for it. A separate "Sec-Treat-Origin-As-Private" header (or something along those lines) could be used to list origins that should be assumed to resolve to private IP addresses.

Additionally, it might be useful if such a header could be specified via an http-equiv meta tag (like one can do with CSP).

### Potential future changes

#### Top-level navigations to local network

Top-level navigations remain a risk after restrictions on subresource local network requests are in place. For an attacker, main frame navigations are noisier (compared to subresource requests and iframe navigations), although popunder techniques could potentially be used to hide navigations from the user.

To prevent these, we could block or show an interstitial warning when a public page navigates to a local one. To avoid too much breakage or over-warning, we could maybe scope protections to just these cases. We might consider that "complex" requests such as POST navigations and GET requests with URL parameters are particularly risky. We might also be able to "defang" navigations by stripping the URL parameters to reduce the risk of exploitation.

#### Consider all cross-origin requests to private addresses as "local network requests"

We could instead define a **local network request** as "any request targeting an IP address in the local or loopback address space, regardless of the requestor's address space", while maintaining the exception for not blocking same-origin requests. This is a stricter / broader definition, and would likely cause more widespread breakage than our proposal to only consider requests that cross from one address space class to a more private one.

This would also allow the mixed content relaxation that is granted when an origin is given the local network access permission to apply to `local` -> `local` or `loopback` -> `local` requests. This has been raised as a concern by developers in https://github.com/WICG/private-network-access/issues/109. Note that the status quo could continue here for now -- i.e., the top level page can get around mixed content checks by remaining on HTTP.

#### More granular permission grants

A [recent study](https://martina.lindorfer.in/files/papers/nwscanning_oakland25.pdf) (Schmidt et al., S&P 2025) found that the _transitivity_ of the local networks permission on iOS was the hardest for users to understand. It might be beneficial to scope the permission grant to an origin to the specific local network the user is currently connected to. Should the user grant permission to example.com on their home network, it may be reasonable for a UA to re-prompt the user if they bring their device to a different location and visit example.com again.

#### Add address space properties to HTML

PNA 1.0 worked to add a new `targetAddressSpace` parameter to `fetch()` to label the target address space of the request, so that mixed content checks could be relaxed (and then enforced if that connection ended up being public, to avoid it being a mixed content blocking bypass). The challenge is how to handle HTML subresources (e.g., img, iframe, etc.). We could add a new property to these HTML elements allowing developers to "label" them as public/local/loopback. This would function similarly to the parameter on `fetch()` and allow the user agent to initially bypass mixed content checks (and to know it should trigger the permission prompt). Currently we don't think this is necessary, as most use cases that require explicitly marking the `targetAddressSpace` should be able to switch to using `fetch()`.

## Security & Privacy Considerations

**Security**

* While the local network access permission exempts requests to a priori known local endpoints from mixed content blocking, the page should still be considered to have loaded mixed content if such a request is made. This means that, for example, browsers can choose to show a different security UI for pages that make insecure connections to the local network.
* Compared to the original PNA proposal, a site granted the local network access permission has more power to probe and connect to devices on the local network, regardless of whether those devices expect it.
* There is some risk of users accepting the permission without understanding it (which couldn't happen with preflights).

**Privacy**

* Compared to the original PNA proposal, no local network connections are allowed until the user has explicitly granted permission to a site.
* Compared to the original PNA proposal, there are no preflights (and thus no risk of timing/probing attacks from them).

## Stakeholder Feedback / Opposition

The previous PNA proposal (using preflights) was positively received by Mozilla (https://github.com/mozilla/standards-positions/issues/143) and WebKit (https://github.com/WebKit/standards-positions/issues/163). 

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Titouan Rigoudy, Jonathan Hao, and Yifan Luo who worked on the original PNA proposals and specification, and generously discussed their work with me and helped brainstorm paths forward.
