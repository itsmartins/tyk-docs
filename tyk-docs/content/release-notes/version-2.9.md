---
title: Tyk Gateway v2.9
menu:
  main:
    parent: "Release Notes"
weight: 1
---

### TCP Proxying

Tyk now can be used as a reverse proxy for your TCP services. It means that you can put Tyk not only on top of your APIs, but on top of **any** network application, like databases, services using custom protocols and etc.

The main benefit of using Tyk as your TCP proxy is that functionality you used to managed your APIs now can be used for your TCP services as well.  Features like load balancing, service discovery, Mutual TLS (both authorisation and communication with upstream), certificate pinning: all work exactly the same way as for your HTTP APIs. 

[TCP Proxy Docs](/docs/concepts/tcp-proxy/)

### APIs as Products

With this release we remove all the barriers on how you can mix and match policies together, providing you the ultimate flexibility for configuring your access rules.

Now key can have multiple policies, each containing rules for different APIs. In this case each distinct policy will have own rate limit and quota counters. For example if first policy gives access to `API1` and second policy to `API2` and `API3`, if you create a key with both policies, user will have access to all three APIs, where `API1` will have quotas and rate limits defined inside first policy, and `API2`, `API3` will have shared quotas and rate limits defined inside second policy.

Additionally now you can mix policies defined for the same API but having different path and methods access rules. For example you can have one policy which allows only access to `/users`  and second policy giving user access to `/companies` path. If you create key with both policies, their access rules will be merged, and user will get access to both paths.

#### Developer Portal Updates

Developers now can have multiple API keys, and subscribe to multiple catalogues with a single key. Go to the portal settings and set `Enable subscribing to multiple APIs with single key` option to enable this new flow. When enabled, developers will see the new API generation user interface, which allows users to request access to multiple Catalogues of the **same type**  with a single key. 

From implementation point of view, Developer objects now have  `Keys` attribute, which is the map where key is a `key` and value is array of policy ids. `Subscriptions` field considered deprecated, with retained compatibility. Added new set of Developer APIs to manage the keys, similar to deprecated subscriptions APIs. [See documentation]

Another changes:
- Added two new Portal templates, which is used by new key request flow  `portal/templates/request_multi_key.html`, `portal/templates/request_multi_key_success.html`
- Portal Catalogue list page updated to show Catalogue authentication mode
- API dashboard screen now show keys instead of subscriptions, and if subscribed to multiple policies, it will show allowance rules for all catalogues.
- Key request API updated to accept `apply_policies` array instead of `for_plan`


### JWT and OpenID scope support 

Now you can set granular permissions on per user basis, by injecting permissions to the "scope" claim of JWT token. To make it work you need to provide mapping between scope and policy ID, and thanks to enchanced policy merging capabilities mentioned above, Tyk will read scope value from JWT token and will generate dynamic access rules. Your JWT scopes can look like "users:read companies:write" or similar, it is up to your imagination. OpenID supports it as well, but at the moment only if your OIDC provider can generate ID tokens in JWT format (which is very common this days).

[JWT Scope docs](/docs/integrate/api-auth-mode/open-id-connect).

### Go plugins
 
[Go](https://golang.org/) is an open source programming language that makes it easy to build simple, reliable, and efficient software. The whole Tyk stack is written in Go language, and it is one of the reasons of behind our success. 

With this release you now can write native Go plugins for Tyk. Which means extreme flexibility and the best performance without any overhead. 

Your plugin can be as simple as:
```go
package main
import (
	"net/http"
)
// AddFooBarHeader adds custom "Foo: Bar" header to the request
func AddFooBarHeader(rw http.ResponseWriter, r *http.Request) {
	r.Header.Add("Foo", "Bar")
}
```

Follow our [documentation](/docs/customise-tyk/plugins/golang-plugins/golang-plugins.md) to learn more.

### Distributed tracing
We listen to our customers, and tracing was one of the most common requests recently. Distributed tracking takes your monitoring and profiling experience to the next level, since you can see the whole request flow, even if it has complex route though multiple services. And inside this flow, you can go deep down to the details like individual middleware execution performance.
At the moment we presenting OpenTracing support, with Zipkin and Jaeger as supported tracers.

See the [documentation](/docs/distributed-tracing/distributed-tracing).


### HMAC request signing 

Now Tyk can sign request with HMAC, before sending to the upsteam.

The feature is implemented using [Draft 10](https://tools.ietf.org/html/draft-cavage-http-signatures-10) RFC.

`(request-target)` and all the headers of the request will be used for generating signature string. 
If request doesn't contain `Date` header, middleware will add one as it is required according to above draft.

A new config option `request_signing` is added in API Definition to enable/disable request signing. It has following format:

```json
"request_signing": {
  "is_enabled": true,
  "secret": "xxxx",
  "key_id": "1",
  "algorithm": "hmac-sha256"
}
```
The following algorithms are supported:

1. `hmac-sha1`
2. `hmac-sha256`
3. `hmac-sha384`
4. `hmac-sha512`

### Simplified Dashboard installation experience
We worked a lot with our clients, to build a way nicer experience of on-boarding to Tyk. Now instead of using command line, you can just run dashboard, and follow user friendly web form, which will configure your dashboard. However, we did not forget about our experienced users too, and now provide CLI enchanced tools for bootstrapping Tyk in advanced maneer. 

See updated [Getting Started](/get-started/with-tyk-on-premise) section and [new CLI documentation](/docs/get-started/with-tyk-on-premise/installation/bootstrapper-cli).

### DNS Caching
Added a global dns cache in order to reduce the number of request to gateway's local dns server and appropriate gateway config section. This feature is turned off by default.

```
"dns_cache": {
    "enabled": true, //Turned off by default
    "ttl": 60, //Time in seconds before the record will be removed from cache
    "multiple_ips_handle_strategy": "pick_first" //A strategy, which will be used when dns query will reply with more than 1 ip address per single host.
},
```

### Python Plugin Improvements
We have made a massive rewrite of our Python scripting engine in order to simplify installation and usage of Python scripts. 
From now on, you no longer need to use a separate Tyk binary for Python plugins. Everything is now bundled to the main binary.
Which also means that you can combine JSVM, Python and Coprocess plugins inside the same installation. 
In addition you can now use any Python 3.x version. Tyk will automatically detect supported a version and will load the required libraries. If you have multiple Python version available, you can specify the exact version using `python_version`. 

### Importing Custom Keys using the Dashboard API
Previously if you wanted migrate to Tyk and keep existing API keys, you had to use low level Tyk Gateway API, which had lot of constraints, especially when its coming to complex setups with multiple organizations and data centers. 

We introducing a new Dashboard API for importing custom keys, which is as simple as `POST /api/keys/{custom_key} {key-payload}`. New API ensures that Keys from multiple orgs will not intersect, and it works for multi-data center setups, and even Tyk SaaS.

### Single sign on for the Tyk SaaS

Before SSO was possible only for Tyk On-Premise, since it required access to low-level Dashboard Admin APIs.
With 2.9 we added new a new Dashboard SSO API, which you can use without having super admin access, and it works on organization level. This means that all our Tyk SaaS users can use 3-rd party IDPs to manange Dashboard users and Portal developers.

See the (documentation)[/docs/tyk-dashboard-api/sso/]

### Importing WSDL APIs

WSDL now is a first class citizen at Tyk. You can take your WSDL definition and simply import to the dashboard, creating a nice boilerplate for your service. Additionally we presenting documentation on how to work with SOAP of any complexity with Tyk. See the [documentation](/docs/concepts/soap)