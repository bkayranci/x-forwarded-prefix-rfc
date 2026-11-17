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

This document specifies the `X-Forwarded-Prefix` HTTP header field, a widely deployed de‑facto standard used by reverse proxies, API gateways, load balancers, and ingress controllers to convey the original URI path prefix that was stripped before forwarding a request.

--- middle

# Introduction

[... truncated ...]