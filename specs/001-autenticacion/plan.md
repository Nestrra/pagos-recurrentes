# Implementation Plan: Autenticación (registro, ingreso y cierre de sesión)

**Branch**: `001-autenticacion` | **Date**: 2026-10-26 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-autenticacion/spec.md`

**Artefactos de este plan**: [research.md](./research.md) · [data-model.md](./data-model.md) · [contracts/openapi.yaml](./contracts/openapi.yaml) · [quickstart.md](./quickstart.md)

## Summary

Construir la puerta de entrada de la app: registro con email y contraseña, ingreso y cierre de sesión por dispositivo, con aislamiento de datos por cuenta. Este plan incluye también **el esqueleto del monorepo y el CI mínimo**, porque es la primera feature.

Enfoque técnico (detalle y alternativas en [research.md](./research.md)):

- **API NestJS + Prisma + PostgreSQL** con módulos `auth`, `sessions`, `throttling`, `users` y `common`.
- **Contraseñas** con argon2id (OWASP: 19 MiB, t=2, p=1) y una **lista local de contraseñas prohibidas** (SecLists + complemento en español) cargada en memoria. Contraseña normalizada a NFC (FR-031), longitud en puntos de código y tope de 1 024 bytes (FR-032).
- **Sesión**: JWT de acceso de 15 min + token de renovación opaco (SHA-256 en BD), rotado en cada uso, con detección de reutilización y gracia de 30 s. Un guard global consulta la sesión en cada solicitud: así el logout es inmediato (SC-005) y cada uso renueva los 30 días (FR-021).
- **Límites** de FR-016 a FR-019 en tablas de PostgreSQL, con advisory lock por email. El origen sale de `CF-Connecting-IP`, con IPv6 agrupado por /64 y guardado como HMAC.
- **Reloj inyectable** (`CLOCK`) en todo cálculo temporal, incluida la verificación del JWT.
- **App Expo** con Expo Router, Zustand (estado de sesión), TanStack Query (llamadas) y expo-secure-store (tokens).

## Technical Context

**Language/Version**: TypeScript (`strict: true`) sobre Node.js 24 LTS (API) y Expo SDK estable vigente / React Native (app)

**Primary Dependencies**:

- API: NestJS, `@nestjs/jwt`, Prisma, `argon2`, `class-validator` + `class-transformer`.
- App: Expo, Expo Router, `expo-secure-store`, Zustand, `@tanstack/react-query`.
- Versiones exactas: se fijan en la tarea de esqueleto, **con confirmación previa**.

**Storage**: PostgreSQL 18 (único servicio). Esquema solo vía migraciones de Prisma. Marcas de tiempo `timestamptz` en UTC.

**Testing**:

- API: Jest + Supertest. Unitarias con `FakeClock`; e2e contra un Postgres real de pruebas.
- App: Jest (`jest-expo`) + React Native Testing Library.
- E2E móvil: Maestro, en local.

**Target Platform**:

- API: contenedor Linux detrás de Cloudflare Tunnel (staging y producción); local con Docker Compose para Postgres.
- App: iOS y Android vía Expo/EAS.

**Project Type**: mobile-app + API (monorepo con npm workspaces)

**Performance Goals**:

- Registro e ingreso p95 < 1 s en staging (dominado por argon2id, objetivo < 250 ms por hash).
- SC-001: registro completo < 2 min. SC-002: ingreso < 30 s (e2e).

**Constraints**:

- Sin servicios aparte de PostgreSQL ni servicios externos de pago.
- Origen = `CF-Connecting-IP`.
- Tiempo inyectable.
- Logs sin credenciales ni datos personales.
- Tokens solo en almacenamiento seguro del sistema operativo.

**Scale/Scope**: proyecto de práctica, decenas o cientos de cuentas, una réplica de API. 3 pantallas (Registro, Ingreso, Inicio) y 6 endpoints.

**NEEDS CLARIFICATION**: ninguno. Los cambios a la spec que surgieron en esta fase (FR-031, FR-032 y la ampliación de FR-016) se aprobaron e incorporaron el 2026-10-26 (ver "Cambios a la spec aprobados en la fase Plan").

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principio | Cómo lo cumple el plan | Estado |
|---|---|---|---|
| I | La spec manda | El plan deriva de la spec con IDs congelados. Lo que el diseño descubrió (NFC, tope en bytes, intentos durante la espera) se llevó **primero a la spec** (FR-031, FR-032, FR-016 → CA-001-28…30) y después al plan. | ✅ |
| II | Trazabilidad | Rama `001-autenticacion`. Tabla FR → componente (abajo). Cada CA se mapea a tests nombrados con su ID (`it('CA-001-14: …')`). Los PR listarán los CA cubiertos. | ✅ |
| III | Tests como contrato | Cada CA-001-01…30 tiene al menos un test (Supertest o RNTL). Umbral del 80 % en `src/**/*.service.ts` aplicado en Jest/CI. Las pantallas Registro, Ingreso e Inicio con RNTL. Maestro para registro, ingreso y cierre de sesión. CI bloqueante. | ✅ (ver nota Maestro) |
| IV | Aislamiento por usuario | `AccessTokenGuard` global con deny by default. `@CurrentUser()` saca el `userId` **solo** del token validado. `ValidationPipe` con `forbidNonWhitelisted` rechaza cualquier `userId` enviado por la app. Helper `findOwnedOrThrow()` devuelve 404 si el recurso no es del usuario (lo usarán las features 002+). | ✅ |
| V | Seguridad por defecto | argon2id. Acceso de vida corta + renovación revocable rotada. Límites en los endpoints de auth. DTOs con whitelist. Secretos solo en env (`.env.example`). Logs sin cuerpos ni cabeceras. `expo-secure-store`. `npm audit --audit-level=high` en CI. HTTPS lo pone Cloudflare. | ✅ |
| VI | Dinero y fechas | No hay montos en esta feature. Marcas de tiempo `timestamptz` en UTC, generadas con el `Clock` inyectado. | ✅ (N/A dinero) |
| VII | Simplicidad | Solo lo que pide la spec. Sin paquete compartido, sin Redis, sin Passport, sin `@nestjs/config`, sin librería de formularios. Abstracciones extra justificadas: `Clock` (exigido por las restricciones), `OriginResolver` (Cloudflare), `findOwnedOrThrow` (FR-024). Formato de error único con filtro global. Migraciones Prisma versionadas. | ✅ |
| VIII | Staging antes que producción | Esta feature no despliega. El CI corre en `develop` y `main`. Variables por ambiente en `.env.example`. El compose de staging y producción y el Dockerfile quedan para la fase Deploy. | ✅ (N/A en Plan) |

**Nota Maestro (III, "DEBERÍA")**: los flujos de Maestro se escriben en esta feature y se ejecutan en local durante Verify. No entran al CI todavía porque los emuladores en GitHub Actions son lentos e inestables y Maestro Cloud es de pago (restricción). La desviación aplica solo a la ejecución en CI, no a la existencia del test.

**Resultado del gate**: ✅ pasa antes de Phase 0. **Re-evaluado tras Phase 1**: ✅ pasa. El diseño de datos y los contratos no introducen violaciones. "Complexity Tracking" queda vacío.

## Cambios a la spec aprobados en la fase Plan (2026-10-26)

Ya incorporados en [spec.md](./spec.md) y registrados en [checklists/requirements.md](./checklists/requirements.md). Detalle técnico en [research.md § R3](./research.md) y § R5.

1. **FR-031 / CA-001-28**: la contraseña se normaliza a NFC al registrar y al ingresar, antes de medirla, compararla y hashearla. FR-011 aclara que eso no es "alterarla".
2. **FR-007**: la longitud (12–128) se cuenta en **puntos de código Unicode tras NFC**.
3. **FR-032 / CA-001-29**: tope de 1 024 bytes sobre la entrada cruda. En el registro → `PASSWORD_TOO_LONG`; en el ingreso → fallo genérico que cuenta para los límites.
4. **FR-016 / CA-001-30**: los intentos rechazados durante una espera no cuentan como fallos (ni por email ni por origen) ni prolongan la espera.

### 📌 Nota de implementación: medir la longitud de la contraseña

> La longitud se mide **siempre** con `[...password.normalize('NFC')].length`, en la app y en la API. **Nunca** con `password.length`.

- `.length` cuenta **unidades UTF-16**: un carácter fuera del BMP (😀 U+1F600) vale 2, así que con `.length` una contraseña válida de 128 caracteres con un emoji se rechazaría y una de 11 caracteres con 6 emojis se aceptaría.
- `[...str]` itera por **puntos de código**, y `normalize('NFC')` va antes para que `á` compuesta y descompuesta midan lo mismo.
- Cada proyecto encapsula la expresión en una única función `passwordLength()` (`apps/api/src/auth/password-policy/password-length.ts` y `apps/mobile/src/auth/password-length.ts`). La revisión de código rechaza cualquier `password.length` en validaciones.
- **Datos de prueba de CA-001-05**: deben incluir al menos un **emoji fuera del BMP**. El caso de 128 caracteres es `'😀' + 127 letras` (128 puntos de código, 129 unidades UTF-16) y **debe aceptarse**. La tabla completa de casos de borde (incluido CA-001-04 con 6 emojis y CA-001-29) está en [research.md § R3](./research.md).
- Verificar en la tarea de esqueleto que Hermes implementa `String.prototype.normalize`. RNTL corre en Node y no lo detectaría; se comprueba con el flujo de Maestro de registro.

## Tabla de trazabilidad FR → componente / endpoint

Rutas relativas a `apps/api/src/` (API) y `apps/mobile/` (app).

| FR | Resumen | Endpoint | Componente(s) | CA / verificación |
|---|---|---|---|---|
| FR-001 | Registro crea cuenta y abre sesión | `POST /v1/auth/register` | `auth/auth.controller.ts` → `auth/auth.service.ts#register` (transacción User + Session + RefreshToken) · app `app/(auth)/register.tsx` | CA-001-01 |
| FR-002 | Normalizar email (trim + minúsculas) | register, login | `auth/email.ts#normalizeEmail` (en el DTO con `@Transform`) | CA-001-03, CA-001-11 |
| FR-003 | Email vacío o inválido → rechazo | `POST /v1/auth/register` | `auth/dto/register.dto.ts` (regex + máx. 254) → `INVALID_EMAIL` · app valida formato antes de enviar | CA-001-02 |
| FR-004 | Email existente → no crear ni modificar | `POST /v1/auth/register` | `auth.service#register`: índice único `user.email` + captura de `P2002` | CA-001-03 |
| FR-005 | Mensaje explícito "email ya tiene cuenta" | `POST /v1/auth/register` | `409 EMAIL_ALREADY_REGISTERED` · app muestra el mensaje y enlace a Ingreso | CA-001-03 |
| FR-006 | < 12 caracteres → rechazo | `POST /v1/auth/register` | `auth/password-policy/password-policy.service.ts` + `password-length.ts` (`[...pw.normalize('NFC')].length`) → `PASSWORD_TOO_SHORT` · app prevalida con `src/auth/password-length.ts` | CA-001-04 |
| FR-007 | Aceptar 12–128 puntos de código tras NFC | `POST /v1/auth/register` | `password-policy.service` + `password-length.ts` (API y app). Datos de prueba con emoji fuera del BMP | CA-001-05 |
| FR-008 | > 128 caracteres → rechazo | `POST /v1/auth/register` | `password-policy.service` + `password-length.ts` → `PASSWORD_TOO_LONG` · app prevalida | CA-001-08 |
| FR-009 | Común, filtrada o igual al email → rechazo | `POST /v1/auth/register` | `password-policy.service` + `password-policy/blocklist.ts` (`Set` cargado de `data/blocklist.txt`; se compara la contraseña en NFC y en minúsculas) → `PASSWORD_TOO_COMMON` / `PASSWORD_TOO_PREDICTABLE` | CA-001-07, CA-001-21, SC-003 |
| FR-010 | Sin reglas de composición | — | `password-policy.service` (no hay regla de composición; un test lo fija) | CA-001-06 |
| FR-011 | Cualquier carácter imprimible, sin alterar (salvo NFC) | register, login | DTO sin `@Transform` en `password` (sin trim); argon2id sobre la forma NFC (FR-031) | CA-001-09 |
| FR-012 | Pegar y usar gestores de contraseñas | — | app `src/components/PasswordField.tsx` (`secureTextEntry`, `autoComplete="new-password"`/`"current-password"`, `textContentType`, sin bloquear pegado) | Revisión + test RNTL de props |
| FR-013 | Registros simultáneos → máximo una cuenta | `POST /v1/auth/register` | Índice único en `user.email` (Postgres) + `P2002` → 409 | CA-001-22 |
| FR-014 | Credenciales correctas → abrir sesión | `POST /v1/auth/login` | `auth.service#login` → `sessions/session.service.ts#open` · app `app/(auth)/login.tsx` | CA-001-10, CA-001-11 |
| FR-015 | Mismo mensaje y tiempo para email inexistente o contraseña incorrecta | `POST /v1/auth/login` | `auth.service#login` con **hash ficticio** + mismo camino de contadores → `401 INVALID_CREDENTIALS` | CA-001-12, CA-001-13, SC-006 |
| FR-016 | Límite por email (exponencial) y por origen (20/15 min); los intentos rechazados durante una espera no cuentan | `POST /v1/auth/login` | `throttling/login-throttle.service.ts` (tablas `login_email_throttle`, `rate_limit_event`, `login_origin_lock`; advisory lock; `assertAllowed` corta **antes** de `recordFailure`) · `common/origin/origin-resolver.ts` | CA-001-14, CA-001-26, CA-001-27, CA-001-30 |
| FR-017 | Rechazar durante la espera, aun con contraseña correcta; informar cuándo | `POST /v1/auth/login` | `login-throttle.service#assertAllowed` → `429 LOGIN_TEMPORARILY_BLOCKED` + `retryAt` + `Retry-After` · app muestra la hora | CA-001-14, CA-001-27 |
| FR-018 | Reiniciar contador con éxito o tras 1 h sin fallos | `POST /v1/auth/login` | `login-throttle.service#recordSuccess` / reinicio perezoso si `lastFailureAt + 1 h ≤ now` | CA-001-26 (incluye el caso de 59 min) |
| FR-019 | 5 registros por origen por hora | `POST /v1/auth/register` | `throttling/register-rate-limit.guard.ts` (antes del `ValidationPipe`) → `429 REGISTER_RATE_LIMITED` | CA-001-23 |
| FR-020 | Sesión vigente sobrevive a cerrar/reabrir la app | `POST /v1/auth/refresh`, `GET /v1/me` | app `src/auth/token-storage.ts` (secure store) + `src/auth/session-store.ts` (hidrata al arrancar) + `app/_layout.tsx` (redirección) | CA-001-16 |
| FR-021 | Expira tras 30 días sin uso; cada uso renueva | todos los autenticados + refresh | `sessions/access-token.guard.ts` y `session.service#touch` (`lastUsedAt`, `expiresAt = now + 30 d` con `Clock`) | CA-001-16, CA-001-17 |
| FR-022 | Sesión expirada o cerrada → rechazo y pantalla de ingreso | todos los autenticados | `access-token.guard` → `401 SESSION_INVALID` · app `src/api/client.ts` (un refresh single-flight; si falla → `sessionStore.signOut()` → redirige a `(auth)/login`) | CA-001-17, CA-001-19 |
| FR-023 | Cuenta determinada solo por la sesión | `GET /v1/me` (y features futuras) | `common/auth/current-user.decorator.ts` + guard global + `ValidationPipe({ whitelist, forbidNonWhitelisted })` | CA-001-15 |
| FR-024 | Dato de otra cuenta → como si no existiera | (features futuras) | `common/ownership/find-owned-or-throw.ts` → `404 NOT_FOUND`. En 001 se prueba con un módulo de prueba (`test/fixtures/owned-probe.module.ts`) que solo se monta en e2e | CA-001-15, SC-004 |
| FR-025 | Sin sesión válida → sin datos | todos los autenticados | `access-token.guard` global (deny by default, `@Public()` explícito) | CA-001-19 |
| FR-026 | Cerrar sesión invalida, borra del teléfono y muestra ingreso | `POST /v1/auth/logout` | `session.service#revoke` (`LOGOUT`) · app `app/(app)/index.tsx` + `session-store#signOut` (borra secure store; si no hay red, guarda el token pendiente y lo reintenta al volver) | CA-001-18, CA-001-19, SC-005 |
| FR-027 | Logout solo afecta a este dispositivo | `POST /v1/auth/logout` | `session.service#revoke` por `sessionId` del token de renovación (no por `userId`) | CA-001-20 |
| FR-028 | No guardar ni mostrar la contraseña legible | — | `auth/password-hasher.ts` (argon2id, PHC) · ningún DTO de respuesta incluye la contraseña ni el hash | CA-001-24 |
| FR-029 | Logs sin contraseñas ni credenciales | — | `common/logging/request-logging.interceptor.ts` (sin cuerpo ni cabeceras) · log de Prisma `warn`/`error` · `common/errors/http-exception.filter.ts` | CA-001-25 |
| FR-030 | Credenciales solo en almacenamiento seguro | — | app `src/auth/token-storage.ts` (`expo-secure-store`; regla de lint que prohíbe `AsyncStorage`) | Revisión |
| FR-031 | Normalizar la contraseña a NFC al registrar e ingresar | register, login | `auth/password-policy/normalize-password.ts` (`pw.normalize('NFC')`), llamado por `auth.service#register` y `#login` antes de la política, la lista, la comparación con el email y argon2id · app normaliza antes de medir | CA-001-28 |
| FR-032 | > 1 024 bytes → rechazo sin procesar | register, login | `auth.service`: `Buffer.byteLength(pw, 'utf8') > 1024` sobre la entrada cruda, antes de NFC. Registro → `400 PASSWORD_TOO_LONG`; ingreso → `login-throttle.service#recordFailure` + `401 INVALID_CREDENTIALS` sin argon2 | CA-001-29 |

### Endpoints (resumen; contrato completo en [contracts/openapi.yaml](./contracts/openapi.yaml))

| Método y ruta | Acceso | Éxito | Errores principales |
|---|---|---|---|
| `POST /v1/auth/register` | Público + límite por origen | `201` tokens + usuario | `400 INVALID_EMAIL`, `PASSWORD_TOO_SHORT`, `PASSWORD_TOO_LONG`, `PASSWORD_TOO_COMMON`, `PASSWORD_TOO_PREDICTABLE`, `VALIDATION_ERROR` · `409 EMAIL_ALREADY_REGISTERED` · `429 REGISTER_RATE_LIMITED` |
| `POST /v1/auth/login` | Público + límites | `200` tokens + usuario | `401 INVALID_CREDENTIALS` · `429 LOGIN_TEMPORARILY_BLOCKED` |
| `POST /v1/auth/refresh` | Token de renovación | `200` tokens nuevos | `401 SESSION_INVALID` |
| `POST /v1/auth/logout` | Token de renovación | `204` (idempotente) | — |
| `GET /v1/me` | Token de acceso | `200` `{ id, email, createdAt }` | `401 SESSION_INVALID` |
| `GET /health` | Público | `200` | — |

Todas las respuestas de error usan el mismo formato (principio VII):

```json
{ "error": { "code": "…", "message": "…en español…", "retryAt": "ISO-8601 opcional", "fields": ["opcional"] } }
```

### Flujo de `POST /v1/auth/register`

1. `RegisterRateLimitGuard` (antes del `ValidationPipe`): si hay ≥ 5 `REGISTER_REQUEST` del origen en la última hora → `429`. Si no, registra el evento.
2. `ValidationPipe` + DTO (email normalizado y con formato válido).
3. Contraseña: tope de 1 024 bytes → NFC → longitud → lista → igual al email (R3).
4. Hash argon2id de la forma NFC.
5. Transacción: `INSERT` de User + Session + RefreshToken **sin consultar antes si el email existe**. Si Prisma lanza **`P2002`** (índice único de `user.email`), `auth.service#register` lo traduce a **`409 EMAIL_ALREADY_REGISTERED`** y la transacción se revierte. Así dos registros simultáneos dan exactamente una cuenta y un 409 (FR-004, FR-013, CA-001-22). Cualquier otro error de Prisma sigue como `500 INTERNAL_ERROR`.
6. `201` con los tokens y el usuario.

### Flujo de `POST /v1/auth/login`

1. `OriginResolver` obtiene `originKey`. Si `login_origin_lock.lockedUntil > now` → `429` (sin tocar el email).
2. Se normaliza el email (`emailKey = HMAC`) y se abre una transacción con `pg_advisory_xact_lock(emailKey)`.
3. Contador perezoso: si `lastFailureAt + 1 h ≤ now`, se trata como 0. Si `lockedUntil > now` → `429` con `retryAt`. **No suma fallo** ni por email ni por origen, ni cambia `lockedUntil` (FR-016, CA-001-30).
4. Si la contraseña cruda supera 1 024 bytes (FR-032), salta directo al paso 5 (fallo) sin argon2.
5. Se normaliza la contraseña a NFC (FR-031) y se busca el usuario. **Si el email no existe, se verifica igualmente con argon2id contra un hash ficticio** (mismos parámetros, generado al arrancar, nunca guardado), para que email inexistente y contraseña incorrecta tarden lo mismo (FR-015, SC-006). Si existe, se verifica contra su hash real.
6. **Fallo**: `consecutiveFailures++`; si es ≥ 5, `lockedUntil = now + min(15, 2^(n−5)) min`. Se inserta un `LOGIN_FAILURE` por origen; si hay ≥ 20 en 15 min → upsert `login_origin_lock = now + 15 min`. Respuesta `401 INVALID_CREDENTIALS`.
7. **Éxito**: se borra la fila del email, se abre la `Session` y se emiten los tokens. Respuesta `200`.

## Project Structure

### Documentation (this feature)

```text
specs/001-autenticacion/
├── spec.md
├── plan.md              # este archivo
├── research.md          # Phase 0
├── data-model.md        # Phase 1
├── quickstart.md        # Phase 1
├── contracts/
│   └── openapi.yaml     # Phase 1
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks; NO lo crea este comando)
```

### Source Code (repository root)

```text
package.json                     # npm workspaces: apps/*; scripts raíz (lint, test, typecheck)
tsconfig.base.json               # strict: true compartido
.nvmrc                           # Node 24
.github/workflows/ci.yml         # jobs api, mobile, audit
deploy/local/compose.yml         # PostgreSQL 18 para desarrollo y tests locales
.env.example                     # plantilla raíz (sin secretos reales)

apps/api/
├── package.json
├── .env.example                 # DATABASE_URL, JWT_ACCESS_SECRET, THROTTLE_HMAC_SECRET, TRUST_CF_CONNECTING_IP, PORT
├── prisma/
│   ├── schema.prisma
│   └── migrations/              # solo migraciones versionadas
├── scripts/
│   └── build-password-blocklist.ts
├── src/
│   ├── main.ts                  # ValidationPipe global (whitelist), filtro de errores, interceptor de logs
│   ├── app.module.ts
│   ├── config/load-config.ts    # valida process.env al arrancar
│   ├── prisma/prisma.service.ts
│   ├── common/
│   │   ├── clock/               # Clock, CLOCK, SystemClock
│   │   ├── origin/              # OriginResolver (CF-Connecting-IP, IPv6 /64, HMAC)
│   │   ├── errors/              # AppError, códigos, http-exception.filter.ts
│   │   ├── logging/             # request-logging.interceptor.ts
│   │   ├── auth/                # @Public(), @CurrentUser()
│   │   └── ownership/           # findOwnedOrThrow()
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth.controller.ts   # register, login, refresh, logout
│   │   ├── auth.service.ts
│   │   ├── email.ts             # normalizeEmail, isValidEmail
│   │   ├── password-hasher.ts   # argon2id + hash ficticio
│   │   ├── dto/
│   │   └── password-policy/
│   │       ├── password-policy.service.ts
│   │       ├── normalize-password.ts   # NFC (FR-031)
│   │       ├── password-length.ts      # [...pw.normalize('NFC')].length; nunca .length
│   │       ├── blocklist.ts
│   │       └── data/            # blocklist.txt (generado) + common-passwords.es.txt (curado)
│   ├── sessions/
│   │   ├── sessions.module.ts
│   │   ├── session.service.ts   # open, touch, rotate (reutilización + gracia), revoke
│   │   ├── token.service.ts     # firmar/verificar JWT con Clock; generar/hashear refresh
│   │   └── access-token.guard.ts
│   ├── throttling/
│   │   ├── throttling.module.ts
│   │   ├── login-throttle.service.ts
│   │   └── register-rate-limit.guard.ts
│   └── users/
│       ├── users.module.ts
│       └── me.controller.ts     # GET /v1/me
└── test/
    ├── support/                 # FakeClock, app factory, truncado de BD, captura de logs
    ├── fixtures/owned-probe.module.ts   # solo e2e, para CA-001-15
    └── e2e/                     # register.e2e-spec.ts, login.e2e-spec.ts, session.e2e-spec.ts, logs.e2e-spec.ts
    # tests unitarios junto al código: src/**/*.spec.ts

apps/mobile/
├── package.json
├── app.json
├── app/
│   ├── _layout.tsx              # QueryClientProvider + redirección por estado de sesión
│   ├── (auth)/login.tsx
│   ├── (auth)/register.tsx
│   └── (app)/index.tsx          # inicio: email de la cuenta + "Cerrar sesión"
├── src/
│   ├── api/client.ts            # fetch + Bearer + refresh single-flight + mapeo de errores
│   ├── api/auth.ts              # register, login, logout, refresh, me
│   ├── auth/session-store.ts    # Zustand: 'loading' | 'signedOut' | 'signedIn'
│   ├── auth/token-storage.ts    # expo-secure-store (+ token de logout pendiente)
│   ├── auth/password-length.ts  # misma medida que la API (prevalidación)
│   ├── components/PasswordField.tsx
│   └── i18n/messages.ts         # mensajes en español por código de error
├── __tests__/                   # RNTL: register.test.tsx, login.test.tsx, home.test.tsx
└── .maestro/                    # registro.yaml, ingreso.yaml, cerrar-sesion.yaml
```

**Structure Decision**: monorepo con npm workspaces y dos aplicaciones, `apps/api` (NestJS, módulos por dominio) y `apps/mobile` (Expo Router), siguiendo la estructura objetivo de `CLAUDE.md`. `deploy/` solo tiene `local/compose.yml` en esta feature; staging y producción llegan en la fase Deploy.

### Orden sugerido para `/speckit-tasks` (orientativo; no son tareas)

1. Esqueleto del monorepo + CI mínimo (con confirmación para instalar dependencias).
2. Base de la API: config, Prisma + primera migración (con confirmación), `Clock`, formato de error, logging.
3. US1 Registro (P1) → US2 Ingreso + límites (P2) → US3 Cierre de sesión (P3); cada historia con sus tests de API y de app.
4. Flujos Maestro y verificación de SC-001/SC-002.

## Complexity Tracking

> Sin violaciones del Constitution Check. No aplica.
