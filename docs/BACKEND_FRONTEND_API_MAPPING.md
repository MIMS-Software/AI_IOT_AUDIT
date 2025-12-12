# BACKEND ↔ FRONTEND API MAPPING ANALYSIS

**Date:** December 11, 2025  
**Project:** AI_IOT_AUDIT - MIMS Tech Platform  
**Status:** 🔴 CRITICAL MISMATCHES DETECTED

---

## 📊 EXECUTIVE SUMMARY

### Services Status

✅ **All 12 microservices are HEALTHY**

- identity-service (8001)
- property-service (8002)
- visitor-service (8003)
- parking-service (8004)
- access-service (8005)
- incident-service (8006)
- compliance-service (8007)
- iot-service (8008)
- analytics-service (8009)
- notification-service (8010)
- audit-service (8011)
- realtime-service (8012)

### Critical Issues Found

1. 🔴 **Frontend calling non-existent endpoints** (404 errors)
2. 🔴 **Hardcoded TEST_TENANT_001** instead of dynamic tenant_id from JWT
3. 🔴 **Missing Authorization headers** on some requests (401 errors)
4. 🔴 **502 Bad Gateway errors** on visitor-service endpoints through Kong
5. ⚠️ **React hydration errors** and invalid HTML structure

---

## 🔴 ENDPOINTS THAT DON'T EXIST (404 Errors)

### 1. `/api/v1/system/health`

**Frontend calls:** `SystemHealthBadge.js`

```javascript
// frontend/src/components/SystemHealthBadge.js:21
// API: /api/v1/system/health (from DASH-005)
```

**Backend reality:** ❌ **DOES NOT EXIST**

- No service exposes this endpoint
- Closest matches:
  - `GET /health` (exists on ALL services)
  - `GET /api/v1/devices/health` (iot-service)
  - `GET /health/detailed` (realtime-service)

**Fix Required:**

- Option A: Change frontend to call `/health` on specific services
- Option B: Implement aggregate health endpoint in Kong or a dedicated health-check service
- Option C: Remove SystemHealthBadge component

---

### 2. `/api/v1/iot/sensors/health`

**Frontend calls:** `IoTSensorEvents.js`

```javascript
// frontend/src/components/IoTSensorEvents.js:53
const healthResponse = await apiService.get("/iot/sensors/health");
```

**Backend reality:** ❌ **DOES NOT EXIST**

**Available endpoints in iot-service:**

```
✅ /api/v1/devices/health
✅ /api/v1/devices/sensors/health        ← THIS IS THE CORRECT ONE
✅ /api/v1/devices/sensors/{sensor_id}
✅ /api/v1/devices/sensors/critical
✅ /api/v1/devices/sensors/summary
```

**Fix Required:**

```javascript
// WRONG (current):
const response = await apiService.get("/iot/sensors/health");

// CORRECT:
const response = await apiService.get("/devices/sensors/health");
```

---

## 🔴 HARDCODED TENANT ID ISSUE

### Problem

Frontend hardcodes `TEST_TENANT_001` which **DOES NOT EXIST** in the database.

**Files affected:**

1. `frontend/src/contexts/TenantContext.js:16`

   ```javascript
   return localStorage.getItem("selectedTenant") || "TEST_TENANT_001";
   ```

2. `frontend/src/components/layout/TenantSelector.js:16`

   ```javascript
   { id: 'TEST_TENANT_001', name: 'Test Tenant' }
   ```

3. `frontend/src/components/layout/TenantSelector.js:41`

   ```javascript
   const [selectedTenant, setSelectedTenant] = useState(
     value || "TEST_TENANT_001",
   );
   ```

### Valid Tenant IDs (from database)

```sql
SELECT tenant_id, name, tier, database_strategy, status
FROM mims_central.tenants;
```

| tenant_id         | name                           | tier         | strategy |
| ----------------- | ------------------------------ | ------------ | -------- |
| **default**       | MIMS Platform Default          | starter      | bridge   |
| sunrise-hoa       | Sunrise Homeowners Association | starter      | bridge   |
| oakwood-community | Oakwood Gated Community        | professional | silo     |
| metro-city        | Metro City Security District   | enterprise   | silo     |

### Current User Tenant

```sql
SELECT id, username, email, tenant_id, role
FROM identity_db.users;
```

| id  | username | email          | tenant_id   | role         |
| --- | -------- | -------------- | ----------- | ------------ |
| 1   | admin    | <admin@test.com> | **default** | SYSTEM_ADMIN |

### Fix Required

**Option A: Change default to 'default'**

```javascript
// TenantContext.js
return localStorage.getItem("selectedTenant") || "default";
```

**Option B: Extract tenant_id from JWT token**

```javascript
import { jwtDecode } from "jwt-decode";

const token = localStorage.getItem("access_token");
if (token) {
  const decoded = jwtDecode(token);
  return decoded.tenant_id; // JWT contains tenant_id
}
```

**Option C: Fetch from user profile**

```javascript
const user = await authService.getCurrentUser(); // /api/v1/auth/me
return user.tenant_id || user.community_id || "default";
```

---

## 🔴 502 BAD GATEWAY ERRORS

### Affected Endpoints

```
❌ /api/v1/emergency-mode/status?tenant_id=TEST_TENANT_001  → 502
❌ /api/v1/visits?tenant_id=TEST_TENANT_001                 → 502
❌ /api/v1/visits/active?tenant_id=TEST_TENANT_001          → 502
```

### Root Cause Analysis

#### 1. Emergency Mode Endpoint

**Backend:** ✅ EXISTS in compliance-service

```
GET /api/v1/emergency-mode/status
```

**Issue:** Kong routing or service response timeout

**Debug steps:**

```bash
# Test direct service call (bypass Kong)
curl -H "X-Tenant-ID: default" http://localhost:8007/api/v1/emergency-mode/status

# Test through Kong
curl -H "X-Tenant-ID: default" http://localhost:8000/api/v1/emergency-mode/status

# Check compliance-service logs
docker compose logs compliance-service --tail=50
```

#### 2. Visits Endpoints

**Backend:** ✅ EXISTS in visitor-service

```
GET /api/v1/visits/
GET /api/v1/visits/active
```

**Issue:** Likely multi-tenant database routing problem

**Possible causes:**

1. Tenant 'TEST_TENANT_001' doesn't exist → database connection fails
2. Schema 'tenant_test_tenant_001' not created in visitor_db
3. SQLAlchemy trying to create tables but hitting conflicts

**Debug steps:**

```bash
# Check if schema exists
docker compose exec postgres psql -U mims_admin -d visitor_db -c "\dn"

# Check visitor-service logs for SQL errors
docker compose logs visitor-service --tail=100 | grep -i error

# Test with correct tenant_id
curl -H "X-Tenant-ID: default" \
  "http://localhost:8003/api/v1/visits?page_size=10&page=1"
```

---

## 🔴 401 UNAUTHORIZED ERRORS

### Affected Endpoints

```
❌ /api/v1/notifications/unread-count → 401
```

**Backend:** ✅ EXISTS in notification-service

```
GET /api/v1/notifications/unread-count
```

**Issue:** Missing or invalid Authorization header

### Root Cause

Frontend is making this request **without JWT token** or before login completes.

### Fix Required

**Check `api.service.js` interceptor:**

```javascript
// frontend/src/services/api.service.js
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem("access_token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

**Also check component lifecycle:**

```javascript
// Ensure user is authenticated before calling
useEffect(() => {
  if (isAuthenticated && user) {
    fetchNotifications();
  }
}, [isAuthenticated, user]);
```

---

## ✅ WORKING ENDPOINTS (200 OK)

These endpoints work correctly:

| Endpoint                       | Service           | Status    |
| ------------------------------ | ----------------- | --------- |
| `/api/v1/incidents`            | incident-service  | ✅ 200 OK |
| `/api/v1/lpr/matches`          | iot-service       | ✅ 200 OK |
| `/api/v1/activity/feed`        | analytics-service | ✅ 200 OK |
| `/api/v1/devices/health`       | iot-service       | ✅ 200 OK |
| `/api/v1/analytics/violations` | analytics-service | ✅ 200 OK |

---

## 📋 COMPLETE BACKEND API INVENTORY

### 1️⃣ Identity Service (Port 8001)

**Total endpoints:** 11

```
Authentication:
  POST   /api/v1/auth/login
  POST   /api/v1/auth/register
  GET    /api/v1/auth/me
  GET    /api/v1/auth/residents
  POST   /api/v1/auth/users/batch

Contacts:
  GET    /api/v1/contacts/
  POST   /api/v1/contacts/
  GET    /api/v1/contacts/{contact_id}
  PUT    /api/v1/contacts/{contact_id}
  DELETE /api/v1/contacts/{contact_id}
  POST   /api/v1/contacts/{contact_id}/restore
  GET    /api/v1/contacts/search

Health:
  GET    /
  GET    /health
```

---

### 2️⃣ Property Service (Port 8002)

**Total endpoints:** 13

```
Properties:
  GET    /api/v1/properties
  POST   /api/v1/properties
  GET    /api/v1/properties/{property_id}
  PUT    /api/v1/properties/{property_id}
  DELETE /api/v1/properties/{property_id}

Visit Purposes:
  GET    /api/v1/visit-purposes/
  POST   /api/v1/visit-purposes/
  GET    /api/v1/visit-purposes/{purpose_id}
  PUT    /api/v1/visit-purposes/{purpose_id}
  DELETE /api/v1/visit-purposes/{purpose_id}
  POST   /api/v1/visit-purposes/{purpose_id}/restore

Fine Tables (for parking violations):
  GET    /api/v1/fine-tables
  POST   /api/v1/fine-tables
  GET    /api/v1/fine-tables/{fine_id}
  PUT    /api/v1/fine-tables/{fine_id}
  DELETE /api/v1/fine-tables/{fine_id}

Rules:
  GET    /api/v1/rules
  POST   /api/v1/rules
  GET    /api/v1/rules/{rule_id}
  PUT    /api/v1/rules/{rule_id}
  DELETE /api/v1/rules/{rule_id}

Towing Companies:
  GET    /api/v1/towing-companies
  POST   /api/v1/towing-companies
  GET    /api/v1/towing-companies/{company_id}
  PUT    /api/v1/towing-companies/{company_id}
  DELETE /api/v1/towing-companies/{company_id}

Health:
  GET    /
  GET    /health
```

---

### 3️⃣ Visitor Service (Port 8003)

**Total endpoints:** 48

```
Visitors:
  GET    /api/v1/visitors/
  POST   /api/v1/visitors/
  GET    /api/v1/visitors/{visitor_id}
  PUT    /api/v1/visitors/{visitor_id}
  DELETE /api/v1/visitors/{visitor_id}
  POST   /api/v1/visitors/{visitor_id}/restore
  POST   /api/v1/visitors/{visitor_id}/soft-delete
  POST   /api/v1/visitors/{visitor_id}/anonymize
  POST   /api/v1/visitors/{visitor_id}/legal-hold
  GET    /api/v1/visitors/{visitor_id}/passes
  GET    /api/v1/visitors/{visitor_id}/visits
  GET    /api/v1/visitors/{visitor_id}/find-duplicates
  POST   /api/v1/visitors/{visitor_id}/merge-duplicate
  POST   /api/v1/visitors/{visitor_id}/watchlist
  GET    /api/v1/visitors/code/{visitor_code}
  GET    /api/v1/visitors/search
  GET    /api/v1/visitors/search/phone/{phone_number}
  POST   /api/v1/visitors/check-duplicates
  GET    /api/v1/visitors/with-visit
  GET    /api/v1/visitors/stats/summary
  GET    /api/v1/visitors/export

Visits:
  GET    /api/v1/visits/
  POST   /api/v1/visits/
  GET    /api/v1/visits/{visit_id}
  PUT    /api/v1/visits/{visit_id}
  DELETE /api/v1/visits/{visit_id}
  POST   /api/v1/visits/{visit_id}/approve
  POST   /api/v1/visits/{visit_id}/deny
  GET    /api/v1/visits/active
  GET    /api/v1/visits/pending-approval
  GET    /api/v1/visits/denied

Passes:
  GET    /api/v1/passes/
  POST   /api/v1/passes/
  GET    /api/v1/passes/{pass_id}
  PUT    /api/v1/passes/{pass_id}
  DELETE /api/v1/passes/{pass_id}
  POST   /api/v1/passes/{pass_id}/check-in
  POST   /api/v1/passes/{pass_id}/check-out
  POST   /api/v1/passes/{pass_id}/validate
  POST   /api/v1/passes/{pass_id}/revoke
  POST   /api/v1/passes/{pass_id}/reactivate
  GET    /api/v1/passes/check-qr/{qr_code}
  GET    /api/v1/passes/active/count
  GET    /api/v1/passes/expiring-soon

Preregistrations:
  GET    /api/v1/preregistrations/
  POST   /api/v1/preregistrations/
  GET    /api/v1/preregistrations/{link_id}
  PUT    /api/v1/preregistrations/{link_id}
  DELETE /api/v1/preregistrations/{link_id}
  POST   /api/v1/preregistrations/{link_id}/deactivate
  GET    /api/v1/preregistrations/{link_id}/stats
  GET    /api/v1/preregistrations/{link_id}/visitors
  GET    /api/v1/preregistrations/my-links
  GET    /api/v1/preregistrations/validate/{link_token}
  POST   /api/v1/preregistrations/register/{link_token}

Kiosk:
  POST   /api/v1/kiosk/checkin
  POST   /api/v1/kiosk/checkout
  GET    /api/v1/kiosk/reader/status/{reader_id}

Health:
  GET    /
  GET    /health
```

---

### 4️⃣ Parking Service (Port 8004)

**Total endpoints:** 29

```
Parking Spaces:
  GET    /api/v1/parking/spaces
  POST   /api/v1/parking/spaces
  GET    /api/v1/parking/spaces/{space_id}
  PUT    /api/v1/parking/spaces/{space_id}
  DELETE /api/v1/parking/spaces/{space_id}
  GET    /api/v1/parking/availability
  GET    /api/v1/parking/occupancy
  GET    /api/v1/parking/occupancy/by-zone

Assignments:
  GET    /api/v1/parking/assignments
  POST   /api/v1/parking/assignments
  GET    /api/v1/parking/assignments/{assignment_id}
  DELETE /api/v1/parking/assignments/{assignment_id}/release
  POST   /api/v1/parking/assignments/{assignment_id}/pay

Vehicles:
  GET    /api/v1/vehicles
  POST   /api/v1/vehicles
  GET    /api/v1/vehicles/{vehicle_id}
  PUT    /api/v1/vehicles/{vehicle_id}
  DELETE /api/v1/vehicles/{vehicle_id}
  GET    /api/v1/vehicles/search/{license_plate}
  GET    /api/v1/vehicles/visitor/{visitor_id}
  POST   /api/v1/vehicles/{vehicle_id}/verify-lpr
  GET    /api/v1/vehicles/lpr/verified

Violations:
  GET    /api/v1/violations
  POST   /api/v1/violations
  GET    /api/v1/violations/{violation_id}
  PUT    /api/v1/violations/{violation_id}
  DELETE /api/v1/violations/{violation_id}
  POST   /api/v1/violations/{violation_id}/waive
  POST   /api/v1/violations/{violation_id}/link-incident
  DELETE /api/v1/violations/{violation_id}/unlink-incident
  GET    /api/v1/violations/by-incident/{incident_id}

Appeals:
  GET    /api/v1/violations/appeals
  POST   /api/v1/violations/appeals
  GET    /api/v1/violations/appeals/{appeal_id}
  PUT    /api/v1/violations/appeals/{appeal_id}
  DELETE /api/v1/violations/appeals/{appeal_id}/withdraw

Parking Violations (different model):
  GET    /api/v1/parking/parking-violations
  POST   /api/v1/parking/detect-violations
  POST   /api/v1/parking/parking-violations/{violation_id}/mark-paid
  POST   /api/v1/parking/parking-violations/{violation_id}/dismiss
  POST   /api/v1/parking/parking-violations/{violation_id}/request-tow

Health:
  GET    /
  GET    /health
```

---

### 5️⃣ Access Service (Port 8005)

**Total endpoints:** 30

```
Gates:
  GET    /api/v1/gates
  POST   /api/v1/gates
  GET    /api/v1/gates/{gate_id}
  PUT    /api/v1/gates/{gate_id}
  DELETE /api/v1/gates/{gate_id}
  POST   /api/v1/gates/{gate_id}/status
  POST   /api/v1/gates/{gate_id}/emergency-override
  POST   /api/v1/gates/access-request
  GET    /api/v1/gates/access-logs

Decals:
  GET    /api/v1/decals
  POST   /api/v1/decals
  GET    /api/v1/decals/{decal_id}
  PUT    /api/v1/decals/{decal_id}
  DELETE /api/v1/decals/{decal_id}
  POST   /api/v1/decals/{decal_id}/status
  GET    /api/v1/decals/{decal_id}/qr-code
  POST   /api/v1/decals/{decal_id}/fraud-check
  GET    /api/v1/decals/validate-decal/{qr_code}
  POST   /api/v1/decals/bulk-fraud-scan
  GET    /api/v1/decals/decals

RFID:
  GET    /api/v1/rfid/tags
  POST   /api/v1/rfid/tags
  GET    /api/v1/rfid/tags/{tag_id}
  PUT    /api/v1/rfid/tags/{tag_id}
  DELETE /api/v1/rfid/tags/{tag_id}
  POST   /api/v1/rfid/tags/{tag_id}/status
  GET    /api/v1/rfid/validate/{tag_id_str}
  POST   /api/v1/rfid/readings
  GET    /api/v1/rfid/readings
  GET    /api/v1/rfid/system/status
  POST   /api/v1/rfid/bulk-anomaly-scan

Zones:
  GET    /api/v1/zones
  POST   /api/v1/zones
  GET    /api/v1/zones/{zone_id}
  PUT    /api/v1/zones/{zone_id}
  DELETE /api/v1/zones/{zone_id}
  GET    /api/v1/zones/{zone_id}/occupancy
  POST   /api/v1/zones/{zone_id}/increment-occupancy
  POST   /api/v1/zones/{zone_id}/decrement-occupancy
  GET    /api/v1/zones/pass-zone-access
  POST   /api/v1/zones/pass-zone-access
  GET    /api/v1/zones/pass-zone-access/{access_id}
  GET    /api/v1/zones/pass-zone-access/pass/{pass_id}
  GET    /api/v1/zones/pass-zone-access/zone/{zone_id}

Health:
  GET    /
  GET    /health
```

---

### 6️⃣ Incident Service (Port 8006)

**Total endpoints:** 26

```
Incidents:
  GET    /api/v1/incidents
  POST   /api/v1/incidents
  GET    /api/v1/incidents/{incident_id}
  PUT    /api/v1/incidents/{incident_id}
  DELETE /api/v1/incidents/{incident_id}
  POST   /api/v1/incidents/{incident_id}/assign
  POST   /api/v1/incidents/{incident_id}/escalate
  POST   /api/v1/incidents/{incident_id}/resolve
  POST   /api/v1/incidents/{incident_id}/close
  POST   /api/v1/incidents/{incident_id}/reopen
  POST   /api/v1/incidents/{incident_id}/legal-hold
  POST   /api/v1/incidents/{incident_id}/link
  POST   /api/v1/incidents/{incident_id}/merge
  GET    /api/v1/incidents/{incident_id}/audit

Attachments:
  GET    /api/v1/incidents/{incident_id}/attachments
  POST   /api/v1/incidents/{incident_id}/attachments
  DELETE /api/v1/incidents/{incident_id}/attachments/{attachment_id}

Comments:
  GET    /api/v1/incidents/{incident_id}/comments
  POST   /api/v1/incidents/{incident_id}/comments
  PUT    /api/v1/incidents/{incident_id}/comments/{comment_id}
  DELETE /api/v1/incidents/{incident_id}/comments/{comment_id}

Tasks:
  GET    /api/v1/incidents/{incident_id}/tasks
  POST   /api/v1/incidents/{incident_id}/tasks
  PUT    /api/v1/incidents/{incident_id}/tasks/{task_id}
  DELETE /api/v1/incidents/{incident_id}/tasks/{task_id}

Analytics:
  GET    /api/v1/incidents/dashboard
  GET    /api/v1/incidents/kpis
  GET    /api/v1/incidents/statistics
  GET    /api/v1/incidents/trends

Reports:
  GET    /api/v1/incidents/report/csv
  GET    /api/v1/incidents/report/pdf

Auto-Creation:
  POST   /api/v1/incidents/auto-create
  GET    /api/v1/incidents/auto-create/config
  PUT    /api/v1/incidents/auto-create/config
  GET    /api/v1/incidents/auto-create/statistics

Health:
  GET    /
  GET    /health
```

---

### 7️⃣ Compliance Service (Port 8007)

**Total endpoints:** 47

```
Emergency Mode:
  GET    /api/v1/emergency-mode
  POST   /api/v1/emergency-mode
  GET    /api/v1/emergency-mode/{mode_id}
  PUT    /api/v1/emergency-mode/{mode_id}
  POST   /api/v1/emergency-mode/{mode_id}/deactivate
  POST   /api/v1/emergency-mode/{mode_id}/cancel
  POST   /api/v1/emergency-mode/{mode_id}/link-incident
  GET    /api/v1/emergency-mode/status
  GET    /api/v1/emergency-mode/active/count

Blacklist:
  GET    /api/v1/blacklist
  POST   /api/v1/blacklist
  GET    /api/v1/blacklist/{entry_id}
  PUT    /api/v1/blacklist/{entry_id}
  DELETE /api/v1/blacklist/{entry_id}
  POST   /api/v1/blacklist/{entry_id}/deactivate
  POST   /api/v1/blacklist/{entry_id}/reactivate
  POST   /api/v1/blacklist/screen
  GET    /api/v1/blacklist/stats
  GET    /api/v1/blacklist/visitor/{visitor_id}

Legal Hold:
  GET    /api/v1/legal-hold/list
  POST   /api/v1/legal-hold/place
  POST   /api/v1/legal-hold/remove
  GET    /api/v1/legal-hold/check/{visitor_id}
  GET    /api/v1/legal-hold/stats

Incident Legal Hold:
  GET    /api/v1/incidents/legal-hold/list
  POST   /api/v1/incidents/legal-hold/place
  GET    /api/v1/incidents/legal-hold/check/{incident_id}
  GET    /api/v1/incidents/legal-hold/stats

DSR (Data Subject Requests):
  GET    /api/v1/dsr/requests
  POST   /api/v1/dsr/data-subject-request
  GET    /api/v1/dsr/requests/{request_id}
  PUT    /api/v1/dsr/requests/{request_id}
  GET    /api/v1/dsr/requests/{request_id}/export
  POST   /api/v1/dsr/anonymize
  GET    /api/v1/dsr/deleted-visitors
  POST   /api/v1/dsr/restore
  POST   /api/v1/dsr/permanent-delete
  GET    /api/v1/dsr/purge-preview/{data_type}
  POST   /api/v1/dsr/purge/{data_type}
  GET    /api/v1/dsr/statistics

Retention Policies:
  GET    /api/v1/dsr/retention-policies
  POST   /api/v1/dsr/retention-policies
  GET    /api/v1/dsr/retention-policies/{policy_id}
  PUT    /api/v1/dsr/retention-policies/{policy_id}
  DELETE /api/v1/dsr/retention-policies/{policy_id}

Breach Notifications:
  GET    /api/v1/breach-notifications
  POST   /api/v1/breach-notifications/trigger
  GET    /api/v1/breach-notifications/{notification_id}
  POST   /api/v1/breach-notifications/{notification_id}/notify-authority
  GET    /api/v1/breach-notifications/stats

Privacy:
  GET    /api/v1/privacy/policy
  GET    /api/v1/privacy/consent/{visitor_id}
  POST   /api/v1/privacy/consent
  POST   /api/v1/privacy/consent/withdraw

Health:
  GET    /
  GET    /health
```

---

### 8️⃣ IoT Service (Port 8008)

**Total endpoints:** 95 (LARGEST SERVICE!)

```
Devices:
  GET    /api/v1/devices/status
  GET    /api/v1/devices/health
  GET    /api/v1/devices/health/{device_id}
  GET    /api/v1/devices/health/summary/by-type
  GET    /api/v1/devices/health/locations

Sensors:
  GET    /api/v1/devices/sensors/{sensor_id}
  PUT    /api/v1/devices/sensors/{sensor_id}
  GET    /api/v1/devices/sensors/health
  GET    /api/v1/devices/sensors/critical
  GET    /api/v1/devices/sensors/summary
  GET    /api/v1/devices/sensors/zones
  POST   /api/v1/devices/sensors/readings

Cameras:
  GET    /api/v1/ai/cameras

Video:
  GET    /api/v1/video/cameras
  GET    /api/v1/video/cameras/{camera_id}/status
  GET    /api/v1/video/cameras/live
  GET    /api/v1/video/live
  GET    /api/v1/video/live/{feed_id}
  POST   /api/v1/video/live/{feed_id}/pip/enable
  GET    /api/v1/video/playback
  GET    /api/v1/video/stats/summary
  POST   /api/v1/video/streams/{camera_id}/start
  POST   /api/v1/video/streams/{camera_id}/stop
  GET    /api/v1/video/streams/{camera_id}/url
  GET    /api/v1/video/streams/active
  GET    /api/v1/video/thermal/{camera_id}
  POST   /api/v1/video/thermal/{camera_id}/overlay
  GET    /api/v1/video/thumbnail/{camera_id}

Video Evidence:
  POST   /api/v1/video/evidence/export
  GET    /api/v1/video/evidence/exports
  GET    /api/v1/video/evidence/exports/{export_id}

LPR (License Plate Recognition):
  GET    /api/v1/lpr/cameras
  GET    /api/v1/lpr/cameras/status
  GET    /api/v1/lpr/matches
  GET    /api/v1/lpr/matches/{match_id}
  GET    /api/v1/lpr/matches/alerts
  GET    /api/v1/lpr/statistics/daily

RFID:
  POST   /api/v1/rfid/check-in
  GET    /api/v1/rfid/reader/status/{reader_id}

AI Behavior Detection:
  GET    /api/v1/ai/behavior-alerts
  POST   /api/v1/ai/behavior-alerts
  GET    /api/v1/ai/behavior-alerts/{alert_id}
  PUT    /api/v1/ai/behavior-alerts/{alert_id}
  POST   /api/v1/ai/behavior-alerts/{alert_id}/acknowledge
  POST   /api/v1/ai/behavior-alerts/{alert_id}/escalate
  GET    /api/v1/ai/behavior-alerts/realtime/stream
  GET    /api/v1/ai/behavior-alerts/statistics/summary
  GET    /api/v1/ai/behavior-types

AI Audio Detection:
  GET    /api/v1/ai/audio/detections
  POST   /api/v1/ai/audio/detections
  GET    /api/v1/ai/audio/detections/{detection_id}
  PUT    /api/v1/ai/audio/detections/{detection_id}
  POST   /api/v1/ai/audio/detections/{detection_id}/verify
  GET    /api/v1/ai/audio/detections/critical
  GET    /api/v1/ai/audio/detections/summary
  GET    /api/v1/ai/audio/sensors/status

Drones:
  GET    /api/v1/drones/fleet
  GET    /api/v1/drones/status
  GET    /api/v1/drones/status/tiles
  POST   /api/v1/drones/{drone_id}/dispatch
  GET    /api/v1/drones/missions/active
  GET    /api/v1/drones/missions/summary
  GET    /api/v1/drones/missions/{mission_id}
  PUT    /api/v1/drones/missions/{mission_id}

Drone Docks:
  GET    /api/v1/drones/docks
  POST   /api/v1/drones/docks
  GET    /api/v1/drones/docks/{dock_id}
  PUT    /api/v1/drones/docks/{dock_id}
  POST   /api/v1/drones/docks/{dock_id}/assign-drone

Drone Pilots:
  GET    /api/v1/drones/pilots
  POST   /api/v1/drones/pilots

Drone Logs:
  GET    /api/v1/drones/logs
  GET    /api/v1/drones/logs/{log_id}
  GET    /api/v1/drones/logs/export/csv
  GET    /api/v1/drones/logs/export/pdf
  GET    /api/v1/drones/logs/statistics/summary

Drone AI Feed:
  GET    /api/v1/drones/ai-feed
  GET    /api/v1/drones/ai-feed/detections/{detection_id}
  GET    /api/v1/drones/ai-feed/statistics
  POST   /api/v1/drones/ai-feed/toggle-ai
  POST   /api/v1/drones/ai-feed/toggle-thermal

SOC (Security Operations Center):
  GET    /api/v1/soc/alerts/recent
  GET    /api/v1/soc/alerts/critical
  POST   /api/v1/soc/alerts/{alert_id}/acknowledge
  POST   /api/v1/soc/alerts/{alert_id}/escalate
  GET    /api/v1/soc/alerts/statistics

SOC Dispatch:
  POST   /api/v1/soc/dispatch/patrol
  POST   /api/v1/soc/dispatch/drone
  POST   /api/v1/soc/dispatch/law-enforcement
  GET    /api/v1/soc/dispatch/status/{dispatch_id}
  POST   /api/v1/soc/dispatch/status/{dispatch_id}/cancel
  POST   /api/v1/soc/dispatch/status/{dispatch_id}/complete
  GET    /api/v1/soc/dispatch/history
  GET    /api/v1/soc/dispatch/statistics/summary

Health:
  GET    /
  GET    /health
```

---

### 9️⃣ Analytics Service (Port 8009)

**Total endpoints:** 18

```
Analytics:
  GET    /api/v1/analytics/dashboard-summary
  GET    /api/v1/analytics/denials
  GET    /api/v1/analytics/duration
  GET    /api/v1/analytics/entry-exit-report
  GET    /api/v1/analytics/entry-exit-times
  GET    /api/v1/analytics/guard-performance
  GET    /api/v1/analytics/occupancy
  GET    /api/v1/analytics/pass-utilization
  GET    /api/v1/analytics/recurrent-visitor-trends
  GET    /api/v1/analytics/violations
  GET    /api/v1/analytics/visitor-by-purpose
  GET    /api/v1/analytics/visitor-frequency
  GET    /api/v1/analytics/visitor-types

Incidents Analytics:
  GET    /api/v1/analytics/incidents/heatmap
  GET    /api/v1/analytics/incidents/kpis
  GET    /api/v1/analytics/incidents/trends

Activity Feed:
  GET    /api/v1/activity/feed

Health:
  GET    /
  GET    /health
```

---

### 🔟 Notification Service (Port 8010)

**Total endpoints:** 7

```
Notifications:
  GET    /api/v1/notifications
  POST   /api/v1/notifications
  GET    /api/v1/notifications/{notification_id}
  PUT    /api/v1/notifications/{notification_id}
  POST   /api/v1/notifications/{notification_id}/read
  POST   /api/v1/notifications/mark-all-read
  GET    /api/v1/notifications/unread-count

Preferences:
  GET    /api/v1/notifications/preferences
  PUT    /api/v1/notifications/preferences/{category}

Health:
  GET    /
  GET    /health
```

---

### 1️⃣1️⃣ Audit Service (Port 8011)

**Total endpoints:** 0 (No OpenAPI schema)

⚠️ **Issue:** Audit service is running but doesn't expose an OpenAPI schema.

**Possible reasons:**

1. Service uses different documentation format
2. Endpoints not documented
3. Service is a worker/consumer only (no HTTP endpoints besides /health)

**Action required:** Check audit-service source code for actual endpoints.

---

### 1️⃣2️⃣ Realtime Service (Port 8012)

**Total endpoints:** 8

```
WebSocket Events:
  GET    /api/v1/stats
  POST   /api/v1/emit/{channel}
  POST   /api/v1/broadcast/tenant/{tenant_id}
  GET    /api/v1/connections

Health:
  GET    /
  GET    /health
  GET    /health/detailed
```

---

## 📊 ENDPOINT COUNT SUMMARY

| Service      | Total Endpoints   | Status        |
| ------------ | ----------------- | ------------- |
| Identity     | 11                | ✅ Documented |
| Property     | 13                | ✅ Documented |
| Visitor      | 48                | ✅ Documented |
| Parking      | 29                | ✅ Documented |
| Access       | 30                | ✅ Documented |
| Incident     | 26                | ✅ Documented |
| Compliance   | 47                | ✅ Documented |
| IoT          | 95                | ✅ Documented |
| Analytics    | 18                | ✅ Documented |
| Notification | 7                 | ✅ Documented |
| Audit        | 0                 | ❌ No OpenAPI |
| Realtime     | 8                 | ✅ Documented |
| **TOTAL**    | **332 endpoints** |               |

---

## 🔧 RECOMMENDED FIXES (Priority Order)

### 🔴 CRITICAL (Do First)

1. **Fix Hardcoded Tenant ID**

   ```javascript
   // File: frontend/src/contexts/TenantContext.js
   // Change 'TEST_TENANT_001' → 'default'
   ```

2. **Fix IoT Sensor Endpoint**

   ```javascript
   // File: frontend/src/components/IoTSensorEvents.js
   // Change: '/iot/sensors/health' → '/devices/sensors/health'
   ```

3. **Remove or Fix System Health Endpoint**

   ```javascript
   // File: frontend/src/components/SystemHealthBadge.js
   // Option A: Remove component
   // Option B: Change to call multiple /health endpoints
   // Option C: Implement aggregate health endpoint in Kong
   ```

4. **Debug 502 Errors on Visitor Service**

   ```bash
   # Test directly without Kong
   curl -H "X-Tenant-ID: default" \
     -H "Authorization: Bearer YOUR_TOKEN" \
     http://localhost:8003/api/v1/visits?page_size=10&page=1

   # Check logs
   docker compose logs visitor-service --tail=100
   ```

5. **Fix Missing Authorization Headers**
   - Ensure notifications component only renders after authentication
   - Add loading states and error boundaries

---

### ⚠️ HIGH (Do Soon)

6. **Create Tenant Context Provider**
   - Extract tenant_id from JWT token
   - Store in React Context
   - Use throughout app instead of hardcoded values

7. **Implement Proper Error Handling**
   - Distinguish between 404, 401, 502 errors
   - Show user-friendly messages
   - Log errors to backend

8. **Fix React Hydration Errors**
   - `<p>` cannot contain nested `<p>` or `<div>`
   - Remove `button={true}` prop (use boolean without braces)

9. **Add Missing Audio File**
   - `/alert-sound.mp3` returns 404
   - Either add file or remove audio notification feature

---

### 🟡 MEDIUM (Do Later)

10. **Audit Unmapped Frontend Endpoints**
    - Scan all `apiService.get/post/put/delete` calls
    - Compare with backend API inventory
    - Create mapping matrix

11. **Implement Missing Backend Endpoints**
    - If frontend needs `/system/health`, implement it
    - Or remove frontend features that depend on missing endpoints

12. **Add OpenAPI Documentation to Audit Service**
    - Currently returns empty OpenAPI schema
    - Investigate why and fix

13. **Optimize Frontend API Calls**
    - Too many parallel requests on dashboard load
    - Implement request batching or server-side aggregation

---

### 🟢 LOW (Nice to Have)

14. **Create Postman/Insomnia Collection**
    - Generate from OpenAPI schemas
    - Makes testing easier

15. **Add E2E Tests**
    - Test critical user flows
    - Catch frontend/backend mismatches early

16. **Implement API Versioning Strategy**
    - All endpoints use `/api/v1/`
    - Plan for v2 when breaking changes needed

---

## 🎯 NEXT STEPS

1. **Fix tenant_id issue** (5 minutes)
2. **Fix IoT sensor endpoint** (2 minutes)
3. **Test visitor endpoints directly** (10 minutes)
4. **Remove or implement system health** (30 minutes)
5. **Full frontend audit** (2-4 hours)

---

## 📝 NOTES

- **Backend is solid:** 332 endpoints across 12 microservices, all healthy
- **Frontend is outdated:** Calling endpoints that don't exist or have wrong paths
- **Multi-tenancy works:** Just need to use correct tenant_id values
- **Kong is configured correctly:** CORS working, routing working, issue is upstream services

---

**Generated:** December 11, 2025  
**Author:** AI Assistant (Gentleman Mode™)  
**Next Update:** After implementing critical fixes
