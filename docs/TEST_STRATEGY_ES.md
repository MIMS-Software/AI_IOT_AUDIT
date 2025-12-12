# 🧪 ESTRATEGIA DE TESTING - AI_IOT MIMS MICROSERVICES

**Fecha:** 11 de Diciembre de 2025  
**Proyecto:** AI_IOT Multi-Tenant Intelligent Management System  
**Contexto:** 12 microservicios, 394 endpoints, 110K líneas - Cobertura actual de testing: ~20%  
**Timeline de Producción:** 2-3 meses

---

## 🎯 RESUMEN EJECUTIVO

### Estado Actual: 20% de Cobertura de Testing (GAP CRÍTICO)

Este proyecto tiene **CERO tests unitarios**, **CERO tests de integración**, y **CERO tests E2E**. Esto NO es aceptable para un sistema productivo que maneja dispositivos IoT, incidentes de seguridad, y datos con compliance GDPR.

### ¿POR QUÉ Estrategia de Testing ANTES de 100% Coverage?

Porque **NO TODOS LOS TESTS APORTAN EL MISMO VALOR**. Escribir tests "por coverage" es perder tiempo. Necesitamos **testing estratégico priorizado por ROI (Return on Investment)**.

---

## 🚨 ANTIPATRONES DE TESTING A EVITAR

### ❌ NO HAGAS ESTO:
- "Vamos a llegar a 100% de cobertura de unit tests" → Pérdida de tiempo en microservicios
- Testear endpoints CRUD → Sin valor de negocio
- Testear getters/setters → Vanity metrics
- Escribir tests DESPUÉS de que el código esté hecho → Los tests se convierten en validación, no en herramienta de diseño

### ✅ HAZ ESTO:
- Testear **flujos críticos de negocio** (autenticación, creación de incidentes, notificaciones)
- Testear **contratos entre servicios** (previene breaking changes)
- Testear **vulnerabilidades de seguridad** (rate limiting, validación JWT, XSS)
- Testear **lógica de negocio compleja** (cálculos, validaciones, máquinas de estado)

---

## 📊 PIRÁMIDE DE TESTING PARA MICROSERVICIOS

```
        ╱╲
       ╱  ╲          E2E Tests (10-15 tests)
      ╱────╲         - Lentos, frágiles, costosos
     ╱      ╲        - Solo happy paths críticos
    ╱────────╲       
   ╱          ╲      Integration Tests (50-100 tests)
  ╱────────────╲     - Velocidad media, alto valor
 ╱              ╲    - Flujos cross-service
╱────────────────╲   
╲                ╱   Contract Tests (100% inter-service)
 ╲──────────────╱    - CRÍTICO para microservicios
  ╲────────────╱     - Previene breaking changes
   ╲──────────╱      
    ╲────────╱       Unit Tests (Solo lógica compleja)
     ╲──────╱        - Rápidos, bajo valor en microservicios
      ╲────╱         - NO para CRUD/controllers
       ╲──╱
        ╲╱
```

**Insight clave:** En microservicios, **Integration y Contract tests aportan MÁS valor** que unit tests.

---

## 🔥 ESTRATEGIA DE TESTS BASADA EN PRIORIDAD

### Prioridad 1: CONTRACT TESTING (CRÍTICO) 🔴

**Timeline:** Semanas 1-3  
**Esfuerzo:** 2-3 semanas  
**ROI:** CRÍTICO - Previene que microservicios se rompan entre sí  
**Cobertura Objetivo:** 100% de comunicación inter-servicios

#### ¿POR QUÉ PRIMERO?

Tienes **12 microservicios** que se hablan entre sí. Si `identity-service` cambia un endpoint y rompe `incident-service`, tu sistema colapsa en producción.

#### Herramientas:
- **Pact** (Python: `pact-python`, Frontend: `@pact-foundation/pact`)
- **Spring Cloud Contract** (si migras a Java)

#### Qué Testear:

**Ejemplo 1: identity-service (proveedor) ↔ incident-service (consumidor)**

```python
# Contrato: POST /api/v1/auth/verify-token debe retornar user_id y roles

from pact import Consumer, Provider

pact = Consumer("incident-service").has_pact_with(Provider("identity-service"))

@pact.given('token válido existe')
@pact.upon_receiving('solicitud de verificación de token')
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
    # Consumidor valida que proveedor cumple contrato
    response = incident_service_client.verify_token('valid-jwt-token')
    assert response['user_id'] == 123
    assert 'admin' in response['roles']
```

**Ejemplo 2: incident-service (proveedor) ↔ notification-service (consumidor)**

```python
# Contrato: POST /api/v1/incidents debe retornar incident_id

@pact.given('usuario autenticado con acceso a propiedad')
@pact.upon_receiving('solicitud de crear incidente')
@pact.with_request(
    method='POST',
    path='/api/v1/incidents',
    body={'title': 'Incendio en Piso 3', 'severity': 'HIGH', 'property_id': 1}
)
@pact.will_respond_with(201, body={
    'id': 1,
    'title': 'Incendio en Piso 3',
    'status': 'OPEN',
    'created_at': '2025-12-11T10:00:00Z'
})
def test_create_incident_contract():
    response = notification_service_client.create_incident({...})
    assert response['id'] is not None
    assert response['status'] == 'OPEN'
```

#### Contratos Críticos a Implementar:

| Servicio Proveedor  | Servicio Consumidor  | Contrato de Endpoint                    |
|---------------------|---------------------|-----------------------------------------|
| identity-service    | TODOS los servicios | POST /auth/verify-token                 |
| property-service    | incident-service    | GET /properties/{id}                    |
| incident-service    | notification-service| POST /incidents (webhook)               |
| incident-service    | audit-service       | GET /incidents/{id}                     |
| iot-service         | incident-service    | POST /sensors/events                    |
| analytics-service   | incident-service    | GET /incidents (bulk query)             |

#### Roadmap de Implementación:

**Semana 1:** Setup Pact broker + CI/CD integration  
**Semana 2:** Implementar top 10 contratos críticos (auth, incidents, properties)  
**Semana 3:** Completar contratos restantes + documentación

---

### Prioridad 2: SECURITY TESTING (CRÍTICO) 🔴

**Timeline:** Semanas 1-2  
**Esfuerzo:** 1-2 semanas  
**ROI:** BLOQUEANTE - No se puede ir a producción sin arreglar 9 vulnerabilidades críticas  
**Cobertura Objetivo:** 100% de vulnerabilidades críticas documentadas en [SECURITY_AUDIT.md](./SECURITY_AUDIT_ES.md)

#### ¿POR QUÉ SEGUNDO?

Porque tienes **9 VULNERABILIDADES CRÍTICAS** que son **bloqueantes para producción**:
1. JWT sin verificación de firma (AUTHENTICATION BYPASS)
2. CORS wildcard (*)
3. Secret keys débiles por default
4. XSS via dangerouslySetInnerHTML
5. No rate limiting
6. Tokens en localStorage
7. SQL injection potential
8. Exception handlers vacíos
9. Passwords en logs

#### Herramientas:
- **Bandit** (Python SAST - Análisis Estático)
- **OWASP ZAP** (DAST - Análisis Dinámico)
- **pytest-security** (Tests de seguridad automatizados)
- **Semgrep** (Detección de vulnerabilidades basada en patrones)
- **npm audit** (Vulnerabilidades en dependencias del frontend)

#### Qué Testear:

**Test 1: Validación de Firma JWT (CRÍTICO)**

```python
import pytest
import jwt
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_jwt_debe_rechazar_firma_invalida():
    """
    CRÍTICO: JWT con firma inválida DEBE ser rechazado.
    Vulnerabilidad actual: middleware usa verify_signature=False
    """
    # Crear token con firma incorrecta
    fake_token = jwt.encode(
        {'user_id': 999, 'tenant_id': 'tenant_hacker'},
        key='wrong-secret',
        algorithm='HS256'
    )
    
    response = client.get(
        "/api/v1/protected-resource",
        headers={"Authorization": f"Bearer {fake_token}"}
    )
    
    # DEBE retornar 401 Unauthorized
    assert response.status_code == 401
    assert "Invalid token" in response.json()["detail"]
    
def test_jwt_debe_rechazar_token_expirado():
    """Tokens expirados deben ser rechazados"""
    expired_token = create_expired_jwt()
    
    response = client.post(
        "/api/v1/incidents",
        headers={"Authorization": f"Bearer {expired_token}"}
    )
    
    assert response.status_code == 401
    assert "Token expired" in response.json()["detail"]
```

**Test 2: Rate Limiting (CRÍTICO)**

```python
def test_login_rate_limiting():
    """
    Endpoint de login debe tener rate limiting (5 intentos/minuto).
    Vulnerabilidad actual: No hay rate limiting implementado
    """
    # Intentar 101 logins (debería limitarse a 100/minuto)
    responses = []
    for i in range(101):
        response = client.post("/api/v1/auth/login", json={
            "email": f"test{i}@test.com",
            "password": "password123"
        })
        responses.append(response)
    
    # Última petición debe ser limitada
    assert responses[-1].status_code == 429  # Too Many Requests
    assert "Rate limit exceeded" in responses[-1].json()["detail"]

def test_rate_limiting_por_ip():
    """Rate limiting debe ser por dirección IP"""
    # Simular peticiones desde diferentes IPs
    for ip in ['192.168.1.1', '192.168.1.2']:
        response = client.post(
            "/api/v1/auth/login",
            headers={"X-Forwarded-For": ip},
            json={"email": "test@test.com", "password": "password123"}
        )
        # Cada IP debería tener rate limit independiente
        assert response.status_code in [200, 401]  # No 429 en primera petición
```

**Test 3: Prevención de XSS (HIGH)**

```python
import pytest
from bs4 import BeautifulSoup

def test_prevencion_xss_en_comentarios():
    """
    Comentarios con tags <script> deben ser sanitizados.
    Vulnerabilidad actual: dangerouslySetInnerHTML sin DOMPurify
    """
    malicious_comment = {
        "text": "Buen incidente! <script>alert('XSS')</script>",
        "incident_id": 1
    }
    
    response = client.post(
        "/api/v1/incidents/1/comments",
        json=malicious_comment,
        headers=auth_headers
    )
    
    assert response.status_code == 201
    
    # Verificar que el tag script fue removido/escapado
    saved_comment = response.json()["text"]
    assert "<script>" not in saved_comment
    assert "alert" not in saved_comment or "&lt;script&gt;" in saved_comment

def test_prevencion_xss_en_titulo_incidente():
    """Títulos de incidentes deben ser sanitizados"""
    malicious_incident = {
        "title": "<img src=x onerror=alert('XSS')>",
        "severity": "HIGH",
        "property_id": 1
    }
    
    response = client.post("/api/v1/incidents", json=malicious_incident)
    assert response.status_code == 201
    
    # Script debe estar escapado
    assert "<img" not in response.json()["title"] or \
           "&lt;img" in response.json()["title"]
```

**Test 4: Prevención de SQL Injection (HIGH)**

```python
def test_sql_injection_en_busqueda():
    """
    Queries de búsqueda no deben ser vulnerables a SQL injection.
    Vulnerabilidad actual: f-string en schema name
    """
    malicious_search = "'; DROP TABLE incidents; --"
    
    response = client.get(
        f"/api/v1/incidents/search?q={malicious_search}",
        headers=auth_headers
    )
    
    # Debe retornar resultados vacíos o error, NO ejecutar SQL
    assert response.status_code in [200, 400]
    
    # Verificar que la tabla sigue existiendo
    check_response = client.get("/api/v1/incidents", headers=auth_headers)
    assert check_response.status_code == 200
```

**Test 5: Configuración CORS (CRÍTICO)**

```python
def test_cors_no_debe_permitir_wildcard():
    """
    CORS NO debe aceptar wildcard (*) en producción.
    Vulnerabilidad actual: CORS_ORIGINS por default es "*"
    """
    import os
    
    # Simular ambiente de producción
    os.environ['APP_ENV'] = 'production'
    os.environ['CORS_ORIGINS'] = '*'
    
    with pytest.raises(ValueError, match="CORS_ORIGINS must be explicitly set"):
        from app.main import app  # Debería fallar al iniciar

def test_cors_permite_solo_origenes_whitelisted():
    """CORS debería permitir solo orígenes configurados"""
    response = client.options(
        "/api/v1/incidents",
        headers={"Origin": "https://evil-site.com"}
    )
    
    # NO debería incluir evil-site.com en orígenes permitidos
    assert "https://evil-site.com" not in \
           response.headers.get("Access-Control-Allow-Origin", "")
```

#### Automatización de Tests de Seguridad con Bandit:

```bash
# Ejecutar Bandit en todos los servicios Python
bandit -r services/ -f json -o security-report.json

# Checks críticos:
# - B105: Hardcoded password
# - B201: Flask debug mode
# - B301: Pickle usage
# - B506: YAML load sin safe loader
# - B608: SQL injection
```

#### Roadmap de Implementación:

**Semana 1:**
- Día 1-2: Fix verificación JWT + escribir tests
- Día 3-4: Implementar rate limiting + escribir tests
- Día 5: Prevención XSS + escribir tests

**Semana 2:**
- Día 1-2: Fixes SQL injection + tests
- Día 3: Configuración CORS + tests
- Día 4-5: Escaneos Bandit + OWASP ZAP + remediación

---

### Prioridad 3: INTEGRATION TESTING (ALTA) 🟡

**Timeline:** Semanas 4-7  
**Esfuerzo:** 3-4 semanas  
**ROI:** ALTO - Valida flujos completos de negocio cross-service  
**Cobertura Objetivo:** Top 20 flujos críticos (regla 80/20)

#### ¿POR QUÉ TERCERO?

Los tests de integración validan **flujos de negocio end-to-end** que cruzan múltiples servicios:
- Autenticación usuario → Token → Acceso a recursos protegidos
- Crear incidente → Disparar notificación → Registrar en audit log
- Evento sensor IoT → Creación incidente → Actualización dashboard

#### Herramientas:
- **pytest** con fixtures de base de datos
- **TestContainers** (Contenedores Docker para tests)
- **FastAPI TestClient**
- **pytest-asyncio** (para endpoints async)

#### Qué Testear:

**Test 1: Flujo Completo de Creación de Incidente**

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def test_db():
    """Setup base de datos de test con Docker"""
    with PostgresContainer("postgres:15") as postgres:
        engine = create_engine(postgres.get_connection_url())
        # Ejecutar migraciones
        run_alembic_migrations(engine)
        yield engine

@pytest.fixture
def auth_headers(test_db):
    """Crear usuario autenticado y retornar headers de auth"""
    user = create_test_user(test_db, email="test@test.com", role="admin")
    token = generate_jwt_token(user)
    return {"Authorization": f"Bearer {token}"}

@pytest.mark.integration
def test_crear_incidente_dispara_notificacion_y_auditoria(
    test_db, 
    auth_headers,
    mock_notification_service
):
    """
    Test de integración: Crear incidente debería:
    1. Guardar en base de datos
    2. Disparar servicio de notificaciones
    3. Crear entrada en audit log
    """
    # 1. Crear incidente
    incident_data = {
        "title": "Incendio en Piso 3",
        "severity": "HIGH",
        "property_id": 1,
        "location_details": "Edificio A, Habitación 301"
    }
    
    response = client.post(
        "/api/v1/incidents",
        json=incident_data,
        headers=auth_headers
    )
    
    assert response.status_code == 201
    incident = response.json()
    incident_id = incident["id"]
    
    # 2. Verificar que se llamó al servicio de notificaciones
    mock_notification_service.send_notification.assert_called_once()
    call_args = mock_notification_service.send_notification.call_args
    assert call_args[0]['incident_id'] == incident_id
    assert call_args[0]['severity'] == 'HIGH'
    
    # 3. Verificar que se creó audit log
    audit_logs = test_db.query(AuditLog).filter_by(
        action="CREATE_INCIDENT",
        resource_id=incident_id
    ).all()
    
    assert len(audit_logs) == 1
    assert audit_logs[0].user_id == 123  # Del token de auth
    assert audit_logs[0].changes['title'] == "Incendio en Piso 3"
    
    # 4. Verificar que el incidente es recuperable
    get_response = client.get(
        f"/api/v1/incidents/{incident_id}",
        headers=auth_headers
    )
    assert get_response.status_code == 200
    assert get_response.json()["title"] == "Incendio en Piso 3"
```

**Test 2: Flujo de Autenticación**

```python
@pytest.mark.integration
def test_flujo_completo_autenticacion(test_db):
    """
    Testear flujo completo de auth:
    1. Usuario se registra
    2. Usuario hace login
    3. Token JWT es emitido
    4. Token otorga acceso a recursos protegidos
    5. Token expira después de timeout
    """
    # 1. Registrar usuario
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
    
    # 3. Extraer token
    access_token = login_response.json()["access_token"]
    assert access_token is not None
    
    # 4. Acceder a recurso protegido
    protected_response = client.get(
        "/api/v1/incidents",
        headers={"Authorization": f"Bearer {access_token}"}
    )
    assert protected_response.status_code == 200
    
    # 5. Verificar que token expira (simular time travel)
    with freeze_time(datetime.now() + timedelta(hours=25)):
        expired_response = client.get(
            "/api/v1/incidents",
            headers={"Authorization": f"Bearer {access_token}"}
        )
        assert expired_response.status_code == 401
```

**Test 3: Aislamiento Multi-Tenancy**

```python
@pytest.mark.integration
def test_aislamiento_datos_multi_tenancy(test_db):
    """
    Verificar que tenants no pueden acceder datos de otros.
    Crítico para compliance GDPR.
    """
    # Crear dos tenants
    tenant_a_user = create_test_user(test_db, tenant="tenant_a")
    tenant_b_user = create_test_user(test_db, tenant="tenant_b")
    
    token_a = generate_jwt_token(tenant_a_user)
    token_b = generate_jwt_token(tenant_b_user)
    
    # Tenant A crea incidente
    incident_a = client.post("/api/v1/incidents", json={
        "title": "Incidente Tenant A",
        "severity": "LOW",
        "property_id": 1
    }, headers={"Authorization": f"Bearer {token_a}"}).json()
    
    # Tenant B crea incidente
    incident_b = client.post("/api/v1/incidents", json={
        "title": "Incidente Tenant B",
        "severity": "HIGH",
        "property_id": 2
    }, headers={"Authorization": f"Bearer {token_b}"}).json()
    
    # Tenant A NO debería ver incidente de Tenant B
    tenant_a_incidents = client.get(
        "/api/v1/incidents",
        headers={"Authorization": f"Bearer {token_a}"}
    ).json()
    
    incident_ids = [inc['id'] for inc in tenant_a_incidents]
    assert incident_a['id'] in incident_ids
    assert incident_b['id'] not in incident_ids  # NO DEBE filtrar
    
    # Tenant A NO debería acceder a incidente de Tenant B por ID
    forbidden_response = client.get(
        f"/api/v1/incidents/{incident_b['id']}",
        headers={"Authorization": f"Bearer {token_a}"}
    )
    assert forbidden_response.status_code == 404  # O 403
```

#### Top 20 Flujos Críticos a Testear:

1. ✅ Flujo autenticación usuario (register → login → access)
2. ✅ Creación incidente → Notificación → Auditoría
3. ✅ Aislamiento de datos multi-tenancy
4. Gestión de propiedades (create → assign → list)
5. Flujo check-in visitante (código QR → validación → acceso otorgado)
6. Escalamiento incidente emergencia (HIGH → CRITICAL → Notificar autoridades)
7. Evento sensor IoT → Auto-creación de incidente
8. Agregación de datos dashboard analytics
9. Exportación datos GDPR (usuario solicita sus datos)
10. Eliminación datos GDPR (derecho al olvido)
11. Control de acceso basado en roles (admin vs resident vs staff)
12. Flujo reserva de parking
13. Generación de reportes de compliance
14. Actualizaciones en tiempo real via WebSocket
15. Hilo de comentarios en incidente (con menciones @user)
16. Subida archivo → Escaneo virus → Almacenamiento
17. Flujo reset de contraseña (email → token → nueva contraseña)
18. Autenticación de dos factores (si implementado)
19. Trail de audit log (cada acción registrada)
20. Cascada de health checks del sistema

#### Roadmap de Implementación:

**Semana 4:** Setup TestContainers + fixtures  
**Semana 5:** Implementar flujos 1-10  
**Semana 6:** Implementar flujos 11-20  
**Semana 7:** Refinar + integración CI/CD

---

### Prioridad 4: END-TO-END TESTING (MEDIA) 🟢

**Timeline:** Semanas 8-9  
**Esfuerzo:** 2-3 semanas  
**ROI:** MEDIO - Valida experiencia de usuario pero lento y frágil  
**Cobertura Objetivo:** 10-15 happy paths críticos SOLAMENTE

#### ¿POR QUÉ CUARTO?

Los tests E2E validan la **experiencia completa del usuario** (Frontend + Backend), pero son:
- **LENTOS** (toman minutos vs segundos para unit tests)
- **FRÁGILES** (se rompen fácilmente con cambios de UI)
- **COSTOSOS** (requieren infraestructura de automatización de browsers)

Úsalos SOLO para **happy paths críticos**, no para edge cases.

#### Herramientas:
- **Playwright** (RECOMENDADO - mejor para microservicios que Cypress)
- **Selenium** (si testing cross-browser es crítico)
- **Docker Compose** (para E2E de sistema completo)

#### Qué Testear:

**Test 1: Usuario Puede Crear y Ver Incidente**

```javascript
// tests/e2e/incident-flow.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Gestión de Incidentes', () => {
  test.beforeEach(async ({ page }) => {
    // Login antes de cada test
    await page.goto('http://localhost:3000/login');
    await page.fill('input[name="email"]', 'admin@test.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    
    // Esperar redirect a dashboard
    await expect(page).toHaveURL('http://localhost:3000/dashboard');
  });

  test('Usuario puede crear incidente y verlo en lista', async ({ page }) => {
    // 1. Navegar a crear incidente
    await page.click('a[href="/incidents/new"]');
    await expect(page).toHaveURL('http://localhost:3000/incidents/new');
    
    // 2. Llenar formulario
    await page.fill('input[name="title"]', 'E2E Test Incident');
    await page.selectOption('select[name="severity"]', 'HIGH');
    await page.selectOption('select[name="property_id"]', '1');
    await page.fill('textarea[name="description"]', 'Este es un incidente de test');
    
    // 3. Enviar formulario
    await page.click('button[type="submit"]');
    
    // 4. Verificar mensaje de éxito
    await expect(page.locator('text=Incidente creado exitosamente')).toBeVisible();
    
    // 5. Navegar a lista de incidentes
    await page.click('a[href="/incidents"]');
    
    // 6. Verificar que incidente aparece en lista
    await expect(page.locator('text=E2E Test Incident')).toBeVisible();
    
    // 7. Click en incidente para ver detalles
    await page.click('text=E2E Test Incident');
    
    // 8. Verificar página de detalles
    await expect(page.locator('h1:has-text("E2E Test Incident")')).toBeVisible();
    await expect(page.locator('text=HIGH')).toBeVisible();
  });

  test('Usuario puede agregar comentario a incidente', async ({ page }) => {
    // Asume que incidente existe del test anterior o fixture
    await page.goto('http://localhost:3000/incidents/1');
    
    // Scroll a sección de comentarios
    await page.locator('textarea[placeholder*="comentario"]').scrollIntoViewIfNeeded();
    
    // Agregar comentario
    await page.fill('textarea[placeholder*="comentario"]', 'Comentario de test E2E');
    await page.click('button:has-text("Publicar Comentario")');
    
    // Verificar que comentario aparece
    await expect(page.locator('text=Comentario de test E2E')).toBeVisible({
      timeout: 5000
    });
  });
});
```

**Test 2: Flujo Completo Login a Dashboard**

```javascript
test('Usuario puede hacer login y ver dashboard', async ({ page }) => {
  // 1. Navegar a login
  await page.goto('http://localhost:3000/login');
  
  // 2. Ingresar credenciales
  await page.fill('input[name="email"]', 'admin@test.com');
  await page.fill('input[name="password"]', 'password123');
  
  // 3. Click login
  await page.click('button[type="submit"]');
  
  // 4. Verificar redirect a dashboard
  await expect(page).toHaveURL('http://localhost:3000/dashboard');
  
  // 5. Verificar que widgets del dashboard cargan
  await expect(page.locator('text=Total de Incidentes')).toBeVisible();
  await expect(page.locator('text=Propiedades Activas')).toBeVisible();
  await expect(page.locator('text=Actividad Reciente')).toBeVisible();
  
  // 6. Verificar menú de navegación
  await expect(page.locator('a[href="/incidents"]')).toBeVisible();
  await expect(page.locator('a[href="/properties"]')).toBeVisible();
  
  // 7. Verificar info de usuario en header
  await expect(page.locator('text=admin@test.com')).toBeVisible();
});
```

**Test 3: Visual Regression Testing**

```javascript
test('Regresión visual del dashboard', async ({ page }) => {
  await page.goto('http://localhost:3000/dashboard');
  
  // Esperar a que todos los widgets carguen
  await page.waitForSelector('[data-testid="dashboard-loaded"]');
  
  // Tomar screenshot y comparar con baseline
  await expect(page).toHaveScreenshot('dashboard-desktop.png', {
    fullPage: true,
    maxDiffPixels: 100  // Permitir pequeñas diferencias
  });
});
```

#### 10 Tests E2E Críticos:

1. ✅ Login → Dashboard → Logout
2. ✅ Crear incidente → Ver en lista → Ver detalles
3. Agregar comentario a incidente
4. Crear propiedad → Asignar residentes
5. Check-in de visitante con código QR
6. Cambiar severidad de incidente → Verificar notificación
7. Buscar incidentes por palabra clave
8. Filtrar incidentes por rango de fechas
9. Exportar incidentes a CSV
10. Configuración de perfil → Cambiar contraseña

#### Roadmap de Implementación:

**Semana 8:** Setup Playwright + ambiente E2E con Docker Compose  
**Semana 9:** Implementar 10 tests + integración CI/CD

---

### Prioridad 5: UNIT TESTING (BAJA PRIORIDAD) ⚪

**Timeline:** Semanas 10-13 (si hay tiempo)  
**Esfuerzo:** 4-6 semanas (si se hace todo)  
**ROI:** BAJO en microservicios - Focus SOLO en lógica de negocio compleja  
**Cobertura Objetivo:** ~30-40% (NO 100%)

#### ¿POR QUÉ AL FINAL?

En **arquitectura de microservicios**, los unit tests tienen MENOS valor que en monolitos porque:
- La mayoría del código es **orquestación** (llamar otros servicios) → Testear con integration tests
- Los controllers son **delgados** (solo manejo HTTP) → Testear con integration tests
- Los repositories son **CRUD** (solo operaciones de BD) → Testear con integration tests

**SOLO escribe unit tests para:**
- ✅ **Lógica de negocio compleja** (cálculos, validaciones, algoritmos)
- ✅ **Funciones puras** (sin dependencias, determinísticas)
- ✅ **Utilidades** (formateo de fechas, manipulación de strings, parsers)

**NO escribas unit tests para:**
- ❌ Controllers/Routers (usa integration tests)
- ❌ Repositories (usa integration tests con BD real)
- ❌ Services que solo orquestan llamadas (usa contract tests)
- ❌ Getters/Setters (vanity metrics)

#### Qué Testear:

**Test 1: Lógica de Negocio Compleja (Cálculo de Severidad de Incidente)**

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
        time_of_day: int  # 0-23 horas
    ) -> Severity:
        """
        Lógica de negocio compleja para calcular severidad de incidente.
        ESTO merece un unit test.
        """
        if fire_detected:
            return Severity.CRITICAL
        
        if smoke_level > 80 or temperature > 100:
            return Severity.CRITICAL
        
        if smoke_level > 50 or temperature > 60:
            # Mayor severidad de noche (cuando la gente duerme)
            if 22 <= time_of_day or time_of_day <= 6:
                return Severity.HIGH
            return Severity.MEDIUM
        
        return Severity.LOW

# tests/unit/test_incident_analyzer.py
import pytest
from app.domain.incident_analyzer import IncidentAnalyzer, Severity

class TestIncidentSeverityCalculation:
    def test_fuego_detectado_siempre_critico(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=True,
            smoke_level=10,
            temperature=20,
            time_of_day=12
        )
        assert severity == Severity.CRITICAL
    
    def test_nivel_humo_alto_critico(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=85,
            temperature=25,
            time_of_day=14
        )
        assert severity == Severity.CRITICAL
    
    def test_temperatura_alta_critica(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=30,
            temperature=105,
            time_of_day=10
        )
        assert severity == Severity.CRITICAL
    
    def test_humo_medio_dia_es_medio(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=60,
            temperature=40,
            time_of_day=14  # Tarde
        )
        assert severity == Severity.MEDIUM
    
    def test_humo_medio_noche_es_alto(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=60,
            temperature=40,
            time_of_day=23  # Noche
        )
        assert severity == Severity.HIGH
    
    def test_humo_bajo_es_severidad_baja(self):
        severity = IncidentAnalyzer.calculate_severity(
            fire_detected=False,
            smoke_level=20,
            temperature=25,
            time_of_day=10
        )
        assert severity == Severity.LOW
```

**Test 2: Funciones Utilitarias (Formateo de Fechas)**

```python
# app/utils/date_helpers.py
from datetime import datetime, timezone

def format_relative_time(dt: datetime) -> str:
    """Convertir datetime a tiempo relativo legible"""
    now = datetime.now(timezone.utc)
    diff = now - dt
    
    if diff.seconds < 60:
        return "justo ahora"
    elif diff.seconds < 3600:
        minutes = diff.seconds // 60
        return f"hace {minutes} minuto{'s' if minutes > 1 else ''}"
    elif diff.days == 0:
        hours = diff.seconds // 3600
        return f"hace {hours} hora{'s' if hours > 1 else ''}"
    elif diff.days == 1:
        return "ayer"
    elif diff.days < 30:
        return f"hace {diff.days} días"
    else:
        return dt.strftime("%d de %B, %Y")

# tests/unit/test_date_helpers.py
import pytest
from datetime import datetime, timedelta, timezone
from app.utils.date_helpers import format_relative_time

def test_justo_ahora():
    dt = datetime.now(timezone.utc) - timedelta(seconds=30)
    assert format_relative_time(dt) == "justo ahora"

def test_hace_minutos():
    dt = datetime.now(timezone.utc) - timedelta(minutes=5)
    assert format_relative_time(dt) == "hace 5 minutos"

def test_hace_horas():
    dt = datetime.now(timezone.utc) - timedelta(hours=3)
    assert format_relative_time(dt) == "hace 3 horas"

def test_ayer():
    dt = datetime.now(timezone.utc) - timedelta(days=1)
    assert format_relative_time(dt) == "ayer"

def test_hace_dias():
    dt = datetime.now(timezone.utc) - timedelta(days=5)
    assert format_relative_time(dt) == "hace 5 días"
```

#### Roadmap de Implementación:

**Semanas 10-13:** Implementar unit tests SOLO para lógica compleja (si hay tiempo)

---

### Prioridad 6: PERFORMANCE TESTING (OPCIONAL) 🔵

**Timeline:** Semana 14 (solo si despliegue a producción es inminente)  
**Esfuerzo:** 1 semana  
**ROI:** OPCIONAL - Solo si se espera carga alta  
**Cobertura Objetivo:** Top 20% de endpoints (80% del tráfico)

#### ¿POR QUÉ OPCIONAL?

Primero, haz que el sistema **FUNCIONE CORRECTAMENTE**. Luego, hazlo **RÁPIDO**.

Pero si vas a desplegar a producción con **dispositivos IoT, drones y sensores**, NECESITAS saber si tu sistema puede aguantar la carga.

#### Herramientas:
- **Locust** (Python - fácil de configurar)
- **k6** (Grafana - más potente)
- **Apache JMeter** (estándar enterprise)

#### Qué Testear:

**Test 1: Creación Concurrente de Incidentes**

```python
# locust_tests/incident_load.py
from locust import HttpUser, task, between
import random

class IncidentUser(HttpUser):
    wait_time = between(1, 3)  # Esperar 1-3 segundos entre peticiones
    
    def on_start(self):
        """Login una vez por usuario"""
        response = self.client.post("/api/v1/auth/login", json={
            "email": f"loadtest{random.randint(1, 100)}@test.com",
            "password": "password123"
        })
        self.token = response.json()["access_token"]
    
    @task(3)  # 3x más probable que otras tasks
    def create_incident(self):
        """Simular creación de incidentes"""
        self.client.post(
            "/api/v1/incidents",
            json={
                "title": f"Incidente de Carga {random.randint(1, 10000)}",
                "severity": random.choice(["LOW", "MEDIUM", "HIGH", "CRITICAL"]),
                "property_id": random.randint(1, 10)
            },
            headers={"Authorization": f"Bearer {self.token}"}
        )
    
    @task(1)
    def list_incidents(self):
        """Simular listado de incidentes"""
        self.client.get(
            "/api/v1/incidents",
            headers={"Authorization": f"Bearer {self.token}"}
        )
    
    @task(1)
    def view_dashboard(self):
        """Simular visualización de dashboard"""
        self.client.get(
            "/api/v1/analytics/dashboard",
            headers={"Authorization": f"Bearer {self.token}"}
        )

# Ejecutar con: locust -f incident_load.py --host=http://localhost:8000 --users 1000 --spawn-rate 50
```

**Test 2: Carga de Eventos Sensor IoT**

```python
class IoTSensorUser(HttpUser):
    wait_time = between(0.1, 0.5)  # Alta frecuencia
    
    @task
    def send_sensor_event(self):
        """Simular sensores IoT enviando eventos"""
        self.client.post("/api/v1/iot/sensor-events", json={
            "sensor_id": f"sensor_{random.randint(1, 1000)}",
            "event_type": "SMOKE_DETECTED",
            "value": random.uniform(0, 100),
            "timestamp": datetime.now().isoformat()
        })
```

#### Targets de Performance:

| Endpoint              | Tiempo Respuesta Target | Throughput Target |
|-----------------------|-------------------------|-------------------|
| GET /incidents        | < 200ms (p95)           | 1000 req/s        |
| POST /incidents       | < 500ms (p95)           | 500 req/s         |
| POST /auth/login      | < 300ms (p95)           | 200 req/s         |
| GET /analytics        | < 1000ms (p95)          | 100 req/s         |
| POST /iot/events      | < 100ms (p95)           | 5000 req/s        |

---

## 📊 ROADMAP DE IMPLEMENTACIÓN DE TESTING

### Resumen de Timeline (14 Semanas)

```
Semana 1-3:  Contract Testing (CRÍTICO)         🔴
Semana 1-2:  Security Testing (CRÍTICO)         🔴
Semana 4-7:  Integration Testing (ALTA)         🟡
Semana 8-9:  E2E Testing (MEDIA)                🟢
Semana 10-13: Unit Testing (BAJA - opcional)    ⚪
Semana 14:   Performance Testing (OPCIONAL)     🔵
```

### Enfoque por Fases

#### Fase 1: Fundación (Semanas 1-3) - BLOQUEANTE

**Objetivo:** Prevenir que microservicios se rompan entre sí + Arreglar vulnerabilidades de seguridad  
**Equipo:** 2-3 desarrolladores  
**Entregables:**
- [ ] Setup de Pact broker
- [ ] 20 contratos críticos implementados
- [ ] Las 9 vulnerabilidades de seguridad arregladas con tests
- [ ] Pipeline CI/CD con contract + security tests

**Criterios de Éxito:**
- ✅ Cero breaking changes entre servicios
- ✅ Todos los escaneos Bandit/OWASP ZAP pasan
- ✅ Rate limiting activo en todos los endpoints de login

---

#### Fase 2: Flujos Críticos (Semanas 4-7) - ALTA PRIORIDAD

**Objetivo:** Validar que funcionalidad core de negocio funciona end-to-end  
**Equipo:** 2-3 desarrolladores  
**Entregables:**
- [ ] Setup de TestContainers
- [ ] Top 20 integration tests implementados
- [ ] Aislamiento multi-tenancy verificado
- [ ] Flujos GDPR compliance testeados

**Criterios de Éxito:**
- ✅ Todos los flujos críticos en verde (auth, incidents, notifications)
- ✅ No hay data leaks entre tenants
- ✅ 60% integration test coverage

---

#### Fase 3: Experiencia de Usuario (Semanas 8-9) - MEDIA PRIORIDAD

**Objetivo:** Asegurar que frontend funciona correctamente con backend  
**Equipo:** 1-2 desarrolladores frontend  
**Entregables:**
- [ ] Setup de Playwright
- [ ] 10 E2E tests para happy paths
- [ ] Visual regression tests
- [ ] CI/CD con E2E tests

**Criterios de Éxito:**
- ✅ Flujo Login → Dashboard → Crear Incidente funciona
- ✅ No hay regresiones visuales
- ✅ E2E tests corren en cada PR

---

#### Fase 4: Edge Cases (Semanas 10-13) - BAJA PRIORIDAD

**Objetivo:** Cubrir lógica de negocio compleja con unit tests  
**Equipo:** 1 desarrollador  
**Entregables:**
- [ ] Unit tests para lógica compleja
- [ ] 30-40% code coverage (NO 100%)
- [ ] Focus en capa de dominio

**Criterios de Éxito:**
- ✅ Todos los cálculos complejos testeados
- ✅ Utilidades cubiertas

---

#### Fase 5: Production Readiness (Semana 14) - OPCIONAL

**Objetivo:** Validar que sistema puede manejar carga de producción  
**Equipo:** 1 ingeniero DevOps  
**Entregables:**
- [ ] Tests de performance con Locust
- [ ] Reporte de load testing
- [ ] Identificación de bottlenecks

**Criterios de Éxito:**
- ✅ Sistema maneja 1000 usuarios concurrentes
- ✅ Tiempos de respuesta p95 < targets

---

## 🎯 MÉTRICAS Y KPIs DE TESTING

### Objetivos de Cobertura

| Tipo de Test   | Cobertura Target | Actual | Gap  |
|----------------|------------------|--------|------|
| Contract       | 100%             | 0%     | 100% |
| Security       | 100% vulns       | 0%     | 100% |
| Integration    | 80% flows        | 0%     | 80%  |
| E2E            | 10-15 tests      | 0      | 15   |
| Unit           | 30-40%           | 0%     | 40%  |
| **OVERALL**    | **60-70%**       | **20%**| **50%** |

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
    # BLOQUEANTE: Debe pasar para hacer merge

  security-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Run Bandit
        run: bandit -r services/
      - name: Run OWASP ZAP
        run: docker run owasp/zap scan
    # BLOQUEANTE: Debe pasar para hacer merge

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Start TestContainers
        run: docker-compose -f docker-compose.test.yml up -d
      - name: Run Integration Tests
        run: pytest tests/integration/
    # BLOQUEANTE: Debe pasar para hacer merge

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Start Full System
        run: docker-compose up -d
      - name: Run Playwright
        run: npx playwright test
    # WARNING: Puede ser salteado si urgente

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Run Unit Tests
        run: pytest tests/unit/
    # INFO: Solo reporte de coverage
```

---

## 🚀 PRÓXIMOS PASOS

### Inmediato (Esta Semana)

1. [ ] **Leer** esta estrategia con equipo técnico
2. [ ] **Aprobar** orden de prioridades y timeline
3. [ ] **Asignar** 2-3 desarrolladores al equipo de testing
4. [ ] **Setup** de Pact broker y infraestructura TestContainers

### Semana 1-2 (CRÍTICO)

1. [ ] Implementar contract tests para top 10 pares de servicios
2. [ ] Arreglar las 9 vulnerabilidades de seguridad
3. [ ] Escribir security tests para cada vulnerabilidad
4. [ ] Setup pipeline CI/CD con contract + security tests

### Mes 1 (ALTA)

1. [ ] Completar todos los contract tests (100% inter-service)
2. [ ] Implementar top 20 integration tests
3. [ ] Verificar aislamiento multi-tenancy
4. [ ] Security audit después de fixes

### Mes 2-3 (MEDIA)

1. [ ] E2E tests para happy paths
2. [ ] Unit tests para lógica compleja (opcional)
3. [ ] Performance testing (opcional)
4. [ ] **GO-LIVE** a producción

---

## 📞 RECOMENDACIONES

### HAZ:
- ✅ Empieza con contract tests (previene breaking changes)
- ✅ Arregla vulnerabilidades de seguridad con tests (bloqueante producción)
- ✅ Focus en integration tests (alto ROI)
- ✅ Escribe E2E tests solo para happy paths (no exageres)
- ✅ Usa TestContainers para integration tests realistas

### NO HAGAS:
- ❌ Perseguir 100% code coverage (vanity metric)
- ❌ Escribir unit tests para controllers (usa integration tests)
- ❌ Escribir unit tests para repositories (usa integration tests)
- ❌ Escribir E2E tests para edge cases (demasiado lentos)
- ❌ Retrasar producción por unit test coverage

---

## 📚 REFERENCIAS

### Documentación
- [Auditoría de Seguridad](./SECURITY_AUDIT_ES.md) - 9 vulnerabilidades críticas a arreglar
- [Auditoría de Calidad de Código](./CODE_QUALITY_AUDIT_ES.md) - Análisis de deuda técnica
- [Auditoría de Funcionalidad](./FUNCTIONALITY_AUDIT_ES.md) - 394 endpoints a testear

### Herramientas
- **Pact:** https://docs.pact.io/
- **pytest:** https://docs.pytest.org/
- **Playwright:** https://playwright.dev/
- **Locust:** https://docs.locust.io/
- **Bandit:** https://bandit.readthedocs.io/
- **OWASP ZAP:** https://www.zaproxy.org/docs/

---

**Fecha de auditoría:** 11 de Diciembre de 2025  
**Próxima revisión:** Después de completar Fase 1 (Semana 3)
