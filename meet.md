# Rayong CMN_PORTAL — Technical Finding Summary

## Affected Asset

**Host:** `rayong-apps.copeland.com`
**Application:** `CMN_PORTAL`
**URL:** `https://rayong-apps.copeland.com/CMN_PORTAL/`
**Backend:** Node.js / Express behind IIS/ARR
**Database:** Microsoft SQL Server
**Database:** `RAYONG_APPS`

---

## What We Found

The application exposed a **generic database API without authentication**:

```text
/CMN_PORTAL/api/db
```

The API accepted a table name through the `table` parameter and provided direct database operations.

Supported operations included:

```text
GET     → Read database records
POST    → Insert records
PATCH   → Modify records
DELETE  → Delete records
```

No authenticated session was required to access these operations.

---

## Sensitive Data Exposed

One of the exposed tables was:

```text
CMN_USERS
```

It contained approximately:

**928 user accounts**

Including information such as:

* User IDs
* Password values
* Names
* Email addresses
* Phone numbers

The password values were stored in a recoverable/plaintext-style format.

The API also exposed:

```text
CMN_PORTAL_APPS_USER
```

Containing approximately:

**9,416 user → application permission records**

This provided visibility into which users had access to internal applications and functionality.

---

## Authentication Impact

The application's authentication flow was also client-side.

The browser retrieved the user's credential information through the exposed API and performed the password comparison in JavaScript.

During validation, we confirmed an end-to-end authentication path using an exposed account.

This means the vulnerability chain was effectively:

```text
Unauthenticated API
        ↓
Access CMN_USERS
        ↓
Obtain user credentials
        ↓
Client-side authentication
        ↓
Authenticate as affected user
        ↓
Access portal functionality
```

This is why the issue was assessed as having **account-takeover impact**, rather than being only a database information disclosure.

---

## Database Integrity Impact

The database API was not read-only.

We verified the complete:

```text
INSERT → UPDATE → DELETE
```

workflow against a reversible test record.

The test record was created, modified, and deleted successfully, and final verification confirmed that the test artifact was removed.

This demonstrates that the exposed API provides **unauthenticated database write capability**, creating both integrity and availability risk.

---

## Additional Exposures

### Password Reset

An additional endpoint was available without authentication:

```text
/CMN_PORTAL/api/password-reset/complete
```

The endpoint accepted a user ID and new password and did not require an authenticated session.

Testing was performed only with a test/fake identity; no real user's password was changed.

### Internal Infrastructure

The following endpoint was also publicly accessible:

```text
/CMN_PORTAL/api/health
```

It disclosed internal information including:

* SQL Server/database name
* Internal server name
* Application port
* Windows filesystem paths

Example internal infrastructure information included:

```text
COPELAND-Server-01\RAYONG_APPS
RAYONG_APPS
C:\RAYONG_APPS\ICON_STORAGE
```

---

## Overall Attack Chain

```text
Internet
   │
   ▼
rayong-apps.copeland.com
   │
   ▼
/CMN_PORTAL/api/db
   │
   ├──► Read CMN_USERS
   │        │
   │        └──► 928 user accounts + credentials
   │
   ├──► Read application permissions
   │        │
   │        └──► 9,416 user → application mappings
   │
   ├──► POST
   │
   ├──► PATCH
   │
   └──► DELETE
   │
   ▼
Client-side authentication
   │
   ▼
Account takeover path
```

---

## Why This Is Critical

The important point is that this was **not a single isolated weakness**.

Multiple weaknesses combined into one attack path:

```text
Unauthenticated API
        +
Sensitive database exposure
        +
Plaintext/recoverable credentials
        +
Client-side authentication
        +
Unauthenticated database writes
        +
Unauthenticated password reset
        =
Critical compromise
```

The most significant finding is that the application's **backend trust boundary was effectively exposed to an unauthenticated internet user**.

---

## What We Want to Understand

The asset was already known to CyCognito.

The key technical question is therefore:

> **At what stage should this type of exposure have been detected?**

Potential detection points include:

1. **Application crawling**

   * Discovery of `/CMN_PORTAL/`

2. **API discovery**

   * Discovery of `/CMN_PORTAL/api/db`

3. **JavaScript analysis**

   * Identification of API endpoints and client-side authentication logic

4. **Unauthenticated API testing**

   * Testing whether sensitive APIs are accessible without credentials

5. **Parameter analysis**

   * Testing the `table` parameter and identifying access to sensitive database objects

6. **Sensitive-data detection**

   * Identifying credentials, employee information, and authorization data in API responses

7. **Authentication analysis**

   * Identifying that authentication depends on data retrieved by the client

8. **Write-operation testing**

   * Determining whether POST/PATCH/DELETE operations are exposed without authorization

---

## Key Takeaway

The asset was successfully identified, but the **critical application-level attack surface underneath the asset was not identified**.

The purpose of discussing this finding is to determine whether:

**CyCognito discovered the application but did not discover the API,**

or:

**the API was discovered but not tested deeply enough,**

or:

**the API was tested but the sensitive-data/authentication/write-access chain was not identified.**

Understanding that boundary will help us determine whether this represents a **coverage gap, detection limitation, configuration issue, or an expected limitation of the current scanning methodology**.
