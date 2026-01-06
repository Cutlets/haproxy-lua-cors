# HAProxy CORS Lua Library

Lua library for enabling CORS in HAProxy.

## Supported versions of HAProxy

This library will work with HAProxy version 1.8 and newer. (Note: Preflight interception requires HAProxy 2.2+).

## Background

Cross-origin Request Sharing (CORS) allows you to permit client-side code running within a different domain to call your services. This module extends HAProxy so that it can:

* Set an **Access-Control-Allow-Methods** header in response to a preflight request.
* Set an **Access-Control-Allow-Headers** header in response to a preflight request.
* Set an **Access-Control-Max-Age** header in response to a preflight request.
* Set an **Access-Control-Allow-Credentials** header to allow authenticated requests (Cookies, Authorization headers, etc.).
* Set an **Access-Control-Allow-Origin** header to whitelist a domain.

This library checks the incoming `Origin` header and tries to match it with the list of permitted domains. If there is a match, that domain is sent back in the `Access-Control-Allow-Origin` header.

It also sets the `Vary` header to `Accept-Encoding, Origin` so that caches do not reuse cached CORS responses for different origins.

## Dependencies

* HAProxy must be compiled with Lua support.

## Installation

1. Copy the `cors.lua` file to your HAProxy server.
2. **Important:** Ensure the `core.register_action` line at the bottom of `cors.lua` is set to accept **4** arguments:
```lua
core.register_action("cors", {"http-req"}, cors_request, 4)

```



## Usage

Load the `cors.lua` file via the `lua-load` directive in the `global` section:

```haproxy
global
    lua-load /path/to/cors.lua

```

### Configuration

In your `frontend` or `listen` section, use `http-request lua.cors` with **4 parameters**:

1. **Allowed Methods**: Comma-delimited list (e.g., `"GET,POST,OPTIONS"`).
2. **Allowed Origins**: Comma-delimited list of permitted domains.
3. **Allowed Headers**: Comma-delimited list of custom headers.
4. **Allow Credentials**: Set to `"true"` to enable, or `"false"` to disable.

> **Note:** The first three parameters can be set to an asterisk (`*`) to allow all values. However, if `Allow Credentials` is `"true"`, the `Origin` cannot be a wildcard.

Add `http-response lua.cors` to the same section to process the response headers.

## Allowed origin patterns

| Pattern | Example | Description |
| --- | --- | --- |
| Domain name | `mydomain.com` | Allow any scheme (HTTP/HTTPS) |
| Generic scheme | `//mydomain.com` | Allow any scheme (HTTP/HTTPS) |
| Specific scheme | `https://mydomain.com` | Allow only HTTPS |
| Specific port | `http://mydomain.com:8080` | Allow only specific port |
| Subdomains | `.mydomain.com` | Allow ALL subdomains |
| Wildcard port | `http://mydomain.com:*` | Allow ANY port |

## Examples

**Example 1: Specific domains with Credentials enabled**

```haproxy
frontend https-in
    bind *:443 ssl crt /etc/ssl/certs/
    
    acl is_api path_beg /api/
    
    # Parameters: Methods, Origins, Headers, Credentials
    http-request lua.cors "GET,POST,DELETE,OPTIONS" "extension.copykiller.com,myapp.com" "Content-Type,Authorization" "true" if is_api
    
    # Apply CORS to the response
    http-response lua.cors if { var(txn.origin) -m found }

```

**Example 2: Public API (No Credentials)**

```haproxy
http-request lua.cors "*" "*" "*" "false"
http-response lua.cors

```

## Preflight Requests

For HAProxy 2.2+, the module intercepts `OPTIONS` requests and returns a `204 No Content` immediately. It returns:

* `Access-Control-Allow-Methods`
* `Access-Control-Allow-Headers`
* `Access-Control-Allow-Credentials` (if set to "true")
* `Access-Control-Max-Age`: 600

## Security Note

When **Allow Credentials** is `"true"`, the library will automatically echo the request's `Origin` (if whitelisted) instead of a wildcard `*`, as required by the CORS specification.

## License & Modifications

This project is licensed under the **Apache License 2.0**.

### Original Work

* **Copyright (c) 2019. Nick Ramirez [nramirez@haproxy.com**](mailto:nramirez@haproxy.com)
* **Copyright (c) 2019. HAProxy Technologies, LLC.**
* Original Source: [HAProxy Technologies GitHub](https://github.com/haproxytech/haproxy-lua-cors) (or relevant origin)

### Modifications

In accordance with the Apache License 2.0, Clause 4(b), the following modifications have been made to the original software:

* **Added support for `Access-Control-Allow-Credentials` header.**
* **Extended `cors_request` and `preflight_request_ver2` functions** to accept and process the `allow_credentials` parameter.
* **Updated `core.register_action**` to support 4 arguments in the configuration.
* **Modified `cors_response**` to include credentials logic in the response phase.
