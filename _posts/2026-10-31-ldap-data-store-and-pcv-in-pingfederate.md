---
layout: post
title: "Setting Up an LDAP Data Store and Password Credential Validator in PingFederate"
date: 2026-10-31
---

If you come from SiteMinder, the LDAP pieces are the friendliest place to start in PingFederate. You already know how directories, bind accounts and user searches work. Only the screens and names are new.

In PingFederate, checking a username and password against LDAP takes two objects: a **data store**, which holds the connection to the directory, and a **password credential validator (PCV)**, which uses that connection to find the user and verify the password. This post walks through both, then shows where they plug into a login.

Menu names below match recent PingFederate versions. Older versions place them slightly differently, but the fields are the same.

## How it maps to a SiteMinder user directory

In SiteMinder, one User Directory object holds everything: the server, the bind account, the search root and the user lookup. PingFederate splits that into two objects.

| SiteMinder User Directory setting | Where it lives in PingFederate |
| --- | --- |
| Namespace (LDAP:) and server | Data store: type and hostname(s) |
| Administrator credentials (bind DN, password) | Data store: user DN and password |
| Require credentials / SSL | Data store: bind settings and "Use LDAPS" |
| Root (search base) | PCV: search base |
| User lookup start and end (for example, `uid=` ... `,ou=people`) | PCV: search filter, such as `uid=${username}` |
| Attributes returned to responses | PCV: extended contract attributes |

The split has a benefit: one data store can feed several PCVs, adapters and attribute lookups, so you configure the connection once.

## Step 1: Create the LDAP data store

Go to **System → Data & Credential Stores → Data Stores** and click **Add New Data Store**. Choose **Directory (LDAP)** as the type.

The key fields:

- **Hostname(s):** the directory server and port, for example `ldap.example.com:636`. You can list more than one host for failover.
- **LDAP type:** Active Directory, PingDirectory, Oracle Directory Server or Generic. This sets sensible defaults for things like attribute names.
- **Bind:** the service account's user DN and password. Like SiteMinder's administrator credentials, this account only needs read and search rights.
- **Use LDAPS:** turn this on for anything beyond a lab. The directory's certificate (or its issuing CA) must be trusted by PingFederate under **Security → Trusted CAs**.

Click **Test Connection** before saving. If it fails here, nothing downstream will work, so fix it first.

The **Advanced** tab holds connection pool sizes and timeouts. The defaults are fine to start; revisit them under load.

## Step 2: Create the password credential validator

Go to **System → Data & Credential Stores → Password Credential Validators** and click **Create New Instance**. Give it a clear name, such as `LDAP-Employees-PCV`, and choose the type **LDAP Username Password Credential Validator**.

On the configuration screen:

- **LDAP data store:** pick the data store from step 1.
- **Search base:** where users live, for example `ou=people,dc=example,dc=com`. This is SiteMinder's "Root."
- **Search filter:** how to find the user from what they typed. `${username}` is replaced with the login name.
- **Scope of search:** usually Subtree, so users in child OUs are found.

Common search filters:

| Directory | Search filter |
| --- | --- |
| Active Directory (log in with short name) | `sAMAccountName=${username}` |
| Active Directory (log in with email-style name) | `userPrincipalName=${username}` |
| Generic LDAP or PingDirectory | `uid=${username}` |
| Generic LDAP, limited to members of one group | `(&(uid=${username})(memberOf=cn=sso-users,ou=groups,dc=example,dc=com))` |

On the **Extend the Contract** screen, add any attributes you want available after login, such as `mail`, `givenName` or `sn`. This is the closest thing to choosing which attributes feed a SiteMinder response.

Many versions also let you test the PCV with a real username and password. Use a test account, not your own.

## Step 3: Use the PCV in a login

A PCV does nothing on its own. It needs something that collects the username and password, which is usually the **HTML Form Adapter**.

1. Go to **Authentication → Integration → IdP Adapters** and open (or create) an HTML Form Adapter instance.
2. Under **Password Credential Validators**, add the PCV from step 2. You can list several; PingFederate tries them in order.
3. On the adapter's contract, map the attributes you extended in step 2 so they flow on to SP connections.
4. Use the adapter in an **authentication policy** or directly in an SP connection.

Now a login works like this: the form collects the credentials, the PCV searches LDAP with the bind account, binds as the user to check the password, and returns the user's attributes to the adapter. That is the same search-then-bind pattern SiteMinder used.

## Common mistakes

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Test Connection fails with a certificate error | Directory's CA not trusted | Import the CA under Security → Trusted CAs |
| Test Connection times out | Firewall or wrong port (389 vs 636) | Check the port and network path from the PingFederate server |
| Every login fails, even correct passwords | Search filter or search base wrong | Test the same filter with an LDAP browser or `ldapsearch` |
| Some users can log in, others cannot | Users outside the search base, or scope set to One Level | Widen the search base or set scope to Subtree |
| Login works but attributes are empty | Attributes not added to the PCV contract, or not mapped in the adapter | Extend the contract in the PCV and map it in the adapter |
| AD users locked out after a few tries | Users typing the wrong format (email vs short name) | Match the filter to how users actually log in |

For anything else, the PingFederate server log shows the LDAP search and bind results in detail. Reading it is the same skill as reading the SiteMinder Policy Server trace.

## What's next

The data store and PCV are the foundation: once they work, every login method in PingFederate can reuse them. If you set up SiteMinder user directories, you already have the instincts for this part.

Next time I'll look at the other side of the login: creating a SAML SP connection and how it compares to protecting an application with a SiteMinder realm.
