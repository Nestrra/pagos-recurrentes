---
description: "Lista de tareas de la feature 001-autenticacion"
---

# Tasks: Autenticación (registro, ingreso y cierre de sesión)

**Input**: Design documents from `specs/001-autenticacion/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/openapi.yaml](./contracts/openapi.yaml), [quickstart.md](./quickstart.md)

**Tests**: obligatorios (constitución III y pedido explícito). Cada CA tiene **al menos una tarea de test** cuyo nombre lleva su ID y que va **antes** de la implementación que lo cumple. Dentro de cada archivo, cada caso se llama `it('CA-001-XX: …')`. FR-012 y FR-030 ("verificado por revisión") llevan una **tarea de revisión**, no de test.

**Organization**: por historia de usuario (US1 Registro P1, US2 Ingreso P2, US3 Cierre de sesión P3), precedidas por Setup (esqueleto + CI) y Foundational.

## Format: `[ID] [P?] [Story] Descripción con ruta`

- **[P]**: se puede hacer en paralelo (archivos distintos, sin dependencias pendientes).
- **[USn]**: historia de usuario a la que pertenece.
- **Cubre**: FR/CA/SC que la tarea cumple o verifica.
- **📦 Dependencias**: paquetes que la tarea instala. **Siempre con confirmación previa** (CLAUDE.md). La versión exacta se fija en `package.json` (sin `^`) con la estable vigente al instalar.
- **⚠ Confirmación**: la tarea crea o aplica una migración, descarga algo de fuera o modifica `CLAUDE.md`.

## Convenciones para todas las tareas

- **Tamaño**: cada tarea es un diff revisable (orientativo ≤ 300 líneas). Se implementa **una tarea a la vez** y se muestra el diff antes de seguir.
- **Rutas**: relativas a la raíz del repo.
- **Commits**: `tipo(001): descripción en español`. Los tests se commitean en rojo antes de la implementación (TDD).
- **Comentarios** en español y abundantes; identificadores en inglés.
- **Tests e2e** (Supertest, Postgres de test):
  - Usan `createTestApp()` con `FakeClock` y `TRUST_CF_CONNECTING_IP=true`.
  - Cada prueba envía un `CF-Connecting-IP` propio (`uniqueOrigin()`), para que el límite de registro (FR-019) no interfiera entre pruebas, salvo cuando la prueba trata justamente de límites por origen.
  - La BD se limpia con `resetDb()` en `beforeEach`.
- **Contraseñas de prueba**: no deben estar en la lista prohibida. Se usan frases no triviales (por ejemplo `'mi gato duerme en la nevera'`) y, para longitudes exactas, los casos de [research.md § R3](./research.md).
- **Longitud de contraseña**: siempre `[...password.normalize('NFC')].length`, **nunca** `.length` (nota de implementación del plan).

---

## Phase 1: Setup — esqueleto del monorepo y CI mínimo

**Purpose**: monorepo npm workspaces con `apps/api` (NestJS) y `apps/mobile` (Expo Router), lint, tests de humo y CI bloqueante. Sin código de producto.

- [ ] T001 Crear la raíz del monorepo: `package.json` (`"private": true`, `"workspaces": ["apps/*"]`, `"engines": { "node": ">=24" }`, scripts `lint`, `typecheck` y `test` que delegan con `--workspaces --if-present`), `.nvmrc` (`24`), `tsconfig.base.json` (`"strict": true`, `"noUncheckedIndexedAccess": true`, `"noImplicitOverride": true`), `.editorconfig` y `.prettierrc.json`
  - Cubre: principio VII, TypeScript estricto (CLAUDE.md)
  - 📦 Dependencias (dev, raíz): `typescript`, `prettier`
- [ ] T002 Configurar ESLint en `eslint.config.mjs` (flat config con `typescript-eslint` y `eslint-config-prettier`):
  - Regla `no-restricted-imports` que prohíbe `@react-native-async-storage/async-storage`, con el mensaje "Usa expo-secure-store (FR-030)".
  - Regla `no-restricted-syntax` que prohíbe `MemberExpression` con `object.name=/password/i` y `property.name='length'`, con el mensaje "Usa passwordLength() (FR-007, R3)".
  - Cubre: FR-007, FR-030 (prevención)
  - 📦 Dependencias (dev, raíz): `eslint`, `@eslint/js`, `typescript-eslint`, `eslint-config-prettier`
- [ ] T003 Crear el esqueleto de la API NestJS a mano (sin `nest new`, para que el diff sea pequeño):
  - `apps/api/package.json` con los scripts `build`, `start:dev`, `lint`, `typecheck`, `test` (unit), `test:e2e` y `test:cov` (unit + e2e con cobertura).
  - `apps/api/tsconfig.json` (extiende `../../tsconfig.base.json`, `emitDecoratorMetadata`, `experimentalDecorators`), `apps/api/tsconfig.build.json` y `apps/api/nest-cli.json`.
  - `apps/api/src/main.ts`, `apps/api/src/app.module.ts` y `apps/api/src/health/health.controller.ts` (`GET /health` → `{ "status": "ok" }`).
  - Cubre: contrato `GET /health`
  - 📦 Dependencias (api): `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express`, `reflect-metadata`, `rxjs`; dev: `@nestjs/cli`, `@types/node`, `@types/express`
- [ ] T004 Configurar Jest en la API:
  - `apps/api/jest.config.ts` con dos `projects`: `unit` (`src/**/*.spec.ts`) y `e2e` (`test/**/*.e2e-spec.ts`, con `setupFiles: ['<rootDir>/test/jest-e2e.setup.ts']`).
  - `coverageThreshold` del 80 % para `src/**/*.service.ts`, medido sobre unit + e2e.
  - Test de humo en `apps/api/test/e2e/health.e2e-spec.ts`.
  - Cubre: principio III (umbral de cobertura)
  - 📦 Dependencias (dev, api): `jest`, `ts-jest`, `@types/jest`, `@nestjs/testing`, `supertest`, `@types/supertest`
- [ ] T005 PostgreSQL local e inicialización de Prisma:
  - `deploy/local/compose.yml` (imagen `postgres:18`, puerto 5432, volumen nombrado) y `deploy/local/init/01-create-test-db.sql` (crea la BD `pagos_test`).
  - `apps/api/.env.example` con `DATABASE_URL` y `DATABASE_URL_TEST`, solo con valores de ejemplo.
  - `apps/api/prisma/schema.prisma` (generator + datasource, sin modelos).
  - `apps/api/src/prisma/prisma.service.ts` y `apps/api/src/prisma/prisma.module.ts` con `log: ['warn', 'error']` (nunca `query`).
  - Scripts `prisma:generate`, `prisma:migrate:dev` y `prisma:migrate:deploy` en `apps/api/package.json`.
  - Cubre: FR-029 (log de Prisma sin parámetros), principio VII
  - 📦 Dependencias (api): `@prisma/client`; dev: `prisma`
- [ ] T006 [P] Crear el esqueleto de la app Expo Router:
  - `apps/mobile/package.json`, `apps/mobile/app.json` (nombre, `scheme`, `extra.apiUrl`), `apps/mobile/tsconfig.json` (extiende `expo/tsconfig.base` con `"strict": true`) y `apps/mobile/metro.config.js` solo si el monorepo lo requiere.
  - `apps/mobile/app/_layout.tsx` (`Stack`) y `apps/mobile/app/index.tsx` (pantalla placeholder).
  - Cubre: estructura del plan
  - 📦 Dependencias (mobile, con `npx expo install`): `expo`, `expo-router`, `react`, `react-native`, `react-native-safe-area-context`, `react-native-screens`, `expo-linking`, `expo-constants`, `expo-status-bar`; dev: `@types/react`
- [ ] T007 Configurar Jest en la app: preset `jest-expo` en `apps/mobile/package.json` (o `apps/mobile/jest.config.js`) y test de humo en `apps/mobile/__tests__/smoke.test.tsx` que renderiza `app/index.tsx` con RNTL
  - Cubre: principio III (base de RNTL)
  - 📦 Dependencias (dev, mobile): `jest`, `jest-expo`, `@testing-library/react-native`, `@types/jest`
- [ ] T008 Verificar en emulador o dispositivo que Hermes implementa `String.prototype.normalize('NFC')`:
  - Se usa un snippet temporal en `apps/mobile/app/index.tsx`: `'canción'.normalize('NFC') === 'canción'`. El snippet se elimina antes del commit.
  - El resultado (versión de Expo/Hermes, iOS/Android) se anota en `specs/001-autenticacion/research.md` § R3, "Verificar en el esqueleto". Si falla, se anota que la app omite la prevalidación de longitud.
  - Cubre: FR-031 (riesgo de R3)
- [ ] T009 Crear el CI mínimo en `.github/workflows/ci.yml`. Se dispara en `pull_request` y en `push` a `develop` y `main`, con Node desde `.nvmrc` y caché de npm. Tres jobs:
  1. `api`: servicio `postgres:18` → `npm ci` → `npm run lint -w apps/api` → `typecheck` → `prisma:migrate:deploy` contra la BD de test → `test:cov`. Falla si la cobertura de `*.service.ts` es menor al 80 %.
  2. `mobile`: `lint` → `typecheck` → `test`.
  3. `audit`: `npm audit --audit-level=high`.
  - Los secretos de test son valores ficticios declarados en el workflow (no secretos reales).
  - Crear `.github/pull_request_template.md` con los campos obligatorios **Spec** (enlace a `specs/NNN-nombre/spec.md`), **CA cubiertos** (lista de `CA-NNN-YY`), **Tareas** (`T###`) y una casilla "CI en verde".
  - Cubre: principios II (cada PR enlaza su spec y lista sus CA), III ("ningún PR con CI en rojo") y V (dependencias vulnerables)
- [ ] T010 Documentar los comandos reales en la sección "Comandos" y actualizar "Estado actual" de `CLAUDE.md`. Ajustar `specs/001-autenticacion/quickstart.md` si algún nombre de script cambió
  - ⚠ Confirmación: modifica `CLAUDE.md`
  - Cubre: CLAUDE.md ("se documentan al crear el esqueleto")

**Checkpoint**: `npm ci`, `npm run lint`, `npm run typecheck` y `npm test` en verde en local y en CI; `GET /health` responde.

---

## Phase 2: Foundational (prerrequisitos bloqueantes)

**Purpose**: reloj inyectable, configuración, formato de error único, logging sin credenciales, origen, decoradores, soporte de tests e2e y base de la app.

**⚠️ CRITICAL**: ninguna historia empieza antes de terminar esta fase.

- [ ] T011 [P] Reloj inyectable:
  - `apps/api/src/common/clock/clock.ts` (interfaz `Clock { now(): Date }`, token `CLOCK`, `SystemClock`) y `apps/api/src/common/clock/clock.module.ts` (global).
  - `apps/api/test/support/fake-clock.ts` (`set(date)`, `advance(ms)`).
  - Test unitario en `apps/api/src/common/clock/clock.spec.ts`.
  - Cubre: restricción "tiempo inyectable" (R7)
- [ ] T012 [P] Configuración validada:
  - `apps/api/src/config/load-config.ts` valida `process.env` al arrancar y falla rápido con un mensaje claro: `DATABASE_URL` (requerido), `JWT_ACCESS_SECRET` (≥ 32 caracteres), `THROTTLE_HMAC_SECRET` (≥ 32 caracteres), `TRUST_CF_CONNECTING_IP` (`true`/`false`, por defecto `false`) y `PORT` (por defecto `3000`). Nunca imprime los valores.
  - Añadir las variables a `apps/api/.env.example`.
  - Test unitario en `apps/api/src/config/load-config.spec.ts`.
  - Cubre: principio V (secretos solo en env)
- [ ] T013 Formato de error único y validación global:
  - `apps/api/src/common/errors/error-codes.ts`: enum idéntico a `ErrorCode` de `contracts/openapi.yaml`.
  - `apps/api/src/common/errors/app-error.ts`.
  - `apps/api/src/common/errors/http-exception.filter.ts`:
    - Responde siempre `{ error: { code, message, retryAt?, fields? } }`.
    - Si hay `retryAt`, añade la cabecera `Retry-After` en segundos.
    - Cualquier error no controlado → `500 INTERNAL_ERROR`, registrando solo `code`, `error.name` y el stack **sin** el `message` de errores de terceros (Prisma, argon2, JWT), que puede incluir valores de entrada (FR-029).
  - `apps/api/src/app.setup.ts` con `configureApp(app)`:
    - Aplica `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })`, con `exceptionFactory` → `400 VALIDATION_ERROR` + `fields`, y el filtro global.
    - `main.ts` lo usa.
  - Test unitario en `apps/api/src/common/errors/http-exception.filter.spec.ts`.
  - Cubre: FR-023 (rechazo de campos no declarados), principio VII (formato único), principio V (whitelist)
  - 📦 Dependencias (api): `class-validator`, `class-transformer`
- [ ] T014 Test **CA-001-25** (parte unitaria) en `apps/api/src/common/logging/request-logging.interceptor.spec.ts`: `it('CA-001-25: el registro de una solicitud solo incluye método, ruta plantilla, estado, duración y requestId; nunca cuerpo, cabeceras ni query')`. Se envía una solicitud con `password`, `refreshToken` y `Authorization` y se comprueba que no aparecen en la salida
  - Cubre: CA-001-25, FR-029
- [ ] T015 Implementar `apps/api/src/common/logging/request-logging.interceptor.ts` (Logger de NestJS; ruta plantilla de Express `req.route.path`; `requestId` aleatorio) y registrarlo en `configureApp()` de `apps/api/src/app.setup.ts` (depende de T013 y T014)
  - Cubre: FR-029 (hace pasar T014)
- [ ] T016 [P] `OriginResolver` en `apps/api/src/common/origin/origin-resolver.ts` (+ módulo):
  - Con `TRUST_CF_CONNECTING_IP=true` usa `CF-Connecting-IP` si es una IP válida; si no, `req.socket.remoteAddress`.
  - Si la flag está activa y la cabecera falta, registra un aviso **sin** la IP.
  - Normaliza las IPv4 mapeadas en IPv6 a IPv4 y agrupa las IPv6 por prefijo **/64**.
  - Devuelve `originKey = HMAC-SHA256(THROTTLE_HMAC_SECRET, ipNormalizada)` en hex (64 caracteres).
  - Tests unitarios en `apps/api/src/common/origin/origin-resolver.spec.ts`.
  - Cubre: FR-016 y FR-019 (base), R6
- [ ] T017 [P] Decoradores `apps/api/src/common/auth/public.decorator.ts` (`@Public()`) y `apps/api/src/common/auth/current-user.decorator.ts` (`@CurrentUser()` → `AuthenticatedUser { userId: string; sessionId: string }`, leído **solo** de `req.user`, que asigna el guard)
  - Cubre: FR-023, principio IV
- [ ] T018 Soporte de tests e2e:
  - `apps/api/test/support/create-test-app.ts`: crea `AppModule`, sustituye `CLOCK` por `FakeClock`, llama a `configureApp()` y expone `{ app, clock, prisma }`.
  - `apps/api/test/support/reset-db.ts`: `TRUNCATE … RESTART IDENTITY CASCADE` sobre todas las tablas del esquema.
  - `apps/api/test/support/origins.ts`: `uniqueOrigin()` devuelve una IP de documentación distinta en cada llamada.
  - `apps/api/test/support/log-capture.ts`: captura la salida del Logger de NestJS.
  - `apps/api/test/jest-e2e.setup.ts`: `DATABASE_URL = DATABASE_URL_TEST`, `TRUST_CF_CONNECTING_IP=true` y secretos ficticios.
  - Cubre: base de todos los tests e2e
- [ ] T019 [P] Cliente HTTP de la app:
  - `apps/mobile/src/api/client.ts`: wrapper de `fetch` con base URL de `app.json` `extra.apiUrl`. Convierte `ErrorResponse` en `ApiError { code, message, retryAt?, fields? }` y un fallo de red en `NetworkError`.
  - `apps/mobile/src/api/types.ts`: tipos escritos a mano a partir de `contracts/openapi.yaml`.
  - `apps/mobile/src/i18n/messages.ts`: mensaje en español por `ErrorCode`, más "Sin conexión. Revisa tu internet e inténtalo de nuevo.".
  - Test en `apps/mobile/__tests__/api-client.test.ts`.
  - Cubre: Edge Case "Sin conexión", supuesto "mensajes en español"
- [ ] T020 Base de sesión en la app:
  - `apps/mobile/src/auth/token-storage.ts`: solo `expo-secure-store`, con las claves `accessToken`, `refreshToken` y `pendingLogoutToken`.
  - `apps/mobile/src/auth/session-store.ts`: Zustand con `status: 'loading' | 'signedOut' | 'signedIn'`, `user`, `signIn(session)` (guarda los tokens) y `signOutLocal()` (borra los tokens).
  - `apps/mobile/app/_layout.tsx`: `QueryClientProvider` y redirección entre los grupos `(auth)` y `(app)` según `status`.
  - Test en `apps/mobile/__tests__/session-store.test.ts` (con mock de secure store).
  - Cubre: FR-030, FR-020 (base)
  - 📦 Dependencias (mobile, con `npx expo install` donde aplique): `zustand`, `@tanstack/react-query`, `expo-secure-store`
- [ ] T021 [P] Componente `apps/mobile/src/components/PasswordField.tsx`: `secureTextEntry`, prop `autoComplete` (`'new-password'` o `'current-password'`), `textContentType` (`'newPassword'` o `'password'`), `autoCapitalize="none"`, `autoCorrect={false}`, sin `contextMenuHidden` (el pegado queda permitido). No recorta el valor
  - Cubre: FR-011 (app), FR-012 (se verifica en T082)

**Checkpoint**: la base está lista; las historias pueden empezar.

---

## Phase 3: User Story 1 — Crear una cuenta propia (Priority: P1) 🎯 MVP

**Goal**: una persona se registra con email y contraseña válidos y queda con sesión abierta en su propia cuenta. Cada regla de rechazo funciona.

**Independent Test**: `POST /v1/auth/register` con un email nuevo → `201` con tokens, y `GET /v1/me` con ese acceso devuelve la cuenta. Cada regla de rechazo (email inválido o repetido; contraseña corta, larga, común, igual al email, de más de 1 024 bytes; límite por origen) devuelve su código. En la app, el formulario de registro lleva a Inicio.

### Tests para User Story 1 (escribir primero; deben fallar)

- [ ] T022 [P] [US1] Test **CA-001-01** en `apps/api/test/e2e/ca-001-01.e2e-spec.ts`: registro de `ana@ejemplo.com` con una contraseña válida → `201` con `accessToken`, `refreshToken`, `accessTokenExpiresAt` y `user.email`. Con ese `accessToken`, `GET /v1/me` → `200` con la misma cuenta. Existe 1 `User` y 1 `Session`
  - Cubre: CA-001-01, FR-001
- [ ] T023 [P] [US1] Test **CA-001-02** en `apps/api/test/e2e/ca-001-02.e2e-spec.ts`: `'ana@'`, `'ana.ejemplo.com'`, `''` y un email de 255 caracteres → `400 INVALID_EMAIL` con `fields: ['email']`, y 0 filas en `user`
  - Cubre: CA-001-02, FR-003
- [ ] T024 [P] [US1] Test **CA-001-03** en `apps/api/test/e2e/ca-001-03.e2e-spec.ts`: con `ana@ejemplo.com` ya registrado, registrar `'ana@ejemplo.com'` y `' Ana@Ejemplo.com '` → `409 EMAIL_ALREADY_REGISTERED` con el mensaje exacto "Este email ya tiene una cuenta. Ingresa con él.". Sigue habiendo 1 `User` y su `passwordHash` no cambió
  - Cubre: CA-001-03, FR-002, FR-004, FR-005
- [ ] T025 [P] [US1] Test **CA-001-04** en `apps/api/test/e2e/ca-001-04.e2e-spec.ts`: 11 caracteres en letras y `'😀'.repeat(6) + 'abcde'` (11 puntos de código, 17 unidades UTF-16) → `400 PASSWORD_TOO_SHORT`, con un mensaje que menciona 12
  - Cubre: CA-001-04, FR-006, FR-007 (medida)
- [ ] T026 [P] [US1] Test **CA-001-05** en `apps/api/test/e2e/ca-001-05.e2e-spec.ts`: se aceptan (`201`) `'😀' + 'gatoazulsol'` (12), `'🦜'.repeat(4)` + 60 letras (64) y **`'😀'` + 127 letras (128 puntos de código, 129 UTF-16)**. Las letras de relleno salen de una frase no trivial
  - Cubre: CA-001-05, FR-007
- [ ] T027 [P] [US1] Test **CA-001-06** en `apps/api/test/e2e/ca-001-06.e2e-spec.ts`: una contraseña de 12+ minúsculas sin números ni símbolos (`'nubesdecolorlila'`) → `201`
  - Cubre: CA-001-06, FR-010
- [ ] T028 [P] [US1] Test **CA-001-07** en `apps/api/test/e2e/ca-001-07.e2e-spec.ts`:
  - `'contraseña123'`, `'Contraseña123'` y `'123456789012'` → `400 PASSWORD_TOO_COMMON`.
  - **SC-003**: una muestra de 50 entradas de `apps/api/src/auth/password-policy/data/blocklist.txt` (PRNG con semilla fija, cada una desde `uniqueOrigin()`) → 100 % rechazadas.
  - Cubre: CA-001-07, FR-009, SC-003
- [ ] T029 [P] [US1] Test **CA-001-08** en `apps/api/test/e2e/ca-001-08.e2e-spec.ts`: `'😀'` + 128 letras (129 puntos de código) → `400 PASSWORD_TOO_LONG`, con un mensaje que menciona 128
  - Cubre: CA-001-08, FR-008
- [ ] T030 [P] [US1] Test **CA-001-09 (registro)** en `apps/api/test/e2e/ca-001-09-registro.e2e-spec.ts`:
  - Se aceptan contraseñas con espacios al inicio y al final, tildes, `ñ`, emojis y `'👨‍👩‍👧'` + 7 letras.
  - El `passwordHash` guardado verifica (con `argon2.verify`) contra la contraseña exacta en NFC y **no** contra su versión recortada.
  - Cubre: CA-001-09 (parte de registro), FR-011
- [ ] T031 [P] [US1] Test **CA-001-21** en `apps/api/test/e2e/ca-001-21.e2e-spec.ts`: registro de `ana@ejemplo.com` con contraseña `'ana@ejemplo.com'` y con `' ANA@ejemplo.com '` → `400 PASSWORD_TOO_PREDICTABLE`, y 0 cuentas
  - Cubre: CA-001-21, FR-009
- [ ] T032 [P] [US1] Test **CA-001-22** en `apps/api/test/e2e/ca-001-22.e2e-spec.ts`: dos `POST /v1/auth/register` simultáneos (`Promise.all`) con el mismo email nuevo → exactamente un `201` y un `409 EMAIL_ALREADY_REGISTERED`; 1 `User` y 1 `Session`
  - Cubre: CA-001-22, FR-013
- [ ] T033 [P] [US1] Test **CA-001-23** en `apps/api/test/e2e/ca-001-23.e2e-spec.ts`:
  - Desde un mismo origen se envían 5 solicitudes de registro (mezcla de válidas e inválidas). La 6.ª → `429 REGISTER_RATE_LIMITED` con `retryAt` y `Retry-After`, sin crear cuenta.
  - Desde otro origen → `201`.
  - Con `clock.advance(1 h)` el primer origen vuelve a ser aceptado.
  - Cubre: CA-001-23, FR-019
- [ ] T034 [P] [US1] Test **CA-001-24** en `apps/api/test/e2e/ca-001-24.e2e-spec.ts`: tras registrar, `user.passwordHash` empieza por `$argon2id$v=19$m=19456,t=2,p=1$` y no contiene la contraseña. Ninguna respuesta (`register`, `/v1/me`) incluye `password` ni `passwordHash`
  - Cubre: CA-001-24, FR-028
- [ ] T035 [P] [US1] Test **CA-001-28 (registro)** en `apps/api/test/e2e/ca-001-28-registro.e2e-spec.ts`: registro con `ó` descompuesta (U+006F U+0301) → `201`. El hash guardado verifica contra la forma compuesta (U+00F3). La longitud se mide tras NFC: una contraseña de 12 puntos de código en NFC que tenía 13 descompuesta se acepta
  - Cubre: CA-001-28 (parte de registro), FR-031
- [ ] T036 [P] [US1] Test **CA-001-29 (registro)** en `apps/api/test/e2e/ca-001-29-registro.e2e-spec.ts`: `'a'.repeat(1025)` y `'😀'.repeat(257)` (1 028 bytes) → `400 PASSWORD_TOO_LONG` y 0 cuentas. Un spy sobre `PasswordHasher.hash` confirma que no se llamó
  - Cubre: CA-001-29 (parte de registro), FR-032
- [ ] T037 [P] [US1] Tests RNTL de la pantalla de registro en `apps/mobile/__tests__/register.test.tsx` (API mockeada):
  - `CA-001-01`: el envío válido llama a `signIn` y navega a Inicio.
  - `CA-001-02`: un email inválido muestra el mensaje sin llamar a la API.
  - `CA-001-03`: un 409 muestra el mensaje y un enlace a Ingreso.
  - `CA-001-04`: la prevalidación con `'😀'.repeat(6) + 'abcde'` muestra el mínimo de 12.
  - `CA-001-05`: `'😀'` + 127 letras **no** muestra error.
  - `CA-001-08`: un 129 muestra el máximo.
  - `CA-001-23`: un 429 muestra el mensaje de espera.
  - Edge "sin conexión": un `NetworkError` muestra el mensaje y conserva el email escrito.
  - Edge "contraseña con espacios": `'  frase con espacios al borde  '` llega **intacta** (sin recortar) a `register()`.
  - Cubre: CA-001-01, -02, -03, -04, -05, -08, -23 (app), FR-011 (app), principio III (pantallas con RNTL)

### Implementación de User Story 1

- [ ] T038 [US1] Modelos y **migración 1** en `apps/api/prisma/schema.prisma` y `apps/api/prisma/migrations/<timestamp>_auth_core/` (con `prisma migrate dev --name auth_core` contra la BD local). Restricciones literales de [data-model.md](./data-model.md):
  - `User`:
    - `id` `uuid` PK
    - `email` `varchar(254)` **UNIQUE**, siempre normalizado
    - `passwordHash` `text`, cadena PHC de argon2id
    - `createdAt` `timestamptz`
  - `Session`:
    - `id` `uuid` PK
    - `userId` `uuid` FK → `User.id` (`onDelete: Cascade`), indexado
    - `createdAt`, `lastUsedAt` y `expiresAt` `timestamptz`
    - `revokedAt` `timestamptz` nullable
    - `revokedReason` enum `SessionRevokedReason { LOGOUT, REFRESH_REUSE }` nullable
  - `RefreshToken`:
    - `id` `uuid` PK
    - `sessionId` `uuid` FK → `Session.id` (`onDelete: Cascade`), indexado
    - `tokenHash` `char(64)` **UNIQUE**
    - `createdAt` `timestamptz`
    - `usedAt` `timestamptz` nullable
    - `replacedById` `uuid` nullable, FK → `RefreshToken.id` (`onDelete: SetNull`)
  - `RateLimitEvent`:
    - `id` `bigserial` PK
    - `scope` enum `RateLimitScope { LOGIN_FAILURE, REGISTER_REQUEST }`
    - `originKey` `char(64)`
    - `occurredAt` `timestamptz`
    - índice compuesto `(scope, originKey, occurredAt)`
  - Todas las tablas usan `@@map` en `snake_case`. **Sin** `@default(now())` ni `@updatedAt` en campos de negocio.
  - Actualizar la lista de tablas de `apps/api/test/support/reset-db.ts`.
  - ⚠ Confirmación: crea y aplica una migración
  - Cubre: FR-001, FR-004, FR-013 (índice único), FR-019 (eventos), entidades "Cuenta" y "Sesión"
- [ ] T039 [P] [US1] `apps/api/src/auth/email.ts`:
  - `normalizeEmail(raw)`: trim + `toLowerCase()`.
  - `isValidEmail(email)`: regex `^[^\s@]+@[^\s@]+\.[^\s@]+$` y ≤ 254.
  - Tests unitarios en `apps/api/src/auth/email.spec.ts`.
  - Cubre: FR-002, FR-003
- [ ] T040 [P] [US1] Normalización y medida de la contraseña en la API:
  - `apps/api/src/auth/password-policy/normalize-password.ts` (`pw.normalize('NFC')`).
  - `apps/api/src/auth/password-policy/password-length.ts`: `passwordLength(pw) = [...pw.normalize('NFC')].length`, con un comentario que explica por qué nunca `.length`.
  - Tests unitarios en `apps/api/src/auth/password-policy/password-length.spec.ts` con la tabla de casos de borde de research.md § R3.
  - Cubre: FR-031, FR-007
- [ ] T041 [P] [US1] Lista prohibida:
  - Script `apps/api/scripts/build-password-blocklist.ts`. Lee las fuentes de SecLists (`100k-most-used-passwords-NCSC` y `10-million-password-list-top-1000000`), descargadas a mano en una carpeta fuera del repo y pasadas por argumento, más `apps/api/src/auth/password-policy/data/common-passwords.es.txt`.
  - El script aplica NFC + minúsculas, filtra a 12–128 puntos de código, elimina duplicados y escribe `apps/api/src/auth/password-policy/data/blocklist.txt`, con una cabecera que lleva el commit SHA de SecLists y la licencia MIT.
  - Crear `common-passwords.es.txt` curado: `contraseña123`, `contrasena123`, `123456789012`, `colombia2026`, `teamo123456789`, etc.
  - Límite: < 2 MB.
  - ⚠ Confirmación: descarga externa de SecLists (la puede hacer la persona)
  - Cubre: FR-009, SC-003 (datos)
- [ ] T042 [US1] `apps/api/src/auth/password-policy/blocklist.ts` (carga `blocklist.txt` en un `Set<string>` al arrancar) y `apps/api/src/auth/password-policy/password-policy.service.ts` con `validateForRegistration(rawPassword, normalizedEmail)`, en este orden:
  1. `Buffer.byteLength(raw,'utf8') > 1024` → `PASSWORD_TOO_LONG`
  2. NFC
  3. `< 12` → `PASSWORD_TOO_SHORT`
  4. `> 128` → `PASSWORD_TOO_LONG`
  5. `blocklist.has(nfc.toLowerCase())` → `PASSWORD_TOO_COMMON`
  6. `nfc.trim().toLowerCase() === normalizedEmail` → `PASSWORD_TOO_PREDICTABLE`
  - Devuelve la forma NFC. Sin reglas de composición.
  - Tests unitarios en `password-policy.service.spec.ts` (depende de T040 y T041).
  - Cubre: FR-006, FR-007, FR-008, FR-009, FR-010, FR-011, FR-031, FR-032 (registro)
- [ ] T043 [P] [US1] `apps/api/src/auth/password-hasher.ts`:
  - `hash(nfc)` y `verify(hash, nfc)` con argon2id `{ memoryCost: 19456, timeCost: 2, parallelism: 1 }` en formato PHC.
  - Test unitario en `password-hasher.spec.ts`.
  - Cubre: FR-028
  - 📦 Dependencias (api): `argon2`
- [ ] T044 [US1] `apps/api/src/sessions/token.service.ts`:
  - `signAccess({ sub, sid })`: HS256, 15 min, `iat`/`exp` desde `CLOCK`.
  - `verifyAccess(token)`: `clockTimestamp` desde `CLOCK`.
  - `generateRefresh()`: 32 bytes `randomBytes` en base64url, 43 caracteres.
  - `hashRefresh(t)`: SHA-256 hex.
  - Test unitario en `token.service.spec.ts` con `FakeClock` (expira a los 15 min).
  - Cubre: FR-021 (base), R4
  - 📦 Dependencias (api): `@nestjs/jwt`
- [ ] T045 [US1] `apps/api/src/sessions/session.service.ts` (+ `sessions.module.ts`):
  - `open(userId, tx?)` crea `Session` (`createdAt = lastUsedAt = now`, `expiresAt = now + 30 días`) y su primer `RefreshToken`, y devuelve el par de tokens.
  - Test unitario en `session.service.spec.ts`.
  - Cubre: FR-001 (abrir sesión), FR-027 (una sesión por dispositivo)
- [ ] T046 [US1] `apps/api/src/sessions/access-token.guard.ts`, registrado como `APP_GUARD` (deny by default; `@Public()` en `/health`):
  - Verifica el JWT con `CLOCK` y carga la `Session` por `sid`.
  - Exige `revokedAt IS NULL`, `expiresAt > now` y `userId = sub`, y asigna `req.user`.
  - En cualquier otro caso → `401 SESSION_INVALID`.
  - Cubre: FR-022, FR-023, FR-025
- [ ] T047 [US1] Límite de registro:
  - `apps/api/src/throttling/rate-limit-events.service.ts`: `countSince(scope, originKey, since)`; `record(scope, originKey)`, que además borra los eventos de esa clave con `occurredAt < now − 1 h`.
  - `apps/api/src/throttling/register-rate-limit.guard.ts`: guard (corre antes del `ValidationPipe`). Si hay ≥ 5 `REGISTER_REQUEST` en la última hora → `429 REGISTER_RATE_LIMITED` con `retryAt = evento más antiguo de la ventana + 1 h`. Si no, registra el evento.
  - `apps/api/src/throttling/throttling.module.ts`.
  - Tests unitarios de `rate-limit-events.service.spec.ts`.
  - Cubre: FR-019
- [ ] T048 [US1] Registro:
  - `apps/api/src/auth/dto/register.dto.ts`: `email` con `@Transform(normalizeEmail)` y validación → `INVALID_EMAIL`; `password` string **sin** `@Transform`.
  - `apps/api/src/auth/auth.service.ts#register`:
    - Política → hash → transacción `User` + `session.open`.
    - **Sin** consultar antes si el email existe. **`P2002` → `409 EMAIL_ALREADY_REGISTERED`** con el mensaje exacto.
  - `apps/api/src/auth/auth.controller.ts`: `POST /v1/auth/register` (`@Public()`, `@UseGuards(RegisterRateLimitGuard)`, `201`).
  - `apps/api/src/auth/auth.module.ts`.
  - Hace pasar T022–T036.
  - Cubre: FR-001, FR-002, FR-003, FR-004, FR-005, FR-013
- [ ] T049 [US1] `apps/api/src/users/me.controller.ts` (+ `users.module.ts`): `GET /v1/me` devuelve `{ id, email, createdAt }` de `@CurrentUser().userId`. No acepta parámetros
  - Cubre: FR-023
- [ ] T050 [US1] Registro en la app:
  - `apps/mobile/src/auth/password-length.ts`: la misma `passwordLength()` que la API, con un test unitario de la tabla de R3.
  - `apps/mobile/src/api/auth.ts`: `register()` y `me()`.
  - `apps/mobile/app/(auth)/register.tsx`: email + `PasswordField`, prevalidación de email y longitud, mensajes por código y enlace a Ingreso en el 409.
  - `apps/mobile/app/(app)/index.tsx`: Inicio mínimo con el email de la cuenta (vía `me()`).
  - Hace pasar T037.
  - Cubre: FR-001, FR-003, FR-005, FR-006, FR-008 (app), Edge "Sin conexión"

**Checkpoint**: US1 funciona y se prueba por sí sola (registro y `GET /v1/me`); MVP demostrable.

---

## Phase 4: User Story 2 — Ingresar a mi cuenta (Priority: P2)

**Goal**: una persona con cuenta ingresa, la sesión dura 30 días desde el último uso y se renueva, los límites protegen contra fuerza bruta, y solo ve sus propios datos.

**Independent Test**: con una cuenta creada en el setup del test, `POST /v1/auth/login` correcto → `200`. Credenciales incorrectas o email inexistente → la misma respuesta `401`. Los límites por email y por origen se aplican con `FakeClock`. `refresh` a los 29 días funciona y a los 30 días + 1 ms falla. Un recurso de otra cuenta → `404`.

### Tests para User Story 2 (escribir primero; deben fallar)

- [ ] T051 [P] [US2] Test **CA-001-10** en `apps/api/test/e2e/ca-001-10.e2e-spec.ts`: con la cuenta `ana@ejemplo.com` creada, `POST /v1/auth/login` correcto → `200` con tokens. `GET /v1/me` → Ana. Se creó una `Session` nueva
  - Cubre: CA-001-10, FR-014
- [ ] T052 [P] [US2] Test **CA-001-11** en `apps/api/test/e2e/ca-001-11.e2e-spec.ts`: login con `' ANA@ejemplo.com '` y la contraseña correcta → `200` en la misma cuenta (mismo `user.id`)
  - Cubre: CA-001-11, FR-002, FR-014
- [ ] T053 [P] [US2] Test **CA-001-12** en `apps/api/test/e2e/ca-001-12.e2e-spec.ts`: contraseña incorrecta → `401 INVALID_CREDENTIALS` con el mensaje "Email o contraseña incorrectos."
  - Cubre: CA-001-12, FR-015
- [ ] T054 [P] [US2] Test **CA-001-13** en `apps/api/test/e2e/ca-001-13.e2e-spec.ts` (determinista, sin medir tiempos):
  - `nadie@ejemplo.com` → cuerpo **idéntico byte a byte** al de CA-001-12.
  - Un spy sobre `PasswordHasher.verifyOrDummy` confirma que con email inexistente **se ejecuta argon2 contra el hash ficticio**, con el mismo número de llamadas que con contraseña incorrecta.
  - Cada caso usa una cuenta o un email distinto y `uniqueOrigin()`, para que ningún límite de FR-016 intervenga (todas las respuestas deben ser `401`, nunca `429`).
  - La medición estadística de tiempos de SC-006 se hace en local en T087 (nota de ejecución local del plan).
  - Cubre: CA-001-13, FR-015, SC-006 (parte determinista)
- [ ] T055 [P] [US2] Test **CA-001-14** en `apps/api/test/e2e/ca-001-14.e2e-spec.ts` (`FakeClock`):
  - 5 fallos de `ana@` → un intento con la contraseña **correcta** antes de 1 min → `429 LOGIN_TEMPORARILY_BLOCKED` con `retryAt = 5.º fallo + 1 min` y `Retry-After`.
  - Tras cada espera, el siguiente fallo impone 2, 4, 8, 15 y 15 min.
  - Con `nadie@ejemplo.com` la secuencia y los cuerpos son idénticos.
  - La secuencia de `ana@` se envía desde un origen (`uniqueOrigin()`) y la de `nadie@` desde otro, para que el límite por origen (20 fallos en 15 min) no se mezcle con el límite por email.
  - Cubre: CA-001-14, FR-016 (por email), FR-017
- [ ] T056 [P] [US2] Test **CA-001-26** en `apps/api/test/e2e/ca-001-26.e2e-spec.ts` (`FakeClock`), con tres casos, cada uno partiendo de 4 fallos:
  - (a) `advance(1 h)` + 1 fallo → sin espera.
  - (b) ingreso exitoso + 1 fallo → sin espera.
  - (c) **`advance(59 min)` + 1 fallo → espera de 1 min (5.º fallo)**.
  - Cubre: CA-001-26, FR-016, FR-018
- [ ] T057 [P] [US2] Test **CA-001-27** en `apps/api/test/e2e/ca-001-27.e2e-spec.ts` (`FakeClock`):
  - 20 fallos desde el origen A repartidos entre emails distintos → Ana con credenciales correctas desde A → `429` con `retryAt` y el mismo cuerpo que para un email inexistente.
  - Desde el origen B → `200`.
  - `advance(15 min)` → desde A `200`.
  - Cubre: CA-001-27, FR-016 (por origen), FR-017
- [ ] T058 [P] [US2] Test **CA-001-30** en `apps/api/test/e2e/ca-001-30.e2e-spec.ts` (`FakeClock`):
  - 5 fallos → 3 intentos durante la espera de 1 min (todos `429`). `lockedUntil` no cambia y los eventos `LOGIN_FAILURE` del origen no aumentan.
  - Tras `advance` hasta el fin de la espera, 1 fallo → espera de **2 min** (6.º fallo).
  - Cubre: CA-001-30, FR-016
- [ ] T059 [P] [US2] Test **CA-001-09 (ingreso)** en `apps/api/test/e2e/ca-001-09-ingreso.e2e-spec.ts`: tras registrar contraseñas con espacios alrededor, tildes, `ñ`, emoji y `'👨‍👩‍👧'`, el login con la cadena exacta → `200`; con la versión recortada → `401`
  - Cubre: CA-001-09 (parte de ingreso), FR-011
- [ ] T060 [P] [US2] Test **CA-001-28 (ingreso)** en `apps/api/test/e2e/ca-001-28-ingreso.e2e-spec.ts`: registro con `ó` descompuesta e ingreso con la compuesta → `200`, y viceversa → `200`
  - Cubre: CA-001-28 (parte de ingreso), FR-031
- [ ] T061 [P] [US2] Test **CA-001-29 (ingreso)** en `apps/api/test/e2e/ca-001-29-ingreso.e2e-spec.ts`: login de `ana@` con `'a'.repeat(1025)` → `401 INVALID_CREDENTIALS` con el cuerpo genérico. `consecutiveFailures` aumenta en 1 y se registra un `LOGIN_FAILURE`. Un spy confirma que `PasswordHasher.verify` **no** se llamó
  - Cubre: CA-001-29 (parte de ingreso), FR-032, FR-015
- [ ] T062 [P] [US2] Test **CA-001-15** en `apps/api/test/e2e/ca-001-15.e2e-spec.ts`:
  - Cuentas Ana y Beto. `GET /v1/me` con el token de Ana → Ana. `GET /v1/me?userId=<beto>` → `400 VALIDATION_ERROR`.
  - Con el módulo de prueba (T073), `GET /__probe/sessions/<sesión de Beto>` con el token de Ana → `404 NOT_FOUND` **idéntico** al de un UUID inexistente.
  - **SC-004**: 0 % de éxito.
  - Cubre: CA-001-15, FR-023, FR-024, SC-004
- [ ] T063 [P] [US2] Test **CA-001-16** en `apps/api/test/e2e/ca-001-16.e2e-spec.ts` (`FakeClock`):
  - Login → `advance(29 d)` → `POST /v1/auth/refresh` → `200`, y `session.expiresAt = now + 30 d`.
  - `advance(29 d)` otra vez → `refresh` → `200` (el plazo se renovó).
  - Una solicitud autenticada a `GET /v1/me` también actualiza `lastUsedAt`.
  - Cubre: CA-001-16, FR-020, FR-021
- [ ] T064 [P] [US2] Test **CA-001-17** en `apps/api/test/e2e/ca-001-17.e2e-spec.ts` (`FakeClock`): login → `advance(30 d + 1 ms)` sin solicitudes → `refresh` → `401 SESSION_INVALID`, y `GET /v1/me` con un acceso firmado en ese momento para esa `sid` → `401`
  - Cubre: CA-001-17, FR-021, FR-022
- [ ] T065 [P] [US2] Test **CA-001-31** en `apps/api/test/e2e/ca-001-31.e2e-spec.ts` (`FakeClock`):
  - `it('CA-001-31: renovar C1→C2 deja C1 como usada')`.
  - `it('CA-001-31: presentar C1 dentro de 30 s con C2 sin usar se acepta y la sesión sigue abierta (gracia)')`.
  - `it('CA-001-31: presentar C1 pasados 30 s cierra la sesión (REFRESH_REUSE) y C2 deja de servir')`.
  - `it('CA-001-31: presentar C1 cuando C2 ya se usó cierra la sesión, dentro o fuera de los 30 s')`.
  - Cubre: CA-001-31, FR-033, FR-020, FR-021
- [ ] T066 [P] [US2] Tests RNTL en `apps/mobile/__tests__/login.test.tsx` y `apps/mobile/__tests__/session.test.tsx` (API mockeada):
  - `CA-001-10`: un login válido navega a Inicio.
  - `CA-001-12`: un 401 muestra el mensaje genérico.
  - `CA-001-14`: un 429 muestra la hora de `retryAt`.
  - `CA-001-16`: con tokens en secure store, al arrancar la app el estado pasa a `signedIn` sin pedir credenciales.
  - `CA-001-17`: un 401 en `refresh` → `signOutLocal` y redirección a Ingreso.
  - Además: dos solicitudes con 401 simultáneas provocan **un solo** `refresh` (single-flight).
  - `CA-001-17` (sin conexión): `hydrate()` **no** invoca `fetch` (abrir la app sin conexión no renueva el plazo).
  - Edge "sin conexión" en el ingreso: un `NetworkError` muestra el mensaje de sin conexión y conserva el email escrito.
  - Cubre: CA-001-10, -12, -14, -16, -17 (app), FR-021 (app), FR-022, Edge "Sin conexión"

### Implementación de User Story 2

- [ ] T067 [US2] Modelos y **migración 2** en `apps/api/prisma/schema.prisma` y `apps/api/prisma/migrations/<timestamp>_login_throttle/`. Restricciones literales de [data-model.md](./data-model.md):
  - `LoginEmailThrottle`:
    - `emailKey` `char(64)` PK
    - `consecutiveFailures` `integer` ≥ 0
    - `lastFailureAt` `timestamptz`
    - `lockedUntil` `timestamptz` nullable
  - `LoginOriginLock`:
    - `originKey` `char(64)` PK
    - `lockedUntil` `timestamptz`
  - Actualizar `apps/api/test/support/reset-db.ts`.
  - ⚠ Confirmación: crea y aplica una migración
  - Cubre: FR-016, FR-017, FR-018 (persistencia)
- [ ] T068 [US2] Hash ficticio en `apps/api/src/auth/password-hasher.ts`:
  - Al arrancar (`onModuleInit`) se genera un hash argon2id con los mismos parámetros sobre un valor aleatorio. Nunca se guarda en BD.
  - `verifyOrDummy(hash | null, nfc)` siempre ejecuta argon2 y devuelve `false` si `hash` es `null`.
  - Ampliar `password-hasher.spec.ts`.
  - Cubre: FR-015, SC-006
- [ ] T069 [US2] `apps/api/src/throttling/login-throttle.service.ts`, con advisory lock `pg_advisory_xact_lock(hashtextextended(emailKey, 0))` en la transacción del ingreso:
  - `assertAllowed(emailKey, originKey)`:
    - `login_origin_lock.lockedUntil > now` → `429`.
    - Reinicio perezoso si `lastFailureAt + 1 h ≤ now`.
    - `lockedUntil > now` → `429` con `retryAt`, **sin** modificar nada.
  - `recordFailure`:
    - `consecutiveFailures++`; si n ≥ 5, `lockedUntil = now + min(15, 2^(n−5))` min.
    - `record(LOGIN_FAILURE)`; si hay ≥ 20 en 15 min → upsert `login_origin_lock = now + 15 min`.
  - `recordSuccess`: borra la fila del email.
  - Tests unitarios en `login-throttle.service.spec.ts`.
  - Cubre: FR-016, FR-017, FR-018
- [ ] T070 [US2] Ingreso:
  - `apps/api/src/auth/dto/login.dto.ts`: `email` con `@Transform(normalizeEmail)`, ≤ 254, sin error 400 por formato; `password` sin `@Transform`.
  - `apps/api/src/auth/auth.service.ts#login`, en este orden:
    1. `OriginResolver`
    2. Transacción + advisory lock
    3. `assertAllowed`
    4. Si son > 1 024 bytes → `recordFailure` + `401` sin argon2
    5. NFC → buscar usuario → `verifyOrDummy`
    6. Fallo → `recordFailure` + `401 INVALID_CREDENTIALS`; éxito → `recordSuccess` + `session.open` + `200`
  - `POST /v1/auth/login` (`@Public()`) en `auth.controller.ts`.
  - Nota: la transacción con advisory lock retiene una conexión del pool mientras corre argon2 (~50–250 ms). Se documenta `connection_limit` en `DATABASE_URL` de `apps/api/.env.example` (por ejemplo `?connection_limit=10`). Es aceptable a esta escala; se revisa si hay ingresos concurrentes sostenidos.
  - Hace pasar T051–T061.
  - Cubre: FR-014, FR-015, FR-016, FR-017, FR-018, FR-031, FR-032 (ingreso)
- [ ] T071 [US2] `session.service.ts#touch(sessionId)`: `lastUsedAt = now` y `expiresAt = now + 30 d`. Se llama desde `access-token.guard.ts` en cada solicitud válida (un `UPDATE` por PK)
  - Cubre: FR-021
- [ ] T072 [US2] Renovación:
  - `apps/api/src/sessions/dto/refresh.dto.ts`: `refreshToken` con el patrón `^[A-Za-z0-9_-]{43}$`.
  - `session.service.ts#rotate(token)` en una transacción con `SELECT … FOR UPDATE`, aplicando las 4 reglas de data-model.md (inválido → 401; sin usar → rotar + `touch`; gracia de 30 s con reemplazo sin usar → crear `t''`, **reasignar `t.replacedById = t''.id` y solo después borrar el reemplazo anterior**; resto → revocar con `REFRESH_REUSE` y 401).
  - `POST /v1/auth/refresh` (`@Public()`) en `auth.controller.ts`.
  - Hace pasar T063–T065.
  - Cubre: FR-020, FR-021, FR-022, FR-033
- [ ] T073 [US2] Recursos ajenos:
  - `apps/api/src/common/ownership/find-owned-or-throw.ts`: helper genérico que busca por `id` **y** `userId` y, si no encuentra, lanza `404 NOT_FOUND`, sin distinguir entre "no existe" y "es de otro".
  - Módulo solo para e2e en `apps/api/test/fixtures/owned-probe.module.ts`: `GET /__probe/sessions/:id` usa el helper sobre `Session`. Lo monta únicamente `createTestApp()`.
  - Hace pasar T062.
  - Cubre: FR-024, principio IV
- [ ] T074 [US2] Sesión persistente en la app:
  - En `apps/mobile/src/api/client.ts`: Bearer automático y, ante un `401 SESSION_INVALID`, **un solo** `refresh` en curso (promesa compartida) y reintento de la solicitud. Si `refresh` falla → `sessionStore.signOutLocal()`.
  - En `apps/mobile/src/auth/session-store.ts`: `hydrate()` al arrancar lee secure store → `signedIn`/`signedOut`, sin red. No renueva el plazo (FR-021).
  - Hace pasar la parte `CA-001-16` y `CA-001-17` de T066.
  - Cubre: FR-020, FR-021, FR-022
- [ ] T075 [US2] Pantalla de ingreso:
  - `apps/mobile/src/api/auth.ts#login` y `apps/mobile/app/(auth)/login.tsx`: email + `PasswordField` (`current-password`), mensaje genérico en 401, hora local de `retryAt` en 429 y "sin conexión" conservando el email. Enlace a Registro.
  - Hace pasar la parte `CA-001-10`, `CA-001-12` y `CA-001-14` de T066.
  - Cubre: FR-014, FR-015, FR-017 (app)

**Checkpoint**: US1 y US2 funcionan y se prueban cada una por separado.

---

## Phase 5: User Story 3 — Cerrar sesión (Priority: P3)

**Goal**: la persona cierra la sesión del dispositivo actual. La credencial anterior deja de servir de inmediato y las sesiones de otros dispositivos siguen vigentes.

**Independent Test**: con una sesión abierta, `POST /v1/auth/logout` → `204`. Después, tanto el acceso como la renovación anteriores → `401`. Una segunda sesión de la misma cuenta sigue funcionando. En la app, "Cerrar sesión" vuelve a Ingreso y borra el secure store.

### Tests para User Story 3 (escribir primero; deben fallar)

- [ ] T076 [P] [US3] Test RNTL **CA-001-18** en `apps/mobile/__tests__/home.test.tsx`:
  - Pulsar "Cerrar sesión" → se llama a `logout`, el secure store queda vacío, se navega a Ingreso y no se muestra el email.
  - Sin conexión: igualmente vuelve a Ingreso y guarda `pendingLogoutToken`. En el siguiente arranque con red se envía el `logout` pendiente y se borra.
  - Cubre: CA-001-18, FR-026, Edge "Cerrar sesión sin conexión"
- [ ] T077 [P] [US3] Test **CA-001-19** en `apps/api/test/e2e/ca-001-19.e2e-spec.ts`:
  - Login → `logout` → `GET /v1/me` con el acceso anterior (aún dentro de sus 15 min) → `401 SESSION_INVALID`, y `refresh` con el token anterior → `401`.
  - Repetir `logout` → `204` (idempotente). Una solicitud sin token → `401` sin datos.
  - **SC-005**: el 100 % de las reutilizaciones se rechaza.
  - Cubre: CA-001-19, FR-022, FR-025, FR-026, SC-005
- [ ] T078 [P] [US3] Test **CA-001-20** en `apps/api/test/e2e/ca-001-20.e2e-spec.ts`: dos logins de Ana (sesiones S1 y S2) → `logout` de S1 → `GET /v1/me` y `refresh` con S2 → `200`. S1 queda con `revokedReason = LOGOUT` y S2 con `revokedAt` nulo
  - Cubre: CA-001-20, FR-027

### Implementación de User Story 3

- [ ] T079 [US3] Cierre de sesión en la API:
  - `apps/api/src/auth/dto/logout.dto.ts`: `refreshToken` con el patrón `^[A-Za-z0-9_-]{43}$`.
  - `session.service.ts#revoke(refreshToken)`: busca por `tokenHash` y fija `revokedAt = now` y `revokedReason = LOGOUT` **solo en esa sesión**. No hace nada si no existe o ya está cerrada.
  - `POST /v1/auth/logout` (`@Public()`, `204`) en `auth.controller.ts`.
  - Hace pasar T077 y T078.
  - Cubre: FR-026, FR-027
- [ ] T080 [US3] Cierre de sesión en la app:
  - `apps/mobile/src/api/auth.ts#logout`.
  - `session-store.ts#signOut()`: intenta `logout`; si hay `NetworkError`, guarda `pendingLogoutToken`. Siempre borra los tokens y pasa a `signedOut`. `hydrate()` reintenta el logout pendiente.
  - Botón "Cerrar sesión" en `apps/mobile/app/(app)/index.tsx`.
  - Hace pasar T076.
  - Cubre: FR-026, FR-027 (app)

**Checkpoint**: las tres historias funcionan por separado.

---

## Phase 6: Polish y verificaciones transversales

**Purpose**: CA transversal de logs, revisiones manuales, e2e móvil, cobertura y trazabilidad.

- [ ] T081 Test **CA-001-25** (flujo completo) en `apps/api/test/e2e/ca-001-25.e2e-spec.ts`:
  - Con `log-capture`, se ejecutan registro, ingreso exitoso, ingreso fallido, refresh y logout.
  - La salida capturada no contiene la contraseña, el `accessToken`, el `refreshToken` ni el email usados.
  - El interceptor (T015) ya existe. Este test va al final porque necesita los endpoints de las tres historias; su parte unitaria (T014) sí precedió a la implementación. Si falla, se corrige el componente que filtre el dato.
  - Cubre: CA-001-25, FR-029
- [ ] T082 [P] **Revisión FR-012** (sin test automatizado): en iOS y Android se comprueba que en los campos de contraseña de Registro e Ingreso se puede pegar texto y que el gestor de contraseñas del sistema ofrece autocompletar y guardar. También se revisan las props de `apps/mobile/src/components/PasswordField.tsx`. El resultado (dispositivo, versión del SO, fecha, OK/NO) se registra en `specs/001-autenticacion/checklists/revision-manual.md`
  - Cubre: FR-012
- [ ] T083 [P] **Revisión FR-030** (sin test automatizado), registrada en `specs/001-autenticacion/checklists/revision-manual.md`:
  - `grep -rn "AsyncStorage" apps/mobile` sin resultados.
  - La regla de lint de T002 está activa.
  - `apps/mobile/src/auth/token-storage.ts` es el único punto que guarda credenciales y solo usa `expo-secure-store`.
  - Los tokens no se guardan en el estado persistido de Zustand ni en la caché de TanStack Query.
  - Cubre: FR-030
- [ ] T084 [P] Flujos Maestro (ejecución local, fuera del CI; ver la nota del Constitution Check):
  - `apps/mobile/.maestro/registro.yaml`: CA-001-01, **SC-001** < 2 min. La contraseña lleva una tilde, para confirmar NFC en Hermes en un dispositivo real (T008).
  - `apps/mobile/.maestro/ingreso.yaml`: CA-001-10, **SC-002** < 30 s, cerrar y reabrir la app sigue en Inicio (CA-001-16).
  - `apps/mobile/.maestro/cerrar-sesion.yaml`: CA-001-18.
  - Cubre: SC-001, SC-002, principio III (flujos críticos)
- [ ] T085 Trazabilidad y cobertura:
  - Ejecutar `npm run -w apps/api test:cov` (≥ 80 % en `*.service.ts`) y el bucle de trazabilidad de `specs/001-autenticacion/quickstart.md` para CA-001-01…31. Ningún CA debe faltar.
  - Comprobar que `contracts/openapi.yaml` coincide con lo implementado (rutas, códigos y `ErrorCode`). Si hay diferencias, se detiene y se propone ajustar el contrato.
  - Cubre: principios II y III
- [ ] T086 Validación de `specs/001-autenticacion/quickstart.md`: escenarios manuales 1–12 contra la API local, `npm audit --audit-level=high` sin vulnerabilidades altas ni críticas, y anotar cualquier diferencia en el propio quickstart
  - Cubre: principio V, verificación final antes de Verify
- [ ] T087 [P] Medición estadística de **SC-006** en local (fuera del CI; ver la nota de ejecución local del plan y research.md § R8):
  - Script `apps/api/test/perf/sc-006-timing.perf-spec.ts`, excluido de los proyectos `unit` y `e2e` de `apps/api/jest.config.ts`, con el script `test:timing` en `apps/api/package.json`.
  - N = 30 ingresos por caso (email inexistente y contraseña incorrecta), cada uno con email o cuenta y origen propios. Se descarta el primer 10 % (calentamiento) y se exige que las medianas no difieran más del 20 %.
  - El resultado (equipo, fecha, medianas, OK/NO) se registra en `specs/001-autenticacion/checklists/revision-manual.md`.
  - Cubre: SC-006 (parte estadística), FR-015

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (T001–T010)**: T001 → T002 → T003 → T004 → T005. T006 → T007 → T008 en paralelo con la API desde T003. T009 depende de T004, T005 y T007. T010 al final.
- **Foundational (T011–T021)**: depende de Setup. T013 → T014 → T015. T018 depende de T011, T013 y T015. T020 depende de T019. Bloquea todas las historias.
- **US1 (T022–T050)**: depende de Foundational.
- **US2 (T051–T075)**: depende de Foundational **y** de T038 (User/Session), T042 (política: NFC), T043 (hasher), T044–T046 (tokens, sesión, guard) y T048 (los tests crean cuentas con `register`). Por eso, en la práctica, va después de US1.
- **US3 (T076–T080)**: depende de US1 (sesión) y de T072 (`refresh`, usado en CA-001-19 y CA-001-20). La pantalla de Inicio de T050 y la base de sesión de T074.
- **Polish (T081–T087)**: depende de las historias que se entreguen.

### Dentro de cada historia

1. Tests (todos [P]: un archivo por CA) → se commitean en rojo.
2. Migración → helpers [P] → servicios → guards/controladores → app.
3. Cada tarea de implementación dice qué tests hace pasar.

### Matriz CA → tarea de test → tarea que lo hace pasar

| CA | Test | Implementación | CA | Test | Implementación |
|---|---|---|---|---|---|
| CA-001-01 | T022, T037, T084 | T048, T050 | CA-001-16 | T063, T066 | T071, T072, T074 |
| CA-001-02 | T023, T037 | T039, T048, T050 | CA-001-17 | T064, T066 | T072, T074 |
| CA-001-03 | T024, T037 | T039, T048, T050 | CA-001-18 | T076, T084 | T080 |
| CA-001-04 | T025, T037 | T040, T042, T050 | CA-001-19 | T077 | T079 |
| CA-001-05 | T026, T037 | T040, T042, T050 | CA-001-20 | T078 | T079 |
| CA-001-06 | T027 | T042 | CA-001-21 | T031 | T042 |
| CA-001-07 | T028 | T041, T042 | CA-001-22 | T032 | T048 |
| CA-001-08 | T029, T037 | T042, T050 | CA-001-23 | T033, T037 | T047 |
| CA-001-09 | T030, T059 | T042, T043, T070 | CA-001-24 | T034 | T043 |
| CA-001-10 | T051, T066, T084 | T070, T075 | CA-001-25 | T014, T081 | T015 |
| CA-001-11 | T052 | T070 | CA-001-26 | T056 | T069 |
| CA-001-12 | T053, T066 | T070, T075 | CA-001-27 | T057 | T069 |
| CA-001-13 | T054 | T068, T070 | CA-001-28 | T035, T060 | T040, T042, T070 |
| CA-001-14 | T055, T066 | T069, T075 | CA-001-29 | T036, T061 | T042, T070 |
| CA-001-15 | T062 | T049, T073 | CA-001-30 | T058 | T069 |
| CA-001-31 | T065 | T072, T074 | | | |

FR verificados por revisión: **FR-012 → T082**, **FR-030 → T083** (con prevención en T002 y T020).

### Parallel Opportunities

- Setup: la app (T006–T008) avanza en paralelo con la API (T003–T005).
- Foundational: T011, T012, T016, T017, T019 y T021 en paralelo.
- US1: los 16 tests T022–T037 en paralelo. Después, T039, T040, T041 y T043 en paralelo.
- US2: los 16 tests T051–T066 en paralelo.
- US3: T076–T078 en paralelo.
- Polish: T082, T083, T084 y T087 en paralelo.

---

## Parallel Example: User Story 1

```bash
# Tests de US1 (un archivo por CA, sin dependencias entre sí):
Task: "T022 Test CA-001-01 en apps/api/test/e2e/ca-001-01.e2e-spec.ts"
Task: "T025 Test CA-001-04 en apps/api/test/e2e/ca-001-04.e2e-spec.ts"
Task: "T032 Test CA-001-22 en apps/api/test/e2e/ca-001-22.e2e-spec.ts"
Task: "T037 Tests RNTL de registro en apps/mobile/__tests__/register.test.tsx"

# Helpers de US1 tras la migración T038:
Task: "T039 email.ts"
Task: "T040 normalize-password.ts + password-length.ts"
Task: "T043 password-hasher.ts"
```

---

## Implementation Strategy

### MVP primero (solo US1)

1. Setup (T001–T010) → esqueleto en verde en CI.
2. Foundational (T011–T021).
3. US1 (T022–T050) → **parar y validar**: registro e `/v1/me` en la API y en la app.

### Entrega incremental

1. US1 → validar → demo.
2. US2 → validar (límites con `FakeClock`, sesión de 30 días, aislamiento).
3. US3 → validar.
4. Polish → Verify (`/speckit-analyze` recomendado antes de implementar).

### Recordatorios de CLAUDE.md al implementar

- Una tarea a la vez, mostrando el diff antes de seguir.
- Si algo contradice la spec: detener y proponer el cambio a la spec.
- Confirmación previa en: 📦 dependencias (T001–T007, T013, T020, T043, T044), ⚠ migraciones (T038, T067), descarga externa (T041) y `CLAUDE.md` (T010).
- Nunca leer, crear ni modificar `.env` reales; solo `.env.example`.
