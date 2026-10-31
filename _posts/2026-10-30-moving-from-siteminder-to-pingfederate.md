---
layout: post
title: "Moving from SiteMinder to PingFederate: What Carries Over and What's Different"
date: 2026-10-30
---

If you know SiteMinder, you already understand most of what PingFederate does. The hard part is not the concepts. It is that the same ideas live in different places and go by different names.

I spent years working with SiteMinder and LDAP, and I am now learning PingFederate. This post is the map I wish I had on day one: what carries over, what changes, and where SiteMinder habits will mislead you.

## The big shift: access management vs. federation

SiteMinder is mainly a web access management product. Its core job is protecting web resources: a web agent sits in front of an application, asks the Policy Server whether this user may reach this URL, and enforces the answer with the SMSESSION cookie.

PingFederate is mainly a federation server. Its core job is issuing and accepting identity tokens across trust boundaries, using standards like SAML, OAuth 2.0 and OpenID Connect. It authenticates the user and tells the application who they are. It does not, by itself, sit in front of every URL and enforce rules.

In the Ping family, the closer match to a SiteMinder web agent is PingAccess, which handles URL-level policy and enforcement. Many SiteMinder migrations end up as PingFederate plus PingAccess, or as apps moved to OIDC so they handle sessions themselves. Knowing this early saves a lot of confusion.

## Concept map

Most SiteMinder building blocks have a rough counterpart. The match is rarely one-to-one, so treat this as a starting point, not a migration spec.

| SiteMinder | Ping equivalent | Notes |
| --- | --- | --- |
| Policy Server | PingFederate server (and PingAccess for URL policy) | Authentication and token issuing in PingFederate; URL enforcement in PingAccess |
| Web Agent | PingAccess agent or gateway | PingFederate has no web agent of its own |
| User Directory (LDAP) | Data store + Password Credential Validator | An LDAP data store holds the connection; the PCV checks passwords against it |
| Authentication scheme (forms, basic) | IdP adapter (for example, HTML Form adapter) | Adapters are how PingFederate collects and verifies credentials |
| Realm, rule, policy | Authentication policies; PingAccess rules | Authentication policies choose which adapters run; access rules live in PingAccess |
| Responses (HTTP headers) | Attribute contract and mapping | Attributes go into SAML assertions, OIDC tokens or headers via PingAccess |
| SMSESSION cookie | PingFederate session + application session | Each app usually keeps its own session after login |
| Federation (Partnerships) | SP connections and IdP connections | In PingFederate this is the main event, not an add-on |

## What carries over

A lot of your SiteMinder experience transfers directly.

- **LDAP knowledge.** Bind DNs, search bases, filters, groups and attribute names work the same way. PingFederate still talks to the same directories.
- **Thinking in attributes.** If you have built SiteMinder responses, you already think about which attributes an app needs and where they come from. That is exactly what attribute mapping is.
- **Troubleshooting instincts.** Reading logs, tracing a login step by step and checking clock skew on assertions all still apply.
- **SAML basics.** If you used SiteMinder Federation, entity IDs, metadata, certificates and assertion signing will feel familiar.

## What is different

These are the places where SiteMinder habits can trip you up.

- **No central URL enforcement.** In SiteMinder, protecting a new app often meant adding a realm and rules. In PingFederate, you create a connection, and the app or PingAccess enforces access.
- **Standards first.** Every integration is SAML, OAuth or OIDC. Apps that relied on SiteMinder headers may need a gateway or a code change.
- **Connections are per partner.** Each application or partner gets its own SP connection with its own attributes, signing settings and endpoints.
- **Sessions are spread out.** There is no single SMSESSION cookie shared by every app. PingFederate keeps its own session, and each app keeps its own. Single logout needs deliberate setup.
- **Expressions and selectors.** Attribute mapping can use OGNL expressions, and authentication policies can branch with selectors. This is more flexible than SiteMinder responses but takes some learning.

## Tips for SiteMinder people learning PingFederate

1. Trace one login end to end. Pick a single SP connection and follow it: which adapter runs, which credential validator checks the password, which attributes come out.
2. Start with the LDAP pieces. Data stores and password credential validators are the most familiar parts, so they are a comfortable entry point.
3. Learn the protocols, not just the screens. A basic understanding of SAML and OIDC flows makes every admin page make sense.
4. Use read-only access to explore. Browsing a real configuration, even without change rights, teaches you how a production setup is organized.
5. Keep a translation sheet. Each time you find a SiteMinder concept's new home, write it down. The table above started that way.

## What's next

The concepts transfer; the architecture changes. Once you see PingFederate as a federation hub rather than a policy server with agents, the rest falls into place.

In the next posts I plan to walk through setting up an LDAP data store and password credential validator, and then how a SAML SP connection maps to what a SiteMinder realm used to do. If you are making the same move, I would like to hear what tripped you up.
