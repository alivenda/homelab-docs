# Identity and access

How the homelab's single sign-on and access control works, and which mode to use for each app.

| | |
|---|---|
| **Components** | Authelia (authentication gateway, OIDC provider) + lldap (lightweight LDAP user store) |
| **Deploy** | [Authelia and lldap](../deploy/authelia.md) |
| **Admin group** | `homelab-admins` (created in lldap; Authelia forwards it in the `groups` claim) |

## The identity stack

Two services form the identity backbone:

- **lldap** — a lightweight LDAP server that stores users and groups. It is the user database Authelia queries when someone logs in.
- **Authelia** — the authentication gateway. It sits behind Traefik and enforces who can access which service, handling 2FA and acting as an OIDC provider.

Every service that needs authentication goes through this stack. The choice of *how* it integrates depends on whether the app has its own login page.

## ForwardAuth versus OIDC { #forwardauth-vs-oidc-read-this-first }

Authelia protects services in two fundamentally different ways. Using the wrong mode causes double-login prompts and breaks API and sync clients:

| Mode | When to use | How it works | Apps using it |
|------|-------------|-------------|----------------|
| **ForwardAuth** | Apps with **no** login page of their own | Traefik intercepts every request, asks Authelia "is this user authenticated?", and either passes the request through or redirects to the Authelia login portal. | Homepage, and any service without its own login screen. |
| **OIDC** | Apps with **their own** user system | The app redirects to Authelia for login, receives a token, and manages its own session — Traefik is not involved in the auth check. | Nextcloud, Forgejo, Paperless-ngx, Vikunja, Actual Budget, Mealie, Audiobookshelf, BookStack, and any app with a built-in user system. |

!!! warning "Never put ForwardAuth in front of an app with its own API clients"
    Nextcloud, Forgejo, and Paperless have desktop sync, Git CLI, or mobile clients that send credentials directly and can't handle an intermediate redirect — ForwardAuth breaks them. Configure those apps as OIDC clients instead; each app's runbook covers its own OIDC setup.

## Choosing the right mode

Use this decision rule:

1. Does the app have a built-in user system with a login page? → **OIDC**.
2. Does the app expose an API that non-browser clients call? → **OIDC** (ForwardAuth breaks those clients).
3. The app has no login page and no API clients → **ForwardAuth**.

Each app's deploy page documents which mode it uses and why. The [Authelia runbook](../deploy/authelia.md) covers the ForwardAuth middleware setup and the OIDC client configuration pattern.
