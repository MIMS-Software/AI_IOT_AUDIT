# 🔐 SECURITY AND CODE QUALITY AUDIT

**Original Audit Date:** December 11, 2025  
**Follow-up Audit Date:** December 21, 2025  
**Methodology:** Comprehensive analysis with ripgrep + Runtime Testing

---

## 🔄 SECURITY UPDATE - DECEMBER 21, 2025

### ⚡ SIGNIFICANT IMPROVEMENTS IN REFACTORED REPOSITORY

The team applied **IMPORTANT FIXES** to critical vulnerabilities:

**📊 CRITICAL VULNERABILITIES STATUS:**

| Vulnerability                   | Original Status (Dec 11) | Current Status (Dec 21) | Change             |
| ------------------------------- | ------------------------ | ----------------------- | ------------------ |
| **JWT without verification**    | 🔴 2/10 CRITICAL         | ⚠️ 7/10 PARTIAL         | ✅ +5 points       |
| **CORS wildcard (\*)**          | 🔴 2/10 CRITICAL         | 🔴 2/10 CRITICAL        | ❌ NOT FIXED       |
| **Weak secret keys**            | 🔴 2/10 CRITICAL         | ✅ 10/10                | ✅ +8 points       |
| **XSS dangerouslySetInnerHTML** | 🔴 2/10 (82 cases)       | ✅ 9/10 (2 cases)       | ✅ -97%            |
| **Rate limiting**               | 🔴 0/10                  | 🔴 0/10                 | ❌ NOT IMPLEMENTED |
| **Tokens in localStorage**      | ⚠️ 4/10                  | ⚠️ 4/10                 | ❌ NO CHANGES      |

### ✅ FIXES APPLIED

1. **JWT Verification** → Multi-issuer pattern implemented (valid but requires discipline)
2. **Secret Keys** → Generated with 512-bit entropy (+300% vs previous 128-bit)
3. **XSS** → Reduced from 82 to 2 occurrences (-97%)
4. **Architecture** → Migrated to TRUE microservices (DB-per-service, gRPC, Kafka)

### 🔴 CRITICAL PENDING

1. **CORS wildcard** → Still allows `origins: ["*"]` - **PRODUCTION BLOCKER**
2. **Rate limiting** → NOT implemented on login endpoints - **PRODUCTION BLOCKER**
3. **27 console.log** → Remaining in frontend (may leak sensitive data)
4. **0% test coverage** → No automated security tests

**🎯 VERDICT:** Security score improved from 3.5/10 → 5.5/10 (+2 points). Still NOT production-ready due to CORS and rate limiting.

**📄 See full details in:** `docs/AUDITORIA_SEGUIMIENTO_DIC_21_2025.md`

---

> **⚠️ NOTE:** Vulnerabilities listed below correspond to the ORIGINAL audit (Dec 11). Many were partially fixed in the refactored repository. Consult follow-up document for current status.

---

## 🚨 CRITICAL VULNERABILITIES (Fix IMMEDIATELY)

### 1. JWT WITHOUT SIGNATURE VERIFICATION - AUTHENTICATION BYPASS ⚠️⚠️⚠️

````
Severity: CRITICAL 🔴
Category: Security - Authentication Bypass
Location: packages/python/mims-shared/mims_shared/middleware/__init__.py:58

❌ PROBLEM:
The middleware decodes JWT without verifying the digital signature.

🔥 RISK:
An attacker can create arbitrary JWT tokens and authenticate as
any user or tenant. THIS IS A COMPLETE AUTHENTICATION BYPASS.

💀 Problematic code:
```python
# Line 58 - CRITICAL VULNERABILITY
payload = jwt.decode(token, options={"verify_signature": False})
````

✅ SOLUTION:

```python
# REMOVE this logic completely
# If you need tenant_id without auth, use header X-Tenant-ID
# Or implement public tenant discovery endpoint
```

### 2. CORS OPEN TO ENTIRE INTERNET (\*)

````
Severity: CRITICAL 🔴
Category: Security - CORS Misconfiguration
Location: 12 services (all main.py)

❌ PROBLEM:
All services allow requests from ANY origin if
CORS_ORIGINS is not defined in .env (default is "*")

🔥 RISK:
- CSRF attacks
- Data exfiltration
- Credential theft from malicious sites

💀 Problematic code:
```python
# services/*/app/main.py
allow_origins=os.getenv("CORS_ORIGINS", "*").split(","),  # ❌ DEFAULT IS "*"
````

✅ SOLUTION:

```python
cors_origins = os.getenv("CORS_ORIGINS")
if not cors_origins or cors_origins == "*":
    if os.getenv("APP_ENV") == "production":
        raise ValueError("CORS_ORIGINS must be explicitly set in production")
    cors_origins = "http://localhost:3000"

app.add_middleware(
    CORSMiddleware,
    allow_origins=cors_origins.split(","),
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)
```

### 3. WEAK SECRET KEYS BY DEFAULT

````
Severity: CRITICAL 🔴
Category: Security - Weak Secrets
Location:
- packages/python/mims-shared/mims_shared/auth/jwt_handler.py:21
- .env.example:52

❌ PROBLEM:
Predictable secret keys and weak defaults

🔥 RISK:
If someone uses these defaults in production, anyone can
sign valid JWTs and impersonate any user

💀 Problematic code:
```python
SECRET_KEY = os.getenv("SECRET_KEY", "mims-tech-super-secret-key-for-development-only-change-in-production")
````

✅ SOLUTION:

```python
import secrets

SECRET_KEY = os.getenv("SECRET_KEY")
if not SECRET_KEY:
    if os.getenv("APP_ENV") == "production":
        raise ValueError("SECRET_KEY must be set in production")
    SECRET_KEY = "dev-only-key-" + secrets.token_hex(32)

if len(SECRET_KEY) < 32:
    raise ValueError("SECRET_KEY must be at least 32 characters")
```

### 4. XSS VIA dangerouslySetInnerHTML WITHOUT SANITIZATION

````
Severity: HIGH 🔴
Category: Security - Cross-Site Scripting (XSS)
Location:
- frontend/src/components/common/PrivacyPolicyDialog.js:79
- frontend/src/components/incidents/CommentsThread.js:274

❌ PROBLEM:
HTML inserted directly without sanitization

🔥 RISK:
An attacker can inject malicious scripts that steal tokens,
credentials, or user sessions

💀 Problematic code:
```javascript
// Backend HTML without sanitization
<Typography dangerouslySetInnerHTML={{ __html: content }} />

// Direct replace without escaping
let processedText = text.replace(/@(\w+)/g, '<span style="color: #1976d2;">@$1</span>');
return <Box dangerouslySetInnerHTML={{ __html: processedText }} />;
````

✅ SOLUTION:

```javascript
// Install: npm install dompurify
import DOMPurify from "dompurify";

<Typography
  dangerouslySetInnerHTML={{
    __html: DOMPurify.sanitize(content, {
      ALLOWED_TAGS: ["p", "strong", "em", "ul", "li", "br"],
      ALLOWED_ATTR: [],
    }),
  }}
/>;
```

### 5. EMPTY EXCEPTION HANDLERS (Hide Errors)

````
Severity: HIGH 🟠
Category: Security & Code Quality
Location:
- services/parking-service/app/api/routers/parking.py:193
- packages/python/mims-shared/mims_shared/database/__init__.py:146
- services/iot-service/app/api/routers/video.py:756

❌ PROBLEM:
except: without specifying exception or logging

🔥 RISK:
Critical errors silenced, makes debugging difficult, can hide vulnerabilities

💀 Problematic code:
```python
try:
    return json.loads(value)
except:  # ❌ Catches EVERYTHING, even KeyboardInterrupt
    return []
````

✅ SOLUTION:

```python
import logging
logger = logging.getLogger(__name__)

try:
    return json.loads(value)
except (json.JSONDecodeError, TypeError) as e:
    logger.warning(f"Failed to parse JSON field: {e}")
    return []
except Exception as e:
    logger.error(f"Unexpected error: {e}")
    raise
```

### 6. SQL INJECTION POTENTIAL

````
Severity: HIGH 🟠
Category: Security - SQL Injection
Location: packages/python/mims-shared/mims_shared/database/__init__.py:188

❌ PROBLEM:
Schema name inserted with f-string in SQL query

🔥 RISK:
If schema_name comes from unvalidated input, can cause SQL injection

💀 Problematic code:
```python
session.execute(text(f"SET search_path TO {schema}, public"))
````

✅ SOLUTION:

```python
import re
if not re.match(r'^tenant_[a-zA-Z0-9_]+$', schema):
    raise ValueError(f"Invalid schema name: {schema}")

from sqlalchemy import literal_column
session.execute(
    text("SET search_path TO :schema, public").bindparams(
        schema=literal_column(schema)
    )
)
```

### 7. JWT TOKENS IN localStorage (XSS Vulnerable)

````
Severity: HIGH 🟠
Category: Security - Token Storage
Location: frontend/src/services/auth.service.js:14-15

❌ PROBLEM:
Access tokens in localStorage are vulnerable to XSS

🔥 RISK:
If there's XSS, the attacker can steal all tokens and
impersonate the user

💀 Problematic code:
```javascript
localStorage.setItem('access_token', access_token);
localStorage.setItem('token_type', token_type);
````

✅ SOLUTION:

```javascript
// Option 1: httpOnly cookies (RECOMMENDED)
// Backend should send:
// Set-Cookie: access_token=xxx; HttpOnly; Secure; SameSite=Strict

// Option 2: If you MUST use localStorage, implement strict CSP
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; script-src 'self';">
```

### 8. NO RATE LIMITING

````
Severity: HIGH 🟠
Category: Security - DoS & Brute Force
Location: All services (absence)

❌ PROBLEM:
No rate limiting on any endpoint

🔥 RISK:
- Brute force attacks on login
- DoS (Denial of Service)
- Credential stuffing
- API abuse

✅ SOLUTION:
```python
# pip install slowapi
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@router.post("/login")
@limiter.limit("5/minute")  # 5 attempts per minute
async def login(request: Request, credentials: UserLogin):
    ...
````

### 9. PASSWORDS IN LOGS (Potential)

````
Severity: MEDIUM-HIGH 🟡
Category: Security - Sensitive Data Exposure
Location: Frontend - 30+ console.log

❌ PROBLEM:
30+ console.log statements in production code

🔥 RISK:
Sensitive data can be logged and exposed

✅ SOLUTION:
```javascript
// package.json
{
  "scripts": {
    "build": "GENERATE_SOURCEMAP=false react-scripts build && terser --compress drop_console=true"
  }
}
````

---

## ⚠️ IMPORTANT VULNERABILITIES (Medium Priority)

### 10. TODO: KMS Mock Implementation

```
Severity: MEDIUM 🟡
Category: Security - Cryptography
Location: packages/python/mims-shared/mims_shared/utils/encryption.py:48

❌ PROBLEM:
Mock KMS implementation instead of real integration

🔥 RISK:
Encryption keys stored locally, no real rotation

✅ SOLUTION:
Implement AWS KMS, Google Cloud KMS or Azure Key Vault
```

### 11. Missing Input Validation

````
Severity: MEDIUM 🟡
Category: Security - Input Validation

❌ PROBLEM:
Some fields don't validate ranges (latitude/longitude, etc.)

✅ SOLUTION:
```python
from pydantic import Field

location_latitude: Optional[float] = Field(None, ge=-90, le=90)
location_longitude: Optional[float] = Field(None, ge=-180, le=180)
````

### 12. No HTTPS Enforcement

```
Severity: MEDIUM 🟡
Category: Security - Transport Security

❌ PROBLEM:
All services expose HTTP without HTTPS

🔥 RISK:
Man-in-the-middle attacks, credential sniffing

✅ SOLUTION:
Implement TLS in Kong Gateway or use reverse proxy with SSL
```

### 13. Database Credentials in Docker Compose

```
Severity: MEDIUM 🟡
Category: Security - Secrets Management
Location: docker-compose.yml:44-45

❌ PROBLEM:
Hardcoded passwords with weak defaults

✅ SOLUTION:
Use Docker Secrets or Vault
```

### 14. Missing CSRF Protection

````
Severity: MEDIUM 🟡
Category: Security - CSRF

❌ PROBLEM:
No CSRF tokens implemented

🔥 RISK:
Cross-Site Request Forgery attacks

✅ SOLUTION:
```python
from fastapi_csrf_protect import CsrfProtect

@app.post("/endpoint")
async def endpoint(csrf_protect: CsrfProtect = Depends()):
    await csrf_protect.validate_csrf(request)
````

### 15. Stack Traces in Responses

````
Severity: MEDIUM 🟡
Category: Security - Information Disclosure

❌ PROBLEM:
Stack traces and internal details exposed

✅ SOLUTION:
```python
@app.exception_handler(Exception)
async def generic_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled exception: {exc}", exc_info=True)

    if os.getenv("APP_ENV") == "production":
        return JSONResponse(
            status_code=500,
            content={"detail": "Internal server error"}
        )
````

### 16-20. Other Important Issues

```
16. Lack of Health Checks in Python services
17. No Resource Limits in containers
18. Very long functions (>100 lines)
19. Missing Type Hints
20. Frontend Dependencies without audit (npm audit)
```

---

## 💡 SUGGESTED IMPROVEMENTS (Low Priority)

```
21. Console.log in production (30+ occurrences)
22. Unresolved TODOs (50+)
23. Missing docstrings
24. Magic numbers without constants
25. Dependency versions not pinned
26. No documented backup strategy
27. Elasticsearch without authentication
```

---

## ✅ GOOD PRACTICES FOUND

1. ✅ **Well-Designed Microservices Architecture**
2. ✅ **Multi-Tenancy Implemented** (Hybrid Bridge/Silo)
3. ✅ **Field-Level Encryption** implemented
4. ✅ **Pydantic Models** for validation
5. ✅ **Alembic Migrations** configured
6. ✅ **Shared Library** (mims-shared) well organized
7. ✅ **Complete Docker Compose** for development
8. ✅ **Environment Variables** correctly used
9. ✅ **Password Hashing** with bcrypt
10. ✅ **JWT with Expiration** configured
11. ✅ **Consistent Code Style**
12. ✅ **Centralized API Gateway (Kong)**
13. ✅ **Health Checks** in infrastructure

---

## 📊 SECURITY AND QUALITY SCORES

### 🔐 Security Score: **3.5/10** ⚠️ CRITICAL

**Justification:**

- JWT without signature verification: **AUTHENTICATION BYPASS** 🔴
- Open CORS: **DATA EXFILTRATION RISK** 🔴
- XSS vulnerabilities: **SESSION HIJACKING RISK** 🔴
- No rate limiting: **BRUTE FORCE VULNERABLE** 🔴
- Tokens in localStorage: **XSS TOKEN THEFT** 🔴
- SQL injection potential: **DATA BREACH RISK** 🔴

**⛔ PRODUCTION BLOCKERS:**

1. JWT without verification (fix IMMEDIATELY)
2. CORS wildcard (configure explicit origins)
3. XSS in comments (sanitize with DOMPurify)
4. Rate limiting (implement slowapi)
5. Secret keys (validate length >32 on startup)

### 🏗️ Quality Score: **6.5/10** 🟡 IMPROVABLE

**Positive aspects:**

- ✅ Good microservices architecture
- ✅ Correct use of Pydantic and SQLAlchemy
- ✅ Generally clean and organized code
- ✅ Separation of concerns

**Negative aspects:**

- ❌ Empty exception handlers
- ❌ Very long files (2000+ lines)
- ❌ 50+ unresolved TODOs
- ❌ Functions without complete type hints

---

## 📋 ISSUE SUMMARY

| Category     | Critical | Important | Improvements | Total  |
| ------------ | -------- | --------- | ------------ | ------ |
| Security     | 9        | 7         | 3            | 19     |
| Code Quality | 0        | 4         | 4            | 8      |
| **TOTAL**    | **9**    | **11**    | **7**        | **27** |

---

## 🚨 IMMEDIATE ACTION PLAN

### BEFORE DEPLOYING TO PRODUCTION

```
🔴 Priority 1 - Non-negotiable:
1. Remove verify_signature=False in middleware
2. Configure CORS with explicit origins
3. Validate SECRET_KEY on startup (>32 chars, no default)
4. Implement rate limiting on login
5. Sanitize HTML with DOMPurify
6. Move tokens to httpOnly cookies or strict CSP

Estimated time: 3-5 days
Responsible: Security team
```

```
🟠 Priority 2 - Important:
7. Implement CSRF protection
8. Add logging to exception handlers
9. Validate inputs (latitude, longitude, etc.)
10. Implement health checks in services
11. Configure resource limits in Docker

Estimated time: 5 days
Responsible: Backend team
```

```
🟡 Priority 3 - Improve:
12. Integrate real KMS (AWS/GCP/Azure)
13. Implement HTTPS in Kong
14. Docker Secrets
15. npm audit fix
16. Refactor long functions

Estimated time: 5 days
Responsible: DevOps + Backend team
```

```
⚪ Priority 4 - Nice to have:
17-27. Code improvements, documentation, etc.

Estimated time: 2-3 weeks
Responsible: Full team
```

---

## 🎯 FINAL SECURITY VERDICT

### ⛔ NOT READY FOR PRODUCTION

This project has a **solid architecture** and demonstrates **good security intentions** (encryption, multi-tenancy, JWT), but has **critical vulnerabilities** that make it **NOT ready for production** in its current state.

### ✅ Can be Production-Ready in 1-2 Weeks

The critical issues are **relatively easy to fix** (days, not weeks), but are **BLOCKERS**. The JWT without signature verification is especially concerning because it completely invalidates authentication security.

### 📈 Security Roadmap

```
Day 1-3:   Fix JWT, CORS, Secret Keys          ✅ Unblocked
Day 4-5:   XSS, Rate Limiting, Tokens           ✅ Secure
Day 6-10:  CSRF, Logging, Health Checks         ✅ Robust
Day 11-15: KMS, HTTPS, Docker Secrets           ✅ Production-Ready
```

**Total timeline: 2-3 weeks for secure production-ready** 🚀

---

## 📞 FINAL RECOMMENDATIONS

1. **Hire external security audit** before go-live
2. **Penetration testing** after fixing critical issues
3. **Bug bounty program** post-launch
4. **Security training** for the team
5. **Mandatory code review** for auth/security changes

---

## 🧪 NEXT STEPS: COMPREHENSIVE TESTING STRATEGY

This security audit identifies **9 CRITICAL vulnerabilities** that must be fixed immediately. However, fixing vulnerabilities is only half the battle - you MUST implement **security testing** to prevent regressions and ensure these issues never return.

### Recommended Approach

1. **Fix vulnerabilities** (Week 1-2) - See "IMMEDIATE ACTION PLAN" above
2. **Write security tests** to validate fixes and prevent regressions
3. **Implement comprehensive testing strategy** for entire project

For a complete testing strategy including:

- ✅ Security Testing (CRITICAL - Priority 2)
- ✅ Contract Testing (prevents breaking changes between microservices)
- ✅ Integration Testing (validates business flows)
- ✅ E2E Testing (validates user experience)
- ✅ Performance Testing (optional)

**See:** [TEST_STRATEGY_EN.md](./TEST_STRATEGY_EN.md) - Comprehensive testing roadmap prioritized by ROI

### Why Testing Strategy Matters

Without proper security testing:

- ❌ Vulnerabilities can be reintroduced during refactoring
- ❌ New code can introduce similar vulnerabilities
- ❌ No automated detection in CI/CD pipeline
- ❌ No confidence in production deployments

With security testing:

- ✅ Automated validation on every PR
- ✅ Prevents regressions
- ✅ Catches vulnerabilities before production
- ✅ Enables confident deployments

---
