---
tags: [definition, concept, reference]
module: core
last_updated: 2026-05-11
source_count: 0
---

# Authentication Protocols

Authentication mechanisms a pentester encounters — how they work, where they appear, and how they fail.

## NTLM (NT LAN Manager)

**What it is:** Microsoft's challenge-response authentication protocol. Used when Kerberos isn't available (e.g., IP-based auth, non-domain, legacy systems).

**How it works:**
1. Client sends `NEGOTIATE` message
2. Server sends a random 8-byte challenge (`NONCE`)
3. Client hashes its password with the challenge → sends `AUTHENTICATE` (the NTLMv2 response)

**Pentester relevance:**
- NTLMv2 hashes can be captured over the network with [[tools/attack/responder]] (LLMNR poisoning)
- Captured hashes are crackable with Hashcat mode 5600
- Hashes can be *relayed* to authenticate to another host without cracking (if target has no SMB signing)
- NT hash alone can be used for Pass-the-Hash against SMB, RDP, WMI
- See [[attack/smb]] for practical techniques

**Where it appears:** SMB auth over IP, Windows HTTP authentication, RDP (when NLA is off), MSSQL Windows auth, IIS NTLM.

## Kerberos

**What it is:** Ticket-based authentication protocol used by Active Directory. Involves a Key Distribution Center (KDC) that issues tickets.

**How it works:**
1. Client authenticates to the KDC's Authentication Service (AS) → receives a Ticket-Granting Ticket (TGT)
2. Client presents TGT to the Ticket-Granting Service (TGS) to request a service ticket (TGS)
3. Client presents the service ticket to the target service

**Key ports:** TCP/UDP 88 (KDC), TCP/464 (kpasswd)

**Pentester relevance:**
- **Kerberoasting** — request TGS tickets for service accounts (SPNs); tickets are encrypted with the service account's hash → crack offline with Hashcat
- **AS-REP Roasting** — accounts with pre-auth disabled return AS-REP encrypted with the user's hash → crack without sending any credentials
- **Pass-the-Ticket** — steal a TGT or TGS from memory and use it on another host
- **Golden Ticket** — forge a TGT using the KRBTGT account hash; grants unlimited domain access
- **Silver Ticket** — forge a TGS for a specific service using the service account hash

## LDAP (Lightweight Directory Access Protocol)

**What it is:** Protocol for querying and modifying directory services (typically Active Directory).

**Ports:** TCP/UDP 389 (plain), TCP/636 (LDAPS/TLS)

**Pentester relevance:**
- LDAP null binds or anonymous binds can enumerate domain users, groups, GPOs, and hosts
- Credential stuffing against LDAP leaks valid usernames via error messages
- LDAP injection is possible in web apps that construct LDAP queries from user input
- Tools: `ldapsearch`, `ldapdomaindump`, NetExec LDAP module (`nxc ldap`)

## Basic Authentication (HTTP)

**What it is:** HTTP credentials sent as `Base64(username:password)` in the `Authorization` header. No encryption — trivially decoded.

**Pentester relevance:** Credentials are base64 encoded, not encrypted. Sniffing cleartext HTTP reveals plaintext creds instantly. Always try default creds. Used on routers, printers, NAS devices, and legacy web apps.

## Digest Authentication (HTTP)

**What it is:** Challenge-response over HTTP. Credentials are MD5-hashed with a server nonce rather than sent in cleartext.

**Pentester relevance:** More resistant to passive sniffing than Basic, but MD5 is weak — captured digests are crackable. Rarely seen outside legacy or embedded systems.

## OAuth 2.0

**What it is:** Authorization *delegation* framework. Allows a user to grant a third-party application access to their resources without sharing their password.

**Flows:** Authorization Code, Implicit (deprecated), Client Credentials, Device Code

**Pentester relevance:**
- Open redirect vulnerabilities in the `redirect_uri` parameter can steal authorization codes
- PKCE bypass — some implementations don't properly validate the code challenge
- Client credentials stored in mobile apps or JavaScript can be extracted
- Token leakage via `Referer` header, logs, or browser history
- Confused deputy attacks (SSRF chained with OAuth)

## SAML (Security Assertion Markup Language)

**What it is:** XML-based SSO protocol. Identity Provider (IdP) signs an XML assertion; Service Provider (SP) trusts it.

**Pentester relevance:**
- **XML Signature Wrapping (XSW)** — manipulate the unsigned portion of a SAML response while keeping the signature valid
- Weak signature algorithms (SHA-1, RSA with small keys) allow forgery
- Comment injection — some parsers handle `<!--evil-->` inside NameID differently
- If the IdP accepts assertions signed with a self-signed cert, forge them entirely

## JWT (JSON Web Token)

**What it is:** Signed (and optionally encrypted) JSON payload used as a stateless bearer token.

**Structure:** `base64url(header).base64url(payload).signature`

**Pentester relevance:**
- `alg: none` attack — set the algorithm to `none` and remove the signature; some libraries accept it
- `alg: HS256` with RS256 public key — switch from RS256 to HS256 and sign with the public key (if the library blindly trusts the header's `alg`)
- Weak secret cracking — if HS256 with a guessable secret, crack with `hashcat -m 16500`
- Tool: `jwt_tool`

## Related pages

- [[attack/smb]] — NTLM relay and Pass-the-Hash
- [[attack/rdp]] — PTH via xfreerdp
- [[attack/sql_databases]] — Windows auth for MSSQL
- [[definitions/security_terminology]]
- [[ports/common_ports]]
