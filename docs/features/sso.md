# Single Sign-On (OIDC, SAML, LDAP)

ShannonStore can hand authentication to an external identity provider, so the
cluster stops being another place that holds passwords and starts being another
thing your directory governs. Three providers are supported — OpenID Connect,
SAML 2.0, and LDAP / Active Directory — and all three work on both planes.

## SSO means two different things here

This is the part most integrations get wrong, so it is worth being precise
before any configuration.

**The admin console is a browser.** It can be redirected, so it uses the flows
built for that: OIDC Authorization Code with PKCE, or SAML 2.0 Web Browser SSO.

**The S3 API is not a browser and cannot redirect anywhere.** An `aws s3` command
has nowhere to display a login page. So the S3 plane federates the way AWS does
it: an application obtains a token from your provider by its own means and
exchanges it through STS, receiving ordinary temporary credentials.

```
  Console                              S3 API
  ───────                              ──────
  browser → provider → back here       app → provider → token
  (Authorization Code + PKCE)                 │
  (SAML Web Browser SSO)                      ▼
                                       AssumeRoleWithWebIdentity
                                       AssumeRoleWithSAML
                                       AssumeRoleWithLDAPIdentity
                                              │
                                              ▼
                                       AccessKeyId / SecretAccessKey / SessionToken
```

Those are the credentials every S3 SDK already understands, which is the point:
**no S3 client needs changing.**

Both paths end at the same token verification and the same group mapping, so an
identity has exactly the permissions on the S3 API that it has in the console.

## What decides permissions

Nothing about the provider does. A federated identity arrives carrying group
names; those are mapped to groups **on this cluster**, and the policies attached
to those groups are what authorize every request. The evaluation is the same code
path as a permanent access key — only where the group list came from differs.

No local user account is created. A federated caller has no password and nothing
to persist; your directory is the record. Creating an account for each person who
ever signed in would put them all in the IAM list and in the replicated snapshot,
and removing them from the directory would leave the copy behind.

### Group mapping

```properties
shannonstore.sso.group.mappings=storage-admins:ss-admins,storage-readers:ss-readers
```

Left empty, provider group names are used as they are — the common case where the
directory already uses this product's group names. **Once set, the mapping is
exhaustive:** a group not named in it is dropped, so creating a group at the
provider cannot grant access here by itself.

An identity whose groups all map to nothing is refused, not admitted with no
groups. Such a session has no policies and is denied every action, so letting it
in produces someone who is signed in and can do nothing, left to work out why
from access-denied errors. Set
`shannonstore.sso.allow.unmapped.groups=true` if you would rather allow it.

## Configuring it

Everything below can be set in the **admin console under Single Sign-On**, which
stores it on the cluster and replicates it to every node — no file edits, no
restart. The properties file is still read for anything left unset, so a cluster
configured by file keeps working untouched.

Local passwords keep working while SSO is on. Enabling it cannot lock you out.

### OpenID Connect

```properties
shannonstore.sso.oidc.enabled=true
shannonstore.sso.oidc.issuer=https://keycloak.example.com/realms/company
shannonstore.sso.oidc.client.id=shannonstore-console
shannonstore.sso.oidc.client.secret=…
shannonstore.sso.oidc.redirect.uri=https://console.example.com/admin/auth/sso/oidc/callback
shannonstore.sso.oidc.groups.claim=groups
```

Endpoints are read from the issuer's discovery document, so they are not
configured individually. The ID token's signature is verified against the
provider's published key set, and its issuer, audience and expiry are all
checked — a token issued for a different application is refused even though it is
genuine and correctly signed.

!!! note "The `groups` scope"
    `shannonstore.sso.oidc.scopes` deliberately does **not** include `groups`. It
    is not a standard scope, and a provider that does not define it rejects the
    whole authorization request with `invalid_scope` — so asking for it by default
    logs nobody in. Group membership comes from a claim the provider is configured
    to include. Add a scope here only if your provider documents one.

### SAML 2.0

A SAML integration is an exchange of metadata, not a form-filling exercise.

1. **Download this cluster's metadata** from the console (or
   `GET /admin/sso/saml/metadata`) and give it to whoever administers your
   identity provider. It carries the entity ID, the assertion consumer URL and,
   if a keypair has been generated, the certificate.
2. **Paste your provider's metadata** into the console. The entity ID, sign-on
   URL and signing certificate are read from it — which beats transcribing three
   fields by hand, where the typos are.

```properties
shannonstore.sso.saml.enabled=true
shannonstore.sso.saml.idp.entity.id=https://idp.example.com/realms/company
shannonstore.sso.saml.idp.sso.url=https://idp.example.com/protocol/saml
shannonstore.sso.saml.idp.certificate=MIIC…
shannonstore.sso.saml.sp.entity.id=shannonstore
shannonstore.sso.saml.sp.acs.url=https://console.example.com/admin/auth/sso/saml/acs
```

Every assertion is checked four ways, and each one is a real attack if skipped:

| Check | What it prevents |
| --- | --- |
| Signature, against the provider's certificate | An assertion the caller wrote |
| Audience | A genuine assertion for another service logging in here |
| Validity window | A captured assertion replayed forever |
| Issuer | Any provider the caller can reach being trusted |

On top of those, an assertion already used is refused: a SAML assertion is a
bearer document, so whoever holds it can present it, and without this one
captured assertion is reusable until it expires.

**Encrypted assertions and signed requests.** Several providers encrypt
assertions or require the authentication request to be signed. Both need a
service-provider keypair — generate one from the console, then re-import the SP
metadata at your provider so it picks up the new certificate. An encrypted
assertion must carry its own signature: encryption proves who the assertion was
*for*, never who wrote it.

**NameID format** is left empty by default, which omits the request entirely and
lets the provider issue whatever it is configured for. Naming one breaks more
integrations than it fixes — several providers refuse a request asking for a
format they do not issue.

### LDAP / Active Directory

```properties
shannonstore.sso.ldap.enabled=true
shannonstore.sso.ldap.url=ldaps://ad.example.com:636
shannonstore.sso.ldap.bind.dn=cn=svc-shannonstore,ou=service,dc=example,dc=com
shannonstore.sso.ldap.bind.password=…
shannonstore.sso.ldap.user.base.dn=ou=people,dc=example,dc=com
shannonstore.sso.ldap.user.filter=(sAMAccountName={0})
shannonstore.sso.ldap.group.base.dn=ou=groups,dc=example,dc=com
shannonstore.sso.ldap.group.filter=(member={0})
```

Authentication is **search then bind**. A service account finds the user's entry
— their DN is something the product cannot construct, since Active Directory puts
people under `CN=John Doe,OU=Staff,…` where neither component is the login name —
and the password is then checked by binding as that DN.

That second bind *is* the authentication. Reading a password attribute and
comparing it would be wrong even where the directory allows it: only the server
knows how its own hashes are salted, and account lockout, expiry and disabled
flags are enforced on bind and nowhere else.

Group membership is read **both ways**: from the user's `memberOf` and from a
search of the group tree. Directories disagree about which side records it —
OpenLDAP usually keeps it on the group, Active Directory mirrors it onto the user
— and reading only one way silently returns no groups against half the servers in
the field.

!!! warning "Use TLS"
    Without `ldaps://` or `shannonstore.sso.ldap.starttls=true`, the bind password
    crosses the network in the clear.

## Behind a load balancer

Both browser flows work on any node, regardless of which node started them. The
login state — including the PKCE verifier — is sealed with a key every node
shares and carried in the `state` parameter itself rather than held in memory on
one node. Unsealing it is also what proves this cluster issued it, which is the
login-CSRF check.

Federated S3 credentials replicate with the IAM snapshot, so they are accepted on
every node and not only the one that minted them.

Neither is a detail: without them SSO works on a single node and fails on roughly
half of all attempts in a cluster.

## Revocation

Disabling someone at the provider stops new logins immediately. Credentials
already issued keep working until they expire — this cluster is not told about
the change. `shannonstore.sso.federated.session.seconds` (default 3600, clamped to
the AWS-compatible 900–43200) bounds how long that gap lasts; shorter is safer.

A console session is not renewable for the same reason: a refresh token would
keep someone signed in after the directory disabled them.

## Password storage

Independent of SSO, local passwords are stored as PBKDF2-HMAC-SHA256 hashes.
Existing plaintext values are upgraded on the owner's next successful login,
which is the only moment the plaintext is available to hash — nobody is locked
out by the change. See `shannonstore.auth.password.hash.iterations` in
[Configuration](configuration.md).

## See also

- [Identity & Access Management](iam.md) — the groups and policies a federated identity maps onto.
- [Authentication & Authorization](auth-authz.md) — the two authentication chains and the STS surface these extend.
- [Configuration](configuration.md) — every setting, with its default.
