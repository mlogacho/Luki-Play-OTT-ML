# Luki Play OTT

Plataforma de streaming Over-The-Top (OTT) construida como monorepo con **Backend (NestJS)** y **Frontend (React Native / Expo)**. Diseñada con arquitectura limpia (Clean Architecture) y principios de Domain-Driven Design (DDD).

---

## Stack Tecnológico

| Capa | Tecnología |
|------|------------|
| **Backend** | NestJS · TypeScript · JWT (Passport) · Bcrypt · Jest |
| **Frontend** | React Native · Expo Router · NativeWind (Tailwind CSS) · TypeScript |
| **Infraestructura** | AWS EC2 (Debian) · PM2 · Nginx · Node.js 20 |

---

## Estructura del Proyecto

```
/
├── backend/                    API REST (NestJS)
│   └── src/
│       ├── main.ts             Punto de entrada
│       ├── app.module.ts       Módulo raíz
│       ├── common/             Filtros, pipes y decoradores globales
│       └── modules/
│           ├── auth/           Autenticación y autorización
│           ├── profiles/       Perfiles de usuario
│           ├── billing/        Gateway de facturación
│           ├── crm/            Gateway de CRM
│           └── access-control/ Permisos RBAC
│
├── frontend/                   App móvil y web (Expo)
│   ├── app/                    Rutas (Expo Router, file-based)
│   │   ├── (auth)/             Login y verificación OTP
│   │   ├── (app)/              Home, favoritos, búsqueda
│   │   ├── admin/              Panel de administración (CMS)
│   │   └── player/[id].tsx     Reproductor de contenido
│   ├── components/             Componentes reutilizables
│   └── services/               Stores de estado (auth, contenido, admin)
│
├── deploy-ec2.sh               Script de despliegue automatizado
└── docker-compose.yml          Orquestación Docker (en preparación)
```

---

## Arquitectura Backend

Cada módulo sigue la estructura de **Clean Architecture**:

```
módulo/
├── domain/          Entidades y puertos (interfaces)
├── application/     Casos de uso y DTOs
├── infrastructure/  Repositorios, adaptadores, servicios técnicos
└── presentation/    Controladores, guards, decoradores HTTP
```

### Módulo de Autenticación

Implementa tres flujos de autenticación:

| Flujo | Descripción | Uso |
|-------|-------------|-----|
| **Login App** | Contrato + email → OTP → tokens JWT | Usuarios móviles |
| **Login CMS** | Email + contraseña → tokens JWT | Administradores |
| **Login QR** | Generación de QR → confirmación → tokens JWT | Alternativa segura |

#### Gestión de Tokens JWT

- **Access Token** — 15 min, contiene `sub`, `role`, `permissions`, `accountId`, `entitlements`
- **Refresh Token** — 7 días, claims mínimos (`sub`, `aud`), hasheado con Bcrypt
- **Login Challenge Token** — 5 min, puente entre validación de credenciales y emisión de JWT

#### Seguridad Implementada

- **Rotación de tokens** — Cada refresh genera un par nuevo
- **Detección de reutilización** — Revoca todas las sesiones ante tokens fraudulentos
- **Hashing de refresh tokens** — Almacenados con Bcrypt, nunca en texto plano
- **Secretos JWT separados** — Claves diferentes para access/refresh/challenge
- **Expiración de sesiones** — Invalidación automática (CMS: 24h, App: 7d)
- **Tracking de dispositivo** — Sesiones vinculadas a `deviceId`

### Sistema de Autorización (RBAC + PBAC)

**Roles:**

| Rol | Permisos |
|-----|----------|
| `superadmin` | `cms:*`, `app:*` (acceso total) |
| `soporte` | `cms:support:*` (operaciones de soporte) |
| `cliente` | `app:content:read`, etc. (acceso a contenido) |

**Guards encadenables:**

1. `JwtAuthGuard` — Valida el bearer token
2. `RolesGuard` — Verifica el rol del usuario (`@Roles(...)`)
3. `PermissionsGuard` — Verifica permisos granulares (`@Permissions(...)`) con soporte de wildcards
4. `AudienceGuard` — Valida audiencia del JWT (APP vs CMS)

---

## Endpoints de la API

### Autenticación — `/auth`

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/auth/login-app` | Login móvil (contrato + email + deviceId) |
| POST | `/auth/otp/request` | Solicitar código OTP |
| POST | `/auth/otp/verify` | Verificar código OTP |
| POST | `/auth/login-app/complete` | Completar login tras OTP |
| POST | `/auth/login-cms` | Login de administrador (email + password) |
| POST | `/auth/refresh` | Renovar tokens |
| POST | `/auth/logout` | Cerrar sesión |
| POST | `/auth/password/change` | Cambiar contraseña |
| GET | `/auth/me` | Obtener usuario actual |
| GET | `/auth/sessions` | Listar sesiones activas |
| DELETE | `/auth/sessions/:id` | Revocar sesión |
| POST | `/auth/qr/init` | Iniciar login por QR |
| POST | `/auth/qr/confirm` | Confirmar login por QR |

**Respuesta de autenticación:**

```json
{
  "accessToken": "eyJhbG...",
  "refreshToken": "eyJhbG...",
  "canAccessOtt": true,
  "restrictionMessage": null,
  "expiresIn": 900
}
```

---

## Integraciones Externas

Se utiliza el patrón **Gateway** con implementaciones mock para desarrollo:

| Gateway | Propósito | Estado |
|---------|-----------|--------|
| `BillingGateway` | Consulta de suscripciones y entitlements | Mock (intercambiable) |
| `CrmGateway` | Sincronización de datos de usuario | Mock (intercambiable) |
| `OtpService` | Generación y envío de códigos OTP | Mock (intercambiable) |
| `HashService` | Hashing con Bcrypt | Implementado (`BcryptHashService`) |

Para producción, se intercambian las implementaciones mock por adaptadores reales en los respectivos módulos.

---

## Frontend

### Navegación (Expo Router)

```
Sin autenticar:
  └── (auth)/login → (auth)/verify-otp

Autenticado (cliente):
  └── (app)/home
  └── (app)/favorites
  └── (app)/search
  └── player/[id]

Autenticado (admin):
  └── admin/index → admin/panel → admin/form
```

### Componentes

| Componente | Descripción |
|------------|-------------|
| `Button` | Botón estilizado con manejo de eventos |
| `Input` | Campo de entrada con feedback de validación |
| `Hero` | Banner/showcase principal |
| `MediaRow` | Fila horizontal de contenido con thumbnails |

### State Management

| Store | Responsabilidad |
|-------|----------------|
| `authStore` | Tokens JWT, perfil, login/logout, estado OTP |
| `contentStore` | Catálogo, búsqueda, favoritos, entitlements |
| `adminStore` | Gestión de contenido y usuarios (CMS) |

---

## Persistencia

Actualmente utiliza **repositorios en memoria** (desarrollo). La arquitectura está preparada para migrar a cualquier base de datos (PostgreSQL, MySQL) mediante el cambio de implementación de repositorios, sin modificar la lógica de negocio.

**Entidades principales:**

- **User** — `id`, `email`, `contractNumber`, `role`, `passwordHash`, `accountId`, `isActive`
- **Session** — `id`, `userId`, `deviceId`, `audience`, `refreshTokenHash`, `expiresAt`
- **Account** — `id`, `canAccessOtt`, `restrictionMessage`

---

## Variables de Entorno

### Backend

```bash
# JWT
JWT_ACCESS_SECRET=clave-secreta-min-32-caracteres
JWT_REFRESH_SECRET=clave-secreta-diferente-min-32-caracteres
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# Servidor
PORT=8100
NODE_ENV=development

# Base de datos (cuando se migre de in-memory)
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=xxx
DB_NAME=luki_play_ott
```

### Frontend

```bash
EXPO_PUBLIC_API_URL=http://localhost:8100
EXPO_PUBLIC_APP_ENV=development
```

---

## Despliegue

### Script automatizado (`deploy-ec2.sh`)

El script realiza el despliegue completo en una instancia EC2 con Debian:

1. Instala dependencias del sistema (Git, Curl, Nginx, Node.js 20, PM2)
2. Clona/actualiza el repositorio
3. Construye el backend y lo ejecuta con PM2 en el puerto **8100**
4. Construye el frontend (web export) y lo sirve con Nginx en el puerto **8090**

```bash
# Ejecutar en la instancia EC2:
bash deploy-ec2.sh
```

Resultado:
- **Frontend:** `http://<IP_PÚBLICA>:8090`
- **Backend:** `http://<IP_PÚBLICA>:8100`

### Checklist de Producción

- [ ] Generar secretos JWT fuertes (>32 caracteres)
- [ ] Configurar HTTPS/SSL (Let's Encrypt + Nginx)
- [ ] Migrar a base de datos persistente (PostgreSQL)
- [ ] Integrar proveedor OTP real (Twilio, AWS SNS)
- [ ] Conectar APIs reales de CRM y Billing
- [ ] Configurar monitoreo y logging (CloudWatch, DataDog)
- [ ] Implementar CI/CD (GitHub Actions)
- [ ] Hardening de seguridad (firewall, security groups)

---

## Testing

```bash
cd backend

# Unit tests
npm run test

# E2E tests
npm run test:e2e

# Coverage
npm run test:cov
```

**Áreas cubiertas:** Guards (Permissions, Roles), Casos de uso (Login, OTP, Token Refresh), DTOs (validación), E2E (`test/app.e2e-spec.ts`).

---

## Desarrollo Local

```bash
# Backend
cd backend
npm install
npm run start:dev        # http://localhost:8100

# Frontend
cd frontend
npm install
npx expo start           # Expo Dev Server
npm run build:web         # Web export → web-build/
```

---

## Licencia

Proyecto privado — Todos los derechos reservados.
