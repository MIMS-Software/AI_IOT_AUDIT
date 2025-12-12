# 🧪 TESTING STRATEGY - AI_IOT MIMS MICROSERVICES

**Date:** December 11, 2025  
**Project:** AI_IOT Multi-Tenant Intelligent Management System  
**Context:** 12 microservices, 394 endpoints, 110K lines - Current testing coverage: ~20%  
**Production Timeline:** 2-3 months

---

## 🎯 EXECUTIVE SUMMARY

### Current State: 20% Testing Coverage (CRITICAL GAP)

This project has **ZERO unit tests**, **ZERO integration tests**, and **ZERO E2E tests**. This is NOT acceptable for a production system handling IoT devices, security incidents, and GDPR-compliant data.

### WHY Testing Strategy BEFORE 100% Coverage?

Because **NOT ALL TESTS PROVIDE EQUAL VALUE**. Writing tests "for coverage" wastes time. We need **strategic testing prioritized by ROI (Return on Investment)**.

---

## 🚨 TESTING ANTI-PATTERNS TO AVOID

### ❌ DON'T DO THIS:
- "Let's get 100% unit test coverage" → Waste of time in microservices
- Testing CRUD endpoints → No business value
- Testing getters/setters → Vanity metrics
- Writing tests AFTER code is done → Tests become validation, not design tool

### ✅ DO THIS:
- Test **critical business flows** (authentication, incident creation, notifications)
- Test **service contracts** between microservices (prevents breaking changes)
- Test **security vulnerabilities** (rate limiting, JWT validation, XSS)
- Test **complex business logic** (calculations, validations, state machines)

---

## 📊 TESTING PYRAMID FOR MICROSERVICES

```
        ╱╲
       ╱  ╲          E2E Tests (10-15 tests)
      ╱────╲         - Slow, brittle, expensive
     ╱      ╲        - Only critical happy paths
    ╱────────╲       
   ╱          ╲      Integration Tests (50-100 tests)
  ╱────────────╲     - Medium speed, high value
 ╱              ╲    - Cross-service flows
╱────────────────╲   
╲                ╱   Contract Tests (100% inter-service)
 ╲──────────────╱    - CRITICAL for microservices
  ╲────────────╱     - Prevents breaking changes
   ╲──────────╱      
    ╲────────╱       Unit Tests (Only complex logic)
     ╲──────╱        - Fast, low value in microservices
      ╲────╱         - NOT for CRUD/controllers
       ╲──╱
        ╲╱
```

**Key Insight:** In microservices, **Integration and Contract tests provide MORE value** than unit tests.

---

## 🔥 PRIORITY-BASED TEST STRATEGY

### Priority 1: CONTRACT TESTING (CRITICAL) 🔴

**Timeline:** Weeks 1-3  
**Effort:** 2-3 weeks  
**ROI:** CRITICAL - Prevents microservices from breaking each other  
**Coverage Target:** 100% of inter-service communication

#### WHY FIRST?

You have **12 microservices** talking to each other. If `identity-service` changes an endpoint and breaks `incident-service`, your system collapses in production.

#### Tools:
- **Pact** (Python: `pact-python`, Frontend: `@pact-foundation/pact`)
- **Spring Cloud Contract** (if migrating to Java)

#### What to Test:

**Example 1: identity-service (provider) ↔ incident-service (consumer)**

```python
# Contract: POST /api/v1/auth/verify-token must return user_id and roles

from pact import Consumer, Provider

pact = Consumer("incident-service").has_pact_with(Provider("identity-service"))

@pact.given('valid token exists')
@pact.upon_receiving('token verification request')
@pact.with_request(
    method='POST', 
    path='/api/v1/auth/verify-token',
    body={'token': 'valid-jwt-token'}
)
@pact.will_respond_with(200, body={
    'user_id': 123,
    'roles': ['admin'],
    'tenant_id': 'tenant_acme'
})
def test_verify_token_contract():
    # Consumer validates that provider fulfills contract
    response = incident_service_client.verify_token('valid-jwt-token')
    assert response['user_id'] == 123
    assert 'admin' in response['roles']
```

**Example 2: incident-service (provider) ↔ notification-service (consumer)**

```python
# Contract: POST /api/v1/incidents must return incident_id

@pact.given('authenticated user with property access')
@pact.upon_receiving('create incident request')
@pact.with_request(
    method='POST',
    path='/api/v1/incidents',
    body={'title': 'Fire on Floor 3', 'severity': 'HIGH', 'property_id': 1}
)
@pact.will_respond_with(201, body={
    'id': 1,
    'title': 'Fire on Floor 3',
    'status': 'OPEN',
    'created_at': '2025-12-11T10:00:00Z'
})
def test_create_incident_contract():
    response = notification_service_client.create_incident({...})
    assert response['id'] is not None
    assert response['status'] == 'OPEN'
```

#### Critical Contracts to Implement:

| Provider Service    | Consumer Service    | Endpoint Contract                       |
|---------------------|---------------------|-----------------------------------------|
| identity-service    | ALL services        | POST /auth/verify-token                 |
| property-service    | incident-service    | GET /properties/{id}                    |
| incident-service    | notification-service| POST /incidents (webhook)               |
| incident-service    | audit-service       | GET /incidents/{id}                     |
| iot-service         | incident-service    | POST /sensors/events                    |
| analytics-service   | incident-service    | GET /incidents (bulk query)             |

#### Implementation Roadmap:

**Week 1:** Setup Pact broker + CI/CD integration  
**Week 2:** Implement top 10 critical contracts (auth, incidents, properties)  
**Week 3:** Complete remaining contracts + documentation

---

### Priority 2: SECURITY TESTING (CRITICAL) 🔴

**Timeline:** Weeks 1-2  
**Effort:** 1-2 weeks  
**ROI:** BLOCKER - Cannot go to production without fixing 9 critical vulnerabilities  
**Coverage Target:** 100% of critical vulnerabilities documented in [SECURITY_AUDIT.md](./SECURITY_AUDIT_EN.md)

#### WHY SECOND?

Because you have **9 CRITICAL security vulnerabilities** that are **production blockers**:
1. JWT without signature verification (AUTHENTICATION BYPASS)
2. CORS wildcard (*)
3. Weak default secret keys
4. XSS via dangerouslySetInnerHTML
5. No rate limiting
6. Tokens in localStorage
7. SQL injection potential
8. Empty exception handlers
9. Passwords in logs

#### Tools:
- **Bandit** (Python SAST - Static Analysis)
- **OWASP ZAP** (DAST - Dynamic Analysis)
- **pytest-security** (Automated security tests)
- **Semgrep** (Pattern-based vulnerability detection)
- **npm audit** (Frontend dependency vulnerabilities)

#### What to Test:

**Test 1: JWT Signature Validation (CRITICAL)**

```python
import pytest
import jwt
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_jwt_must_reject_invalid_signature():
    """
    CRITICAL: JWT with invalid signature MUST be rejected.
    Current vulnerability: middleware uses verify_signature=False
    """
    # Create token with wrong signature
    fake_token = jwt.encode(
        {'user_id': 999, 'tenant_id': 'tenant_hacker'},
        key='wrong-secret',
        algorithm='HS256'
    )
    
    response = client.get(
        "/api/v1/protected-resource",
        headers={"Authorization": f"Bearer {fake_token}"}
    )
    
    # MUST return 401 Unauthorized
    assert response.status_code == 401
    assert "Invalid token" in response.json()["detail"]
    
def test_jwt_must_reject_expired_token():
    """Expired tokens must be rejected"""
    expired_token = create_expired_jwt()
    
    response = client.post(
        "/api/v1/incidents",
        headers={"Authorization": f"Bearer {expired_token}"}
    )
    
    assert response.status_code == 401
    assert "Token expired" in response.json()["detail"]
```

**Test 2: Rate Limiting (CRITICAL)**

```python
def test_login_rate_limiting():
    """
    Login endpoint must have rate limiting (5 attempts/minute).
    Current vulnerability: No rate limiting implemented
    """
    # Attempt 101 logins (should be limited to 100/minute)
    responses = []
    for i in range(101):
        response = client.post("/api/v1/auth/login", json={
            "email": f"test{i}@test.com",
            "password": "password123"
        })
        responses.append(response)
    
    # Last request should be rate limited
    assert responses[-1].status_code == 429  # Too Many Requests
    assert "Rate limit exceeded" in responses[-1].json()["detail"]

def test_rate_limiting_per_ip():
    """Rate limiting must be per IP address"""
    # Simulate requests from different IPs
    for ip in ['192.168.1.1', '192.168.1.2']:
        response = client.post(
            "/api/v1/auth/login",
            headers={"X-Forwarded-For": ip},
            json={"email": "test@test.com", "password": "password123"}
        )
        # Each IP should have independent rate limit
        assert response.status_code in [200, 401]  # Not 429 on first request
```

**Test 3: XSS Prevention (HIGH)**

```python
import pytest
from bs4 import BeautifulSoup

def test_xss_prevention_in_comments():
    """
    Comments with <script> tags must be sanitized.
    Current vulnerability: dangerouslySetInnerHTML without DOMPurify
    """
    malicious_comment = {
        "text": "Nice incident! <script>alert('XSS')</script>",
        "incident_id": 1
    }
    
    response = client.post(
        "/api/v1/incidents/1/comments",
        json=malicious_comment,
        headers=auth_headers
    )
    
    assert response.status_code == 201
    
    # Verify script tag was removed/escaped
    saved_comment = response.json()["text"]
    assert "<script>" not in saved_comment
    assert "alert" not in saved_comment or "&lt;script&gt;" in saved_comment

def test_xss_prevention_in_incident_title():
    """Incident titles must be sanitized"""
    malicious_incident = {
        "title": "<img src=x onerror=alert('XSS')>",
        "severity": "HIGH",
        "property_id": 1
    }
    
    response = client.post("/api/v1/incidents", json=malicious_incident)
    assert response.status_code == 201
    
    # Script should be escaped
    assert "<img" not in response.json()["title"] or \
           "&lt;img" in response.json()["title"]
```

**Test 4: SQL Injection Prevention (HIGH)**

```python
def test_sql_injection_in_search():
    """
    Search queries must not be vulnerable to SQL injection.
    Current vulnerability: f-string in schema name
    """
    malicious_search = "'; DROP TABLE incidents; --"
    
    response = client.get(
        f"/api/v1/incidents/search?q={malicious_search}",
        headers=auth_headers
    )
    
    # Should return empty results or error, NOT execute SQL
    assert response.status_code in [200, 400]
    
    # Verify table still exists
    check_response = client.get("/api/v1/incidents", headers=auth_headers)
    assert check_response.status_code == 200
```

**Test 5: CORS Configuration (CRITICAL)**

```python
def test_cors_must_not_allow_wildcard():
    """
    CORS must NOT accept wildcard (*) in production.
    Current vulnerability: Default CORS_ORIGINS="*"
    """
    import os
    
    # Simulate production environment
    os.environ['APP_ENV'] = 'production'
    os.environ['CORS_ORIGINS'] = '*'
    
    with pytest.raises(ValueError, match="CORS_ORIGINS must be explicitly set"):
        from app.main import app  # Should fail on startup

def test_cors_allows_only_whitelisted_origins():
    """CORS should only allow configured origins"""
    response = client.options(
        "/api/v1/incidents",
        headers={"Origin": "https://evil-site.com"}
    )
    
    # Should NOT include evil-site.com in allowed origins
    assert "https://evil-site.com" not in \
           response.headers.get("Access-Control-Allow-Origin", "")
```

#### Security Test Automation with Bandit:

```bash
# Run Bandit on all Python services
bandit -r services/ -f json -o security-report.json

# Critical checks:
# - B105: Hardcoded password
# - B201: Flask debug mode
# - B301: Pickle usage
# - B506: YAML load without safe loader
# - B608: SQL injection
```

#### Implementation Roadmap:

**Week 1:**
- Day 1-2: Fix JWT verification + write tests
- Day 3-4: Implement rate limiting + write tests
- Day 5: XSS prevention + write tests

**Week 2:**
- Day 1-2: SQL injection fixes + tests
- Day 3: CORS configuration + tests
- Day 4-5: Bandit + OWASP ZAP scans + remediation

---

### Priority 3: INTEGRATION TESTING (HIGH) 🟡

**Timeline:** Weeks 4-7  
**Effort:** 3-4 weeks  
**ROI:** HIGH - Validates complete business flows across services  
**Coverage Target:** Top 20 critical flows (80/20 rule)

#### WHY THIRD?

Integration tests validate **end-to-end business flows** that cross multiple services:
- User authentication → Token → Access protected resources
- Create incident → Trigger notification → Register in audit log
- IoT sensor event → Incident creation → Dashboard update

#### Tools:
- **pytest** with database fixtures
- **TestContainers** (Docker containers for tests)
- **FastAPI TestClient**
- **pytest-asyncio** (for async endpoints)

#### What to Test:

**Test 1: Complete Incident Creation Flow**

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def test_db():
    """Setup test database with Docker"""
    with PostgresContainer("postgres:15") as postgres:
        engine = create_engine(postgres.get_connection_url())
        # Run migrations
        run_alembic_migrations(engine)
        yield engine

@pytest.fixture
def auth_headers(test_db):
    """Create authenticated user and return auth headers"""
    user = create_test_user(test_db, email="test@test.com", role="admin")
    token = generate_jwt_token(user)
    return {"Authorization": f"Bearer {token}"}

@pytest.mark.integration
def test_create_incident_triggers_notification_and_audit(
    test_db, 
    auth_headers,
    mock_notification_service
):
    """
    Integration test: Creating incident should:
    1. Save to database
    2. Trigger notification service
    3. Create audit log entry
    """
    # 1. Create incident
    incident_data = {
        "title": "Fire on Floor 3",
        "severity": "HIGH",
        "property_id": 1,
        "location_details": "Building A, Room 301"
    }
    
    response = client.post(
        "/api/v1/incidents",
        json=incident_data,
        headers=auth_headers
    )
    
    assert response.status_code == 201
    incident = response.json()
    incident_id = incident["id"]
    
    # 2. Verify notification service was called
    mock_notification_service.send_notification.assert_called_once()
    call_args = mock_notification_service.send_notification.call_args
    assert call_args[0]['incident_id'] == incident_id
    assert call_args[0]['severity'] == 'HIGH'
    
    # 3. Verify audit log was created
    audit_logs = test_db.query(AuditLog).filter_by(
        action="CREATE_INCIDENT",
        resource_id=incident_id
    ).all()
    
    assert len(audit_logs) == 1
    assert audit_logs[0].user_id == 123  # From auth token
    assert audit_logs[0].changes['title'] == "Fire on Floor 3"
    
    # 4. Verify incident is retrievable
    get_response = client.get(
        f"/api/v1/incidents/{incident_id}",
        headers=auth_headers
    )
    assert get_response.status_code == 200
    assert get_response.json()["title"] == "Fire on Floor 3"
```

**Test 2: Authentication Flow**

```python
@pytest.mark.integration
def test_complete_authentication_flow(test_db):
    """
    Test complete auth flow:
    1. User registers
    2. User logs in
    3. JWT token is issued
    4. Token grants access to protected resources
    5. Token expires after timeout
    """
    # 1. Register user
    register_response = client.post("/api/v1/auth/register", json={
        "email": "newuser@test.com",
        "password": "SecurePass123!",
        "tenant_id": "tenant_acme"
    })
    assert register_response.status_code == 201
    
    # 2. Login
    login_response = client.post("/api/v1/auth/login", json={
        "email": "newuser@test.com",
        "password": "SecurePass123!"
    })
    assert login_response.status_code == 200
    
    # 3. Extract token
    access_token = login_response.json()["access_token"]
    assert access_token is not None
    
    # 4. Access protected resource
    protected_response = client.get(
        "/api/v1/incidents",
        headers={"Authorization": f"Bearer {access_token}"}
    )
    assert protected_response.status_code == 200
    
    # 5. Verify token expires (simulate time travel)
    with freeze_time(datetime.now() + timedelta(hours=25)):
        expired_response = client.get(
            "/api/v1/incidents",
            headers={"Authorization": f"Bearer {access_token}"}
        )
        assert expired_response.status_code == 401
```

**Test 3: Multi-Tenancy Isolation**

```python
@pytest.mark.integration
def test_multi_tenancy_data_isolation(test_db):
    """
    Verify that tenants cannot access each other's data.
    Critical for GDPR compliance.
    """
    # Create two tenants
    tenant_a_user = create_test_user(test_db, tenant="tenant_a")
    tenant_b_user = create_test_user(test_db, tenant="tenant_b")
    
    token_a = generate_jwt_token(tenant_a_user)
    token_b = generate_jwt_token(tenant_b_user)
    
    # Tenant A creates incident
    incident_a = client.post("/api/v1/incidents", json={
        "title": "Tenant A Incident",
        "severity": "LOW",
        "property_id": 1
    }, headers={"Authorization": f"Bearer {token_a}"}).json()
    
    # Tenant B creates incident
    incident_b = client.post("/api/v1/incidents", json={
        "title": "Tenant B Incident",
        "severity": "HIGH",
        "property_id": 2
    }, headers={"Authorization": f"Bearer {token_b}"}).json()
    
    # Tenant A should NOT see Tenant B's incident
    tenant_a_incidents = client.get(
        "/api/v1/incidents",
        headers={"Authorization": f"Bearer {token_a}"}
    ).json()
    
    incident_ids = [inc['id'] for inc in tenant_a_incidents]
    assert incident_a['id'] in incident_ids
    assert incident_b['id'] not in incident_ids  # MUST NOT leak
    
    # Tenant A should NOT access Tenant B's incident by ID
    forbidden_response = client.get(
        f"/api/v1/incidents/{incident_b['id']}",
        headers={"Authorization": f"Bearer {token_a}"}
    )
    assert forbidden_response.status_code == 404  # Or 403
```

#### Top 20 Critical Flows to Test:

1. ✅ User authentication flow (register → login → access)
2. ✅ Incident creation → Notification → Audit
3. ✅ Multi-tenancy data isolation
4. Property management (create → assign → list)
5. Visitor check-in flow (QR code → validation → access granted)
6. Emergency incident escalation (HIGH → CRITICAL → Notify authorities)
7. IoT sensor event → Incident auto-creation
8. Analytics dashboard data aggregation
9. GDPR data export (user requests their data)
10. GDPR data deletion (right to be forgotten)
11. User role-based access control (admin vs resident vs staff)
12. Parking reservation flow
13. Compliance report generation
14. Real-time WebSocket updates
15. Incident comment thread (with mentions @user)
16. File upload → Virus scan → Storage
17. Password reset flow (email → token → new password)
18. Two-factor authentication (if implemented)
19. Audit log trail (every action recorded)
20. System health check cascade

#### Implementation Roadmap:

**Week 4:** Setup TestContainers + fixtures  
**Week 5:** Implement flows 1-10  
**Week 6:** Implement flows 11-20  
**Week 7:** Refine + CI/CD integration

---

### Priority 4: END-TO-END TESTING (MEDIUM) 🟢

**Timeline:** Weeks 8-9  
**Effort:** 2-3 weeks  
**ROI:** MEDIUM - Validates user experience but slow and brittle  
**Coverage Target:** 10-15 critical happy paths ONLY

#### WHY FOURTH?

E2E tests validate the **complete user experience** (Frontend + Backend), but they are:
- **SLOW** (take minutes vs seconds for unit tests)
- **BRITTLE** (break easily with UI changes)
- **EXPENSIVE** (require browser automation infrastructure)

Use them ONLY for **critical happy paths**, not edge cases.

#### Tools:
- **Playwright** (RECOMMENDED - better for microservices than Cypress)
- **Selenium** (if cross-browser testing is critical)
- **Docker Compose** (for full system E2E)

#### What to Test:

**Test 1: User Can Create and View Incident**

```javascript
// tests/e2e/incident-flow.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Incident Management', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    await page.goto('http://localhost:3000/login');
    await page.fill('input[name="email"]', 'admin@test.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    
    // Wait for redirect to dashboard
    await expect(page).toHaveURL('http://localhost:3000/dashboard');
  });

  test('User can create incident and see it in list', async ({ page }) => {
    // 1. Navigate to create incident
    await page.click('a[href="/incidents/new"]');
    await expect(page).toHaveURL('http://localhost:3000/incidents/new');
    
    // 2. Fill form
    await page.fill('input[name="title"]', 'E2E Test Incident');
    await page.selectOption('select[name="severity"]', 'HIGH');
    await page.selectOption('select[name="property_id"]', '1');
    await page.fill('textarea[name="description"]', 'This is a test incident');
    
    // 3. Submit form
    await page.click('button[type="submit"]');
    
    // 4. Verify success message
    await expect(page.locator('text=Incident created successfully')).toBeVisible();
    
    // 5. Navigate to incidents list
    await page.click('a[href="/incidents"]');
    
    // 6. Verify incident appears in list
    await expect(page.locator('text=E2E Test Incident')).toBeVisible();
    
    // 7. Click on incident to view details
    await page.click('text=E2E Test Incident');
    
    // 8. Verify details page
    await expect(page.locator('h1:has-text("E2E Test Incident")')).toBeVisible();
    await expect(page.locator('text=HIGH')).toBeVisible();
  });

  test('User can add comment to incident', async ({ page }) => {
    // Assumes incident exists from previous test or fixture
    await page.goto('http://localhost:3000/incidents/1');
    
    // Scroll to comments section
    await page.locator('textarea[placeholder*="comment"]').scrollIntoViewIfNeeded();
    
    // Add comment
    await page.fill('textarea[placeholder*="comment"]', 'E2E test comment');
    await page.click('button:has-text("Post Comment")');
    
    // Verify comment appears
    await expect(page.locator('text=E2E test comment')).toBeVisible({
      timeout: 5000
    });
  });
});
```

**Test 2: Complete Login to Dashboard Flow**

```javascript
test('User can login and see dashboard', async ({ page }) => {
  // 1. Navigate to login
  await page.goto('http://localhost:3000/login');
  
  // 2. Enter credentials
  await page.fill('input[name="email"]', 'admin@test.com');
  await page.fill('input[name="password"]', 'password123');
  
  // 3. Click login
  await page.click('button[type="submit"]');
  
  // 4. Verify redirect to dashboard
  await expect(page).toHaveURL('http://localhost:3000/dashboard');
  
  // 5. Verify dashboard widgets load
  await expect(page.locator('text=Total Incidents')).toBeVisible();
  await expect(page.locator('text=Active Properties')).toBeVisible();
  await expect(page.locator('text=Recent Activity')).toBeVisible();
  
  // 6. Verify navigation menu
  await expect(page.locator('a[href="/incidents"]')).toBeVisible();
  await expect(page.locator('a[href="/properties"]')).toBeVisible();
  
  // 7. Verify user info in header
  await expect(page.locator('text=admin@test.com')).toBeVisible();
});
```

**Test 3: Visual Regression Testing**

```javascript
test('Dashboard visual regression', async ({ page }) => {
  await page.goto('http://localhost:3000/dashboard');
  
  // Wait for all widgets to load
  await page.waitForSelector('[data-testid="dashboard-loaded"]');
  
  // Take screenshot and compare with baseline
  await expect(page).toHaveScreenshot('dashboard-desktop.png', {
    fullPage: true,
    maxDiffPixels: 100  // Allow small differences
  });
});
```

#### 10 Critical E2E Tests:

1. ✅ Login → Dashboard → Logout
2. ✅ Create incident → View in list → View details
3. Add comment to incident
4. Create property → Assign residents
5. Visitor check-in with QR code
6. Change incident severity → Verify notification
7. Search incidents by keyword
8. Filter incidents by date range
9. Export incidents to CSV
10. Profile settings → Change password

#### Implementation Roadmap:

**Week 8:** Setup Playwright + Docker Compose E2E environment  
**Week 9:** Implement 10 tests + CI/CD integration

---

### Priority 5: UNIT TESTING (LOW PRIORITY) ⚪

**Timeline:** Weeks 10-13 (if time permits)  
**Effort:** 4-6 weeks (if doing everything)  
**ROI:** LOW in microservices - Focus ONLY on complex business logic  
**Coverage Target:** ~30-40% (NOT 100%)

#### WHY LAST?

In **microservices architecture**, unit tests have LESS value than in monoliths because:
- Most code is **orchestration** (calling other services) → Test with integration tests
- Controllers are **thin** (just HTTP handling) → Test with integration tests
- Repositories are **CRUD** (just database operations) → Test with integration tests

**ONLY write unit tests for:**
- ✅ **Complex business logic** (calculations, validations, algorithms)
- ✅ **Pure functions** (no dependencies, deterministic)
- ✅ **Utilities** (date formatting, string manipulation, parsers)

**DO NOT write unit tests for:**
- ❌ Controllers/Routers (use integration tests)
- ❌ Repositories (use integration tests with real DB)
- ❌ Services that only orchestrate calls (use contract tests)
- ❌ Getters/Setters (vanity metrics)

#### What to Test:

**Test 1: Complex Business Logic (Incident Severity Calculation)**

```python
# app/domain/incident_analyzer.py
from enum import Enum

class Severity(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

class IncidentAnalyzer:
    @staticmethod
    def calculate_severity(
        fire_detected: bool,
        smoke_level: int,  # 0-100
        temperature: float,  # Celsius
        time_of_day: int  # 0-23 hours
    ) -> Severity:
        """
        Complex business logic to calculate incident severity.
        THIS deserves a unit test.
        """
        if fire_detected:
            return Severity.CRITICAL
        
        if smoke_level > 80 or temperature > 100:
            return Severity.CRITICAL
        
        if smoke_level > 50 or temperature > 60:
            # Higher severity at night (when people sleep)
            if 22 <= time_of_day or time_of_day <= 6:
                return Severity.HIGH
            return Severity.MEDIUM
        
        return Severity.LOW

# tests/unit/test_incident_analyzer.py
import pytest
from app.domain.incident_analyzer import IncidentAnalyzer, Severity

class TestIncidentSeverityCalculation:
    def test_fire_detected_always_critical(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=True,
            smoke_level=10,
            temperature=20,
            time_of_day=12
        )
        assert severity == Severity.CRITICAL
    
    def test_high_smoke_level_critical(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=85,
            temperature=25,
            time_of_day=14
        )
        assert severity == Severity.CRITICAL
    
    def test_high_temperature_critical(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=30,
            temperature=105,
            time_of_day=10
        )
        assert severity == Severity.CRITICAL
    
    def test_medium_smoke_daytime_is_medium(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=60,
            temperature=40,
            time_of_day=14  # Afternoon
        )
        assert severity == Severity.MEDIUM
    
    def test_medium_smoke_nighttime_is_high(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=60,
            temperature=40,
            time_of_day=23  # Night
        )
        assert severity == Severity.HIGH
    
    def test_low_smoke_is_low_severity(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=20,
            temperature=25,
            time_of_day=10
        )
        assert severity == Severity.LOW
```

**Test 2: Utility Functions (Date Formatting)**

```python
# app/utils/date_helpers.py
from datetime import datetime, timezone

def format_relative_time(dt: datetime) -> str:
    """Convert datetime to human-readable relative time"""
    now = datetime.now(timezone.utc)
    diff = now - dt
    
    if diff.seconds < 60:
        return "just now"
    elif diff.seconds < 3600:
        minutes = diff.seconds // 60
        return f"{minutes} minute{'s' if minutes > 1 else ''} ago"
    elif diff.days == 0:
        hours = diff.seconds // 3600
        return f"{hours} hour{'s' if hours > 1 else ''} ago"
    elif diff.days == 1:
        return "yesterday"
    elif diff.days < 30:
        return f"{diff.days} days ago"
    else:
        return dt.strftime("%B %d, %Y")

# tests/unit/test_date_helpers.py
import pytest
from datetime import datetime, timedelta, timezone
from app.utils.date_helpers import format_relative_time

def test_just_now():
    dt = datetime.now(timezone.utc) - timedelta(seconds=30)
    assert format_relative_time(dt) == "just now"

def test_minutes_ago():
    dt = datetime.now(timezone.utc) - timedelta(minutes=5)
    assert format_relative_time(dt) == "5 minutes ago"

def test_hours_ago():
    dt = datetime.now(timezone.utc) - timedelta(hours=3)
    assert format_relative_time(dt) == "3 hours ago"

def test_yesterday():
    dt = datetime.now(timezone.utc) - timedelta(days=1)
    assert format_relative_time(dt) == "yesterday"

def test_days_ago():
    dt = datetime.now(timezone.utc) - timedelta(days=5)
    assert format_relative_time(dt) == "5 days ago"
```

#### Implementation Roadmap:

**Week 10-13:** Implement unit tests ONLY for complex logic (if time permits)

---

### Priority 6: PERFORMANCE TESTING (OPTIONAL) 🔵

**Timeline:** Week 14 (only if production deployment is imminent)  
**Effort:** 1 week  
**ROI:** OPTIONAL - Only if expecting high load  
**Coverage Target:** Top 20% endpoints (80% traffic)

#### WHY OPTIONAL?

First, make the system **WORK CORRECTLY**. Then, make it **FAST**.

But if you're deploying to production with **IoT devices, drones, and sensors**, you NEED to know if your system can handle the load.

#### Tools:
- **Locust** (Python - easy to configure)
- **k6** (Grafana - more powerful)
- **Apache JMeter** (enterprise standard)

#### What to Test:

**Test 1: Concurrent Incident Creation**

```python
# locust_tests/incident_load.py
from locust import HttpUser, task, between
import random

class IncidentUser(HttpUser):
    wait_time = between(1, 3)  # Wait 1-3 seconds between requests
    
    def on_start(self):
        """Login once per user"""
        response = self.client.post("/api/v1/auth/login", json={
            "email": f"loadtest{random.randint(1, 100)}@test.com",
            "password": "password123"
        })
        self.token = response.json()["access_token"]
    
    @task(3)  # 3x more likely than other tasks
    def create_incident(self):
        """Simulate creating incidents"""
        self.client.post(
            "/api/v1/incidents",
            json={
                "title": f"Load Test Incident {random.randint(1, 10000)}",
                "severity": random.choice(["LOW", "MEDIUM", "HIGH", "CRITICAL"]),
                "property_id": random.randint(1, 10)
            },
            headers={"Authorization": f"Bearer {self.token}"}
        )
    
    @task(1)
    def list_incidents(self):
        """Simulate listing incidents"""
        self.client.get(
            "/api/v1/incidents",
            headers={"Authorization": f"Bearer {self.token}"}
        )
    
    @task(1)
    def view_dashboard(self):
        """Simulate viewing dashboard"""
        self.client.get(
            "/api/v1/analytics/dashboard",
            headers={"Authorization": f"Bearer {self.token}"}
        )

# Run with: locust -f incident_load.py --host=http://localhost:8000 --users 1000 --spawn-rate 50
```

**Test 2: IoT Sensor Event Load**

```python
class IoTSensorUser(HttpUser):
    wait_time = between(0.1, 0.5)  # High frequency
    
    @task
    def send_sensor_event(self):
        """Simulate IoT sensors sending events"""
        self.client.post("/api/v1/iot/sensor-events", json={
            "sensor_id": f"sensor_{random.randint(1, 1000)}",
            "event_type": "SMOKE_DETECTED",
            "value": random.uniform(0, 100),
            "timestamp": datetime.now().isoformat()
        })
```

#### Performance Targets:

| Endpoint              | Target Response Time | Target Throughput |
|-----------------------|----------------------|-------------------|
| GET /incidents        | < 200ms (p95)        | 1000 req/s        |
| POST /incidents       | < 500ms (p95)        | 500 req/s         |
| POST /auth/login      | < 300ms (p95)        | 200 req/s         |
| GET /analytics        | < 1000ms (p95)       | 100 req/s         |
| POST /iot/events      | < 100ms (p95)        | 5000 req/s        |

---

## 📊 TESTING IMPLEMENTATION ROADMAP

### Timeline Overview (14 Weeks)

```
Week 1-3:  Contract Testing (CRITICAL)         🔴
Week 1-2:  Security Testing (CRITICAL)         🔴
Week 4-7:  Integration Testing (HIGH)          🟡
Week 8-9:  E2E Testing (MEDIUM)                🟢
Week 10-13: Unit Testing (LOW - optional)      ⚪
Week 14:   Performance Testing (OPTIONAL)      🔵
```

### Phased Approach

#### Phase 1: Foundation (Weeks 1-3) - BLOCKER

**Goal:** Prevent microservices from breaking each other + Fix security vulnerabilities  
**Team:** 2-3 developers  
**Deliverables:**
- [ ] Pact broker setup
- [ ] 20 critical contracts implemented
- [ ] All 9 security vulnerabilities fixed with tests
- [ ] CI/CD pipeline with contract + security tests

**Success Criteria:**
- ✅ Zero breaking changes between services
- ✅ All Bandit/OWASP ZAP scans pass
- ✅ Rate limiting active on all login endpoints

---

#### Phase 2: Critical Flows (Weeks 4-7) - HIGH PRIORITY

**Goal:** Validate core business functionality works end-to-end  
**Team:** 2-3 developers  
**Deliverables:**
- [ ] TestContainers setup
- [ ] Top 20 integration tests implemented
- [ ] Multi-tenancy isolation verified
- [ ] GDPR compliance flows tested

**Success Criteria:**
- ✅ All critical flows green (auth, incidents, notifications)
- ✅ No data leaks between tenants
- ✅ 60% integration test coverage

---

#### Phase 3: User Experience (Weeks 8-9) - MEDIUM PRIORITY

**Goal:** Ensure frontend works correctly with backend  
**Team:** 1-2 frontend developers  
**Deliverables:**
- [ ] Playwright setup
- [ ] 10 E2E tests for happy paths
- [ ] Visual regression tests
- [ ] CI/CD with E2E tests

**Success Criteria:**
- ✅ Login → Dashboard → Create Incident flow works
- ✅ No visual regressions
- ✅ E2E tests run on every PR

---

#### Phase 4: Edge Cases (Weeks 10-13) - LOW PRIORITY

**Goal:** Cover complex business logic with unit tests  
**Team:** 1 developer  
**Deliverables:**
- [ ] Unit tests for complex logic
- [ ] 30-40% code coverage (NOT 100%)
- [ ] Focus on domain layer

**Success Criteria:**
- ✅ All complex calculations tested
- ✅ Utilities covered

---

#### Phase 5: Production Readiness (Week 14) - OPTIONAL

**Goal:** Validate system can handle production load  
**Team:** 1 DevOps engineer  
**Deliverables:**
- [ ] Locust performance tests
- [ ] Load testing report
- [ ] Bottleneck identification

**Success Criteria:**
- ✅ System handles 1000 concurrent users
- ✅ p95 response times < targets

---

## 🎯 TESTING METRICS & KPIs

### Coverage Goals

| Test Type      | Target Coverage | Current | Gap  |
|----------------|-----------------|---------|------|
| Contract       | 100%            | 0%      | 100% |
| Security       | 100% vulns      | 0%      | 100% |
| Integration    | 80% flows       | 0%      | 80%  |
| E2E            | 10-15 tests     | 0       | 15   |
| Unit           | 30-40%          | 0%      | 40%  |
| **OVERALL**    | **60-70%**      | **20%** | **50%** |

### Quality Gates (CI/CD)

```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  contract-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Run Pact Tests
        run: pytest tests/contract/
      - name: Publish Pacts
        run: pact-broker publish
    # BLOCKER: Must pass to merge

  security-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Run Bandit
        run: bandit -r services/
      - name: Run OWASP ZAP
        run: docker run owasp/zap scan
    # BLOCKER: Must pass to merge

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Start TestContainers
        run: docker-compose -f docker-compose.test.yml up -d
      - name: Run Integration Tests
        run: pytest tests/integration/
    # BLOCKER: Must pass to merge

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Start Full System
        run: docker-compose up -d
      - name: Run Playwright
        run: npx playwright test
    # WARNING: Can be skipped if urgent

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Run Unit Tests
        run: pytest tests/unit/
    # INFO: Coverage report only
```

---

## 🚀 NEXT STEPS

### Immediate (This Week)

1. [ ] **Read** this strategy with tech team
2. [ ] **Approve** priority order and timeline
3. [ ] **Assign** 2-3 developers to testing team
4. [ ] **Setup** Pact broker and TestContainers infrastructure

### Week 1-2 (CRITICAL)

1. [ ] Implement contract tests for top 10 service pairs
2. [ ] Fix all 9 security vulnerabilities
3. [ ] Write security tests for each vulnerability
4. [ ] Setup CI/CD pipeline with contract + security tests

### Month 1 (HIGH)

1. [ ] Complete all contract tests (100% inter-service)
2. [ ] Implement top 20 integration tests
3. [ ] Verify multi-tenancy isolation
4. [ ] Security audit after fixes

### Month 2-3 (MEDIUM)

1. [ ] E2E tests for happy paths
2. [ ] Unit tests for complex logic (optional)
3. [ ] Performance testing (optional)
4. [ ] **GO-LIVE** to production

---

## 📞 RECOMMENDATIONS

### DO:
- ✅ Start with contract tests (prevent breaking changes)
- ✅ Fix security vulnerabilities with tests (production blocker)
- ✅ Focus on integration tests (high ROI)
- ✅ Write E2E tests for happy paths only (don't overdo it)
- ✅ Use TestContainers for realistic integration tests

### DON'T:
- ❌ Chase 100% code coverage (vanity metric)
- ❌ Write unit tests for controllers (use integration tests)
- ❌ Write unit tests for repositories (use integration tests)
- ❌ Write E2E tests for edge cases (too slow)
- ❌ Delay production for unit test coverage

---

## 📚 REFERENCES

### Documentation
- [Security Audit](./SECURITY_AUDIT_EN.md) - 9 critical vulnerabilities to fix
- [Code Quality Audit](./CODE_QUALITY_AUDIT_EN.md) - Technical debt analysis
- [Functionality Audit](./FUNCTIONALITY_AUDIT_EN.md) - 394 endpoints to test

### Tools
- **Pact:** https://docs.pact.io/
- **pytest:** https://docs.pytest.org/
- **Playwright:** https://playwright.dev/
- **Locust:** https://docs.locust.io/
- **Bandit:** https://bandit.readthedocs.io/
- **OWASP ZAP:** https://www.zaproxy.org/docs/

---

**Audit date:** December 11, 2025  
**Next review:** After Phase 1 completion (Week 3)
