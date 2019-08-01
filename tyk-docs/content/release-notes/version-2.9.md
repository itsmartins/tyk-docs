---
title: Tyk Gateway v2.9
menu:
  main:
    parent: "Release Notes"
weight: 1
---


### HMAC request signing 

Now Tyk can sign request with HMAC, before sending to the upsteam.

The feature is implemented using [Draft 10](https://tools.ietf.org/html/draft-cavage-http-signatures-10) RFC.

`(request-target)` and all the headers of the request will be used for generating signature string. 
If request doesn't contain `Date` header, middleware will add one as it is required according to above draft.

A new config option `request_signing` is added in API Definition to enable/disable request signing. It has following format

```json
"request_signing": {
  "is_enabled": true,
  "secret": "xxxx",
  "key_id": "1",
  "algorithm": "hmac-sha256"
}
```
Following algorithms are supported:
1. `hmac-sha1`
2. `hmac-sha256`
3. `hmac-sha384`
4. `hmac-sha512`

### DNS Caching
Added a global dns cache in order to reduce the number of request to gateway's local dns server and appropriate gateway config section. This feature is turned off by default.

```
"dns_cache": {
    "enabled": true, //Turned off by default
    "ttl": 60, //Time in seconds before the record will be removed from cache
    "multiple_ips_handle_strategy": "pick_first" //A strategy, which will be used when dns query will reply with more than 1 ip address per single host.
},
```