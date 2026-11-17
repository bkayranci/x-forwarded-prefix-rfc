---
stand_alone: true
ipr: trust200902
cat: std
submissiontype: IETF
area: Applications and Real-Time (ART)
wg: HTTP Working Group

docname: draft-bkayranci-x-forwarded-prefix-latest

title: The X-Forwarded-Prefix HTTP Header Field
abbrev: X-Forwarded-Prefix Header
lang: en
kw:
  - HTTP
  - Reverse Proxy
  - API Gateway
  - Forwarded
  - X-Forwarded-Prefix
  - Path Prefix
  - Routing
  - URI Reconstruction
author:
- role: editor
  ins: T. B. Kayrancioglu
  name: Türkalp Burak Kayrancıoğlu
  asciiFullname: Turkalp Burak Kayrancioglu
  country: TR
  email: turkalp.kayrancioglu@trendyol.com
- role: editor
  ins: Y. Ozen
  name: Yunus Özen
  asciiFullname: Yunus Ozen
  country: TR
  email: yunus.ozen@yalova.edu.tr

normative:
  RFC9110:
  RFC3986:
  RFC5234:
  RFC9651:

informative:
  RFC7239:
  RFC6648:
  SPRING-WEB:
    target: https://docs.spring.io/spring-framework/reference/web/webmvc/filters.html#x-forwarded-prefix
    title: Spring Framework Reference Documentation - Forwarded Headers
    author:
      org: VMware / Broadcom
    date: 2024
  ASPNET-CORE:
    target: https://learn.microsoft.com/aspnet/core/host-and-deploy/proxy-load-balancer
    title: Configure ASP.NET Core to work with proxy servers and load balancers
    author:
      org: Microsoft Corporation
    date: 2024
  KEYCLOAK-PROXY:
    target: https://www.keycloak.org/server/reverseproxy#_configure_the_reverse_proxy_headers
    title: Keycloak Server Administration Guide - Configuring the Reverse Proxy Headers
    author:
      org: Red Hat
    date: 2024
  ANGULAR-SECURITY:
    target: https://angular.dev/best-practices/security
    title: Angular Security Best Practices - Server-Side Request Forgery (SSRF) Prevention
    author:
      org: Google LLC
    date: 2024
  FABIO-LB:
    target: https://github.com/fabiolb/fabio
    title: Fabio - A fast, modern, zero-conf load-balancing HTTP(S) router for Consul
    author:
      name: Frank Schroeder
    date: 2024

--- abstract

This document specifies the `X-Forwarded-Prefix` HTTP header field, a widely deployed de-facto standard used by reverse proxies, API gateways, load balancers, and ingress controllers to communicate the original URI path prefix that was stripped, rewritten, or matched prior to forwarding a request to an upstream service. Standardizing this header provides normative behavior across HTTP intermediaries and origin servers, enabling upstream applications to reliably reconstruct original client-facing target URIs, issue accurate HTTP redirects, generate self-referential hypermedia links, configure application base paths, and mitigate security vulnerabilities arising from improper path handling and header injection. Furthermore, this document establishes the requirement for reverse proxy implementations to emit this header unless explicitly configured otherwise, specifies deterministic prefix concatenation for multi-proxy chaining, defines strict percent-encoding preservation, establishes atomic trust boundaries for the forwarded header family, specifies structured field mapping in accordance with RFC 9651, and registers the field in the IANA HTTP Field Name Registry along with an extension parameter for the RFC 7239 Forwarded header.

--- middle

# Introduction

In modern distributed web and microservice architectures, reverse proxies, API gateways, ingress controllers, load balancers, and service meshes frequently intercept incoming client requests and route them to upstream origin servers based on URI path prefixes. During this routing process, intermediaries commonly strip, rewrite, or trim the matched path prefix before transmitting the request to the upstream application service.

For example, an API gateway receiving a request for:

~~~
https://api.example.com/customers/v1/profile
~~~

may match the path prefix `/customers/v1`, strip that prefix, and forward the request to an internal upstream microservice as:

~~~
GET /profile HTTP/1.1
Host: internal-customers.local:8080
~~~

While path stripping simplifies internal application routing, it isolates upstream services from the client-facing context. Upstream applications frequently require knowledge of the original client-facing path prefix to:

* Construct absolute and path-relative URLs in `Location` response headers for HTTP redirects (e.g., redirecting `/profile` to `/customers/v1/profile/edit` or an authentication endpoint).
* Generate hypermedia controls and links (such as REST HATEOAS, HAL, or JSON:API specifications).
* Serve HTML documents containing correct relative or root-relative asset references (e.g., `<base href="...">`, CSS, JavaScript, and image URLs).
* Correctly configure Single Page Application (SPA) client-side routers and Server-Side Rendering (SSR) engines (such as Angular SSR).
* Provide accurate redirect and callback URIs for Identity Providers and authentication protocols, including OAuth 2.0 (RFC 6749) and OpenID Connect (e.g., Keycloak or ASP.NET Core OpenIdConnect handlers).
* Produce accurate interactive API documentation (e.g., OpenAPI / Swagger UI).

Without knowledge of the stripped prefix, an upstream service that emits a redirect to `/login` causes the client browser to request `https://api.example.com/login`, which bypasses the intended gateway route and typically results in an HTTP 404 (Not Found) error or improper routing.

## Scope and Delimitation

This specification is intentionally and strictly delimited to path-prefix stripping, prepending, and matching operations (commonly referred to as path-prefix routing).

Arbitrary regular-expression-based URL rewriting (such as transforming `/v1/users` into `/api/v2/users`, reordering arbitrary path segments, or dynamically rewriting query parameters) is inherently non-invertible from the perspective of an upstream application without distributing full bidirectional rewrite rules. Such arbitrary transformations are outside the scope of this specification and require alternate application-layer mechanisms (such as dedicated original-URI headers).

In modern cloud API gateways, ingress controllers, and microservice routers, the overwhelming majority (exceeding 95%) of service routing rules rely on deterministic prefix matching and stripping (e.g., matching a service mount point `/billing` and forwarding the remainder to an internal billing service). The `X-Forwarded-Prefix` header field is specifically and deterministically designed to resolve this universal operational pattern.

## Relationship to RFC 7239 and RFC 6648

{{RFC7239}} created the `Forwarded` HTTP header field to standardize proxy-related information, defining parameters for the client IP address (`for`), the proxy IP address (`by`), the original request authority (`host`), and the scheme (`proto`). However, {{RFC7239}} did not define a parameter or mechanism for conveying stripped or rewritten path prefixes.

Consequently, the software engineering industry adopted the `X-Forwarded-Prefix` header field as a de-facto standard. Over more than a decade, `X-Forwarded-Prefix` has achieved ubiquitous implementation across major web frameworks, reverse proxies, and enterprise platforms:

* The Spring Framework (`ForwardedHeaderFilter`, `UriComponentsBuilder`, and Spring Cloud Gateway) {{SPRING-WEB}}.
* Microsoft .NET and ASP.NET Core (`ForwardedHeadersMiddleware`, `ForwardedHeadersOptions`) {{ASPNET-CORE}}.
* Red Hat Keycloak identity and access management {{KEYCLOAK-PROXY}}.
* Angular Server-Side Rendering (SSR) base path resolution and SSRF defense {{ANGULAR-SECURITY}}.
* Go web ecosystems and load balancers (such as Fabio LB {{FABIO-LB}}, Traefik, and Chi/Gin middlewares).
* Python web frameworks (FastAPI / Starlette `root_path`, Uvicorn proxy header processing).
* Node.js, Express, and NestJS reverse proxy middleware.

Section 3 of {{RFC6648}} deprecated the "X-" prefix for newly designed application protocols, but explicitly acknowledged that widely deployed constructs MAY be documented and standardized for backwards compatibility and interoperability. `X-Forwarded-Prefix` is not a newly designed header field; it is an established de-facto protocol mechanism that currently routes billions of HTTP transactions daily across global enterprise infrastructure.

## Standards Track Justification

This document is targeted for the Standards Track, rather than Informational status, because it is prescriptive rather than descriptive:

1. **Normative Protocol Requirements:** It establishes mandatory protocol requirements (BCP 14 `MUST`, `MUST NOT`, `SHOULD`) governing proxy forwarding behavior, header sanitization at perimeter boundaries, and deterministic prefix concatenation across chained intermediaries.
2. **Mandatory Security Protections:** It mandates essential security validation mechanisms for upstream applications to prevent critical vulnerabilities, including open redirect attacks (e.g., `//evil.com`), server-side request forgery (SSRF), cache poisoning (requiring `Vary` header controls), and path traversal.
3. **Formal Extension of RFC 7239:** It defines normative updates to {{RFC7239}} by establishing the `prefix` parameter within the IANA HTTP Forwarded Parameters registry, ensuring a forward-compatible migration path away from the "X-" prefix without stranding legacy infrastructure.
4. **Permanent IANA Registration:** It requests permanent registration (`permanent` status) in the IANA HTTP Field Name Registry.

Leaving `X-Forwarded-Prefix` undocumented or merely Informational would perpetuate fragmented implementations, inconsistent path concatenation, and severe security vulnerabilities across the global web ecosystem.

## Requirements Language

{::boilerplate bcp14-tagged}

# The X-Forwarded-Prefix Header Field

The `X-Forwarded-Prefix` HTTP header field conveys the original URI path prefix that was removed, rewritten, or matched by one or more intermediate reverse proxies before the request was dispatched to the upstream origin server.

## Header Field Syntax

The `X-Forwarded-Prefix` header field is defined using the Augmented Backus-Naur Form (ABNF) notation of {{RFC5234}} and the core rules defined in Section 5.6 of {{RFC9110}}:

~~~ abnf
X-Forwarded-Prefix = #prefix-element
prefix-element     = path-absolute
~~~
{: title="ABNF Syntax for X-Forwarded-Prefix" sourcecode-markers="true"}

Where `path-absolute` is defined in Section 3.3 of {{RFC3986}} (a path beginning with a single `/`, but not beginning with `//`).

Examples of valid `X-Forwarded-Prefix` header fields include:

~~~ http
X-Forwarded-Prefix: /api
X-Forwarded-Prefix: /customers/v1
X-Forwarded-Prefix: /services/billing
X-Forwarded-Prefix: /
~~~

## Structured Field Values (RFC 9651) Considerations

Modern HTTP specifications encourage field definitions to map directly to Structured Field Values for HTTP {{RFC9651}}. However, `X-Forwarded-Prefix` was widely deployed before the standardization of {{RFC9651}}, and its syntax reflects pre-existing HTTP header conventions:

* In {{RFC9651}}, a token (`sf-token`) MUST begin with an ASCII letter (`ALPHA`) or asterisk (`*`), and cannot begin with a slash (`/`). Consequently, an unquoted path such as `/api/v1` is not a valid Structured Field token.
* In Structured Fields syntax, path values are represented as Strings (`sf-string`), which require enclosing quotation marks (e.g., `"/api/v1"`).
* In existing production deployments, reverse proxies and web frameworks universally emit and parse unquoted `path-absolute` values.

To ensure consistency with modern HTTP specifications while preserving backwards compatibility:

1. Senders generating `X-Forwarded-Prefix` using classical HTTP syntax emit unquoted `path-absolute` values as specified in Section 2.1.
2. Implementations that parse `X-Forwarded-Prefix` using generic Structured Field parsers map the field conceptually to a List of Strings (`sf-list` consisting of `sf-string` elements). Such parsers SHOULD accommodate unquoted string values to interoperate with existing reverse proxies.
3. The standardized {{RFC7239}} `prefix` parameter defined in Section 5 strictly employs quoted string syntax (`prefix="/api/v1"`), providing full compliance with Structured Field String rules.

## Path Normalization, Trailing Slashes, and Percent-Encoding

To prevent ambiguity when upstream applications concatenate the prefix with the received request path, intermediaries and origin servers MUST observe the following path formatting rules:

{:vspace}
Leading Slash:
: The `prefix-element` MUST begin with a single solidus (`/`). It MUST NOT begin with two or more solidi (`//`), to prevent downstream interpretation as a protocol-relative URI (see {{Security}}).

Trailing Slash:
: A reverse proxy SHOULD NOT append a trailing solidus (`/`) to the prefix value unless the entire prefix consists solely of a single solidus (`/`). For example, a proxy stripping `/api/v1/` from `/api/v1/users` SHOULD set `X-Forwarded-Prefix: /api/v1`. This allows an upstream application that receives request path `/users` to concatenate the prefix and path directly: `/api/v1` + `/users` = `/api/v1/users`.

Dot Segments:
: Intermediaries MUST remove any dot-segments (`.` and `..`) from the prefix in accordance with Section 5.2.4 of {{RFC3986}} prior to emitting the header field.

Percent-Encoding Preservation:
: The value of `X-Forwarded-Prefix` MUST strictly adhere to the `path-absolute` production of {{RFC3986}}. Unencoded spaces, control characters, and reserved characters outside the `pchar` production are prohibited and MUST be percent-encoded.
: When stripping a path prefix from an incoming request-target, an intermediary MUST preserve the raw percent-encoded representation as received in the request URI (e.g., preserving `/my%20app`, rather than decoding it to `/my app` or double-encoding it to `/my%2520app`). Intermediaries MUST NOT decode percent-encoded triplets prior to emitting the header field. Uppercase hexadecimal digits SHOULD be used for percent-encoding in accordance with Section 2.1 of {{RFC3986}}.
: Because the header preserves the exact percent-encoding of the original client URI, upstream applications can concatenate the prefix directly with the request path without introducing double-encoding or decoding mismatches.

## Multiple Intermediaries and Chaining

In multi-tier proxy deployments, an HTTP request may traverse multiple intermediaries that each perform path prefix stripping (for example, an edge API gateway stripping `/portal` followed by an internal cluster ingress stripping `/finance`).

Web application frameworks (including Spring Framework, ASP.NET Core, FastAPI, and Keycloak) universally expect `X-Forwarded-Prefix` to contain a single, consolidated path string (e.g., `/portal/finance`) to populate application base path properties (such as `PathBase` or `contextPath`). These frameworks do not parse or match routing rules against comma-separated lists.

To guarantee deterministic interoperability across multi-proxy topologies:

1. **Intermediary Concatenation Requirement:** When an intermediary receives an incoming request containing an `X-Forwarded-Prefix` header from a trusted upstream intermediary and subsequently strips an additional path prefix, the intermediary MUST concatenate its stripped prefix onto the existing prefix value (e.g., appending `/finance` to an existing `/portal` to yield `X-Forwarded-Prefix: /portal/finance`).
2. **List Emission Prohibition:** Intermediaries MUST NOT emit multiple `X-Forwarded-Prefix` header fields or format the prefix as a comma-separated list unless explicitly configured for legacy debugging.
3. **Upstream Application Parsing:** Although intermediaries MUST concatenate prefixes into a single canonical path, an upstream application that receives multiple comma-separated values (for example, due to legacy intermediaries or HTTP header field folding under Section 5.3 of {{RFC9110}}) MUST evaluate and concatenate the list elements from left to right into a single normalized path (e.g., resolving `/portal, /finance` into `/portal/finance`), or reject the request if configured for strict single-prefix validation.

# Reverse Proxy and Gateway Behavior

## Intermediary Emission Requirement

An intermediary that strips, rewrites, or matches a URI path prefix prior to forwarding a request to an upstream service SHOULD emit the `X-Forwarded-Prefix` header field unless explicitly configured otherwise.

Historical deployments frequently required manual operator configuration (e.g., explicitly configuring `proxy_set_header X-Forwarded-Prefix $prefix;`). Establishing this forwarding behavior as the standard default prevents configuration oversights that lead to broken links, broken redirects, and authentication failures in microservice deployments.

## Path Stripping and Header Generation

When an intermediary matches a request-target against a routing rule and rewrites or strips a path prefix:

1. The intermediary MUST compute the exact prefix path stripped from the original request URI.
2. The intermediary MUST normalize the prefix to comply with `path-absolute` syntax, preserving raw percent-encoding.
3. The intermediary MUST emit the `X-Forwarded-Prefix` header field on the forwarded request.

### Architecture and Flow Example

The following diagram illustrates the path stripping and prefix forwarding sequence:

~~~
+--------+                +-------------+             +----------+
| Client |                | API Gateway |             | Upstream |
+--------+                +-------------+             +----------+
    |                            |                          |
    | GET /store/products/42     |                          |
    | Host: example.com          |                          |
    |--------------------------->|                          |
    |                            | Strips prefix "/store"   |
    |                            | Sets X-Forwarded-Prefix  |
    |                            |------------------------->|
    |                            |                          | (Request)
    |                            |                          |
~~~
{: title="Path Stripping and Prefix Forwarding Flow"}

The forwarded HTTP request received by the upstream origin server:

~~~ http
GET /products/42 HTTP/1.1
Host: catalog-service.internal
X-Forwarded-Prefix: /store
X-Forwarded-Proto: https
X-Forwarded-Host: example.com
~~~

## Perimeter Ingress and Header Sanitization

An intermediary deployed at an untrusted trust boundary (such as an external Internet gateway accepting connections from arbitrary clients) MUST NOT trust incoming `X-Forwarded-Prefix` headers sent by clients.

The ingress intermediary MUST perform one of the following actions:

* Strip and discard any client-supplied `X-Forwarded-Prefix` header fields before applying routing rules.
* Overwrite the header field with the verified prefix determined by the gateway's own routing configuration.

Failure to sanitize incoming headers allows external attackers to perform header injection and open redirect attacks (see {{Security}}).

## Internal Chaining and Aggregation

Within a trusted internal network, when an intermediary receives an incoming request containing an `X-Forwarded-Prefix` header from a trusted peer intermediary:

* If the receiving intermediary does not modify the request path, it MUST forward the `X-Forwarded-Prefix` header unchanged.
* If the receiving intermediary strips an additional path segment, it MUST append its stripped segment to the existing prefix in accordance with Section 2.4 (e.g., appending `/checkout` to `/shop` to yield `/shop/checkout`).

# Origin Server and Upstream Application Processing

## Trust Boundaries and Atomic Forwarded Header Evaluation

Upstream origin servers and application frameworks MUST NOT blindly trust the `X-Forwarded-Prefix` header. An application MUST only evaluate `X-Forwarded-Prefix` when the immediate client connection originates from a trusted reverse proxy.

Trusted proxy verification mechanisms include:

* IP address allowlists (e.g., verifying the socket remote address matches private proxy subnets or container network gateways).
* Mutual TLS (mTLS) authentication between the proxy and upstream service.
* Shared secret tokens or internal network isolation.

### Atomic Evaluation Rule

`X-Forwarded-Prefix` shares an identical and inseparable trust boundary with `X-Forwarded-Host`, `X-Forwarded-Proto`, and `X-Forwarded-For` (as well as {{RFC7239}} `Forwarded`). A critical security failure occurs when an intermediary or origin server selectively validates or sanitizes one forwarded header (e.g., `X-Forwarded-Host`) while trusting or neglecting another (e.g., `X-Forwarded-Prefix`).

Intermediaries and upstream origin servers MUST apply an atomic trust policy:

* If the incoming connection is from an untrusted client or network interface, the intermediary MUST sanitize (strip or overwrite) ALL incoming `X-Forwarded-*` headers, and the upstream application MUST ignore or reject ALL incoming `X-Forwarded-*` headers.
* Intermediaries and applications MUST NOT selectively accept or trust `X-Forwarded-Prefix` without verifying the entire forwarded header set under the same trusted proxy policy.

## Target URI Reconstruction

When an upstream application needs to construct the original client-facing URI, it combines the forwarded headers with local request metadata:

| Component | Source Header Field / Metadata | Fallback |
| :--- | :--- | :--- |
| Scheme | `X-Forwarded-Proto` (or `Forwarded: proto=...`) | Incoming connection scheme (`http` / `https`) |
| Host & Port | `X-Forwarded-Host`, `X-Forwarded-Port` (or `Forwarded: host=...`) | `Host` header field |
| Path Prefix | `X-Forwarded-Prefix` (or `Forwarded: prefix=...`) | Empty string `""` |
| Request Path | Path component of the incoming HTTP request-line | `/` |
| Query String | Query component of the incoming HTTP request-line | None |
{: title="URI Component Resolution Matrix"}

The full client-facing URI is reconstructed as:

~~~
reconstructed_uri = Scheme + "://" + Authority +
                    Prefix + RequestPath [ "?" Query ]
~~~

Before performing string concatenation, the application MUST verify that combining `Prefix` and `RequestPath` does not introduce consecutive slashes (e.g., `/api//users`), resolving the path to a canonical normalized form.

## Framework Integration and Routing

Major web frameworks utilize the resolved prefix to establish an application base path (often termed `PathBase`, `contextPath`, or `root_path`).

When returning HTTP redirect status codes (`301`, `302`, `303`, `307`, or `308`), the server MUST prepend the prefix to path-absolute references in the `Location` header field:

~~~ http
HTTP/1.1 302 Found
Location: /store/products/42/details
~~~

In HTML generation, the application uses the prefix to populate the `<base href="...">` tag or to prefix relative asset links (stylesheets, scripts, images). In REST APIs, generated links in JSON payloads (e.g., `_links.self.href`) and OpenAPI schema `servers[].url` entries MUST incorporate the forwarded prefix.

## Identity Providers and OAuth 2.0 / OpenID Connect

Identity and Access Management (IAM) systems (such as Keycloak or ASP.NET Core OpenID Connect handlers) rely on deterministic URI reconstruction:

* The authorization server generates callback URIs, discovery documents (`/.well-known/openid-configuration`), and issuer URLs (`iss`) based on the client-facing base path.
* If an IAM system behind a reverse proxy does not receive `X-Forwarded-Prefix`, it generates issuer URLs reflecting internal hostnames and stripped paths (e.g., `http://internal-auth:8080/realms/master` instead of `https://auth.example.com/auth/realms/master`). This mismatch causes client token signature validation and redirect URI verification to fail.
* Conformant IAM systems MUST support `X-Forwarded-Prefix` to configure the public base path dynamically.

# Forwarded Parameter Extension for RFC 7239

{{RFC7239}} specifies the `Forwarded` header field and provides an IANA registry for forwarded parameters. While `X-Forwarded-Prefix` remains the overwhelmingly dominant implementation in existing deployments, this document also registers the `prefix` parameter for the `Forwarded` header field to provide a unified syntax for future systems.

Although {{RFC6648}} deprecated the "X-" prefix for newly designed application protocols, `X-Forwarded-Prefix` has already achieved universal adoption across production software. Documenting both the widely deployed `X-Forwarded-Prefix` header and the forward-compatible `prefix` parameter for {{RFC7239}} balances immediate interoperability with long-term protocol hygiene.

## The "prefix" Parameter Definition

The `prefix` parameter is defined in ABNF as an extension to the `forwarded-pair` rule of {{RFC7239}}:

~~~ abnf
forwarded-pair =/ "prefix=" prefix-value
prefix-value   = token / quoted-string
~~~
{: title="ABNF Syntax for Forwarded prefix parameter" sourcecode-markers="true"}

Where `prefix-value` MUST conform to `path-absolute` after any unquoting.

### Example

~~~ http
Forwarded: for=198.51.100.1;proto=https;
           host=example.com;prefix="/api/v1"
~~~

Intermediaries and origin servers that implement {{RFC7239}} MAY emit and process the `prefix` parameter. When both `Forwarded: prefix=...` and `X-Forwarded-Prefix` are present on a request from a trusted proxy, the value in the `Forwarded` header field takes precedence.

# Security Considerations {#Security}

## Header Injection and Spoofing

If an intermediary at the network boundary fails to strip or overwrite incoming `X-Forwarded-Prefix` headers from external clients, an attacker can supply an arbitrary prefix:

~~~ http
GET /profile HTTP/1.1
Host: example.com
X-Forwarded-Prefix: /attacker-prefix
~~~

If the upstream origin server trusts this header, the server will incorporate the malicious prefix into generated links, form actions, and redirects, leading to URL hijacking and application malfunction.

Perimeter reverse proxies MUST strip or sanitize untrusted `X-Forwarded-Prefix` headers. Upstream applications MUST NOT trust forwarded headers unless received over a cryptographically or architecturally verified connection from an authorized proxy, applying the atomic trust boundary rule specified in Section 4.1.

## Open Redirect Vulnerabilities

A common vulnerability occurs when an upstream server constructs a redirect target by concatenating an untrusted prefix with a relative path:

~~~
target = header("X-Forwarded-Prefix") + "/login"
response.setHeader("Location", target)
~~~

If an attacker injects a prefix starting with a double solidus (`//`), such as:

~~~ http
X-Forwarded-Prefix: //evil.example.com
~~~

The resulting `Location` header becomes:

~~~ http
Location: //evil.example.com/login
~~~

Modern web browsers interpret URIs beginning with `//` as protocol-relative absolute URIs, redirecting the user to `https://evil.example.com/login`. This constitutes an open redirect vulnerability, which can be leveraged in phishing attacks and credential theft.

To mitigate this risk:

* Reverse proxies MUST NOT emit prefixes starting with `//`.
* Origin servers MUST validate that `X-Forwarded-Prefix` matches the `path-absolute` production and strictly begins with exactly one `/` followed by a non-slash character (or is exactly `/`).
* Any prefix value failing this validation MUST be rejected or ignored.

## HTTP Response and Cache Poisoning

If an origin server generates response content that reflects the `X-Forwarded-Prefix` header (for example, embedding the prefix into HTML asset URLs or API payloads), and an intermediate caching layer (such as a Content Delivery Network or shared cache) caches the response without keying on the prefix, an attacker can trigger cache poisoning.

An attacker could issue a request with `X-Forwarded-Prefix: /malicious`, causing the origin server to generate responses with corrupted links. If the cache stores this response and serves it to legitimate users requesting `/`, all subsequent users will receive corrupted pages.

To prevent cache poisoning:

* Origin servers generating dynamic content based on `X-Forwarded-Prefix` MUST either include `X-Forwarded-Prefix` in the `Vary` response header or instruct intermediaries not to store the response (e.g., `Cache-Control: private, no-cache`).

~~~ http
Vary: X-Forwarded-Prefix
~~~

* Edge proxies and CDNs MUST NOT allow external clients to vary the `X-Forwarded-Prefix` header on public cache keys.

## Server-Side Request Forgery (SSRF) in Client and SSR Frameworks

In frameworks utilizing Server-Side Rendering (SSR), such as Angular Universal or Node.js SSR engines, the server frequently executes client application code to render the initial DOM. If the SSR engine resolves API requests relative to the client-facing base path derived from `X-Forwarded-Prefix`, a spoofed prefix could direct internal HTTP requests to unintended backend services or internal cloud metadata endpoints (e.g., `http://169.254.169.254/`).

As detailed in Angular security guidance {{ANGULAR-SECURITY}}, applications MUST ensure that base URIs derived from forwarded headers are strictly validated against an allowlist of permitted domain names and internal path structures before initiating server-side requests.

## Path Traversal

An attacker might attempt path traversal attacks by injecting `../` segments into the prefix header:

~~~ http
X-Forwarded-Prefix: /api/v1/../../admin
~~~

Intermediaries and upstream origin servers MUST normalize path segments according to Section 5.2.4 of {{RFC3986}} and reject any prefix that attempts to navigate above the root directory.

# IANA Considerations {#IANA}

## HTTP Field Name Registration: X-Forwarded-Prefix

IANA is requested to register the `X-Forwarded-Prefix` header field in the [HTTP Field Name Registry](https://www.iana.org/assignments/http-fields/) in accordance with {{RFC9110}}:

* **Field Name:** `X-Forwarded-Prefix`
* **Status:** `permanent`
* **Structured Type:** List of Strings (see Section 2.2 and {{RFC9651}})
* **Specification Document:** This document
* **Comments:** Conveys the original URI path prefix stripped or rewritten by reverse proxies, API gateways, and load balancers. Classical deployments emit unquoted path-absolute values.

## HTTP Forwarded Parameter Registration: prefix

IANA is requested to register the `prefix` parameter in the [HTTP Forwarded Parameters Registry](https://www.iana.org/assignments/http-parameters/) established by {{RFC7239}}:

* **Parameter Name:** `prefix`
* **Specification Document:** This document
* **Description:** The original URI path prefix removed or rewritten by an intermediary.

--- back

# Existing Implementations and Industry Examples

This non-normative appendix documents the widespread deployment of `X-Forwarded-Prefix` across major platforms, illustrating the de-facto consensus that motivated this specification.

## Java / Spring Framework

In the Java ecosystem, the Spring Framework provides native processing of `X-Forwarded-Prefix` through `ForwardedHeaderFilter` and `UriComponentsBuilder` {{SPRING-WEB}}:

* When `ForwardedHeaderFilter` is enabled, Spring extracts `X-Forwarded-Prefix` and adjusts the servlet request's context path.
* `ServletUriComponentsBuilder.fromCurrentRequest()` automatically prepends the forwarded prefix when constructing self-referential links, HATEOAS hypermedia resources, and redirect targets.
* Spring Cloud Gateway provides built-in filter factories (such as `PrefixPath` and `StripPrefix`) that automatically populate `X-Forwarded-Prefix` when routing requests to downstream microservices.

## Microsoft .NET / ASP.NET Core

ASP.NET Core implements native handling of reverse proxy headers via `Microsoft.AspNetCore.HttpOverrides.ForwardedHeadersMiddleware` {{ASPNET-CORE}}:

* Setting `ForwardedHeadersOptions.ForwardedHeaders` to include `ForwardedHeaders.All` enables processing of `X-Forwarded-Prefix`.
* The middleware extracts the header value and assigns it to `HttpContext.Request.PathBase`.
* Subsequent routing, URL generation (`IUrlHelper.Action`), and redirect helpers incorporate `PathBase`, ensuring consistent path resolution across reverse proxies.

## Red Hat Keycloak

Keycloak, an open-source identity and access management solution, requires reverse proxy headers to function correctly when mounted behind path-rewriting gateways {{KEYCLOAK-PROXY}}:

* When Keycloak is deployed at a subpath (e.g., `/auth`) on a public reverse proxy, but served from `/` internally, the proxy passes `X-Forwarded-Prefix: /auth`.
* Keycloak utilizes this header to establish the frontend base URL, generating correct endpoints in OpenID Connect discovery metadata, login forms, and authorization redirect URIs.

## Google Angular and Frontend Frameworks

Frontend frameworks with Server-Side Rendering (SSR) capabilities, including Angular, rely on proxy headers to reconstruct the public URL and base href {{ANGULAR-SECURITY}}:

* Angular SSR applications use the forwarded prefix to populate the `<base href>` element in rendered HTML.
* Documentation explicitly emphasizes the security boundaries required when consuming `X-Forwarded-Prefix` to prevent Server-Side Request Forgery (SSRF) vulnerabilities in SSR execution environments.

## Go Ecosystem and Routing Proxies

In the Go programming language ecosystem, numerous reverse proxies and routing libraries implement `X-Forwarded-Prefix`:

{:vspace}
Fabio:
: A Consul-aware HTTP routing proxy {{FABIO-LB}} that strips route prefixes and injects `X-Forwarded-Prefix` before passing requests upstream.

Traefik:
: An edge router and ingress controller that includes `X-Forwarded-Prefix` by default when the `StripPrefix` middleware is applied.

Chi and Gin:
: Popular Go HTTP routing frameworks provide middleware libraries to consume `X-Forwarded-Prefix` and update request context paths accordingly.

## Python Web Frameworks

{:vspace}
FastAPI and Starlette:
: Provide the `root_path` configuration parameter specifically designed to consume `X-Forwarded-Prefix`. This enables OpenAPI/Swagger interactive documentation to locate the correct API server endpoints.

Uvicorn:
: An ASGI web server implementation that features a `--proxy-headers` flag, parsing `X-Forwarded-Prefix` to set the ASGI scope `root_path`.

# Acknowledgements
{: numbered="false"}

The author would like to acknowledge the open-source maintainers, web framework architects, and network engineers across the Spring, .NET, Keycloak, Angular, Fabio, and wider Internet engineering communities whose consistent, pragmatic implementation of path-forwarding headers established the foundation for this standard.
