# Data Model — 001-autenticacion

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Research**: [research.md](./research.md)

Convenciones:

- Nombres de modelos en inglés (PascalCase en Prisma, `snake_case` en tablas vía `@@map`).
- Todas las marcas de tiempo son `timestamptz` en UTC (principio VI) y las asigna la aplicación con el `Clock` inyectado. No se usa `@default(now())` ni `@updatedAt` en campos con lógica de negocio (R7).
- El esquema solo cambia mediante migraciones de Prisma versionadas (principio VII).

## Diagrama

```text
User 1 ──── * Session 1 ──── * RefreshToken
                                   └── replacedBy → RefreshToken (0..1)

LoginEmailThrottle   (independiente: clave = HMAC del email, exista o no la cuenta)
RateLimitEvent       (independiente: clave = HMAC del origen)
LoginOriginLock      (independiente: clave = HMAC del origen)
```

---

## User — "Cuenta de usuario"

| Campo | Tipo (Postgres) | Reglas |
|---|---|---|
| `id` | `uuid` PK | Generado por la app. |
| `email` | `varchar(254)` **UNIQUE** | Siempre **normalizado** (trim + minúsculas, FR-002). Formato validado (FR-003). El índice único garantiza FR-004 y FR-013. |
| `passwordHash` | `text` | Cadena PHC de argon2id calculada sobre la contraseña en NFC (FR-028, FR-031). Nunca se serializa en respuestas. |
| `createdAt` | `timestamptz` | `clock.now()` al registrar. |

- **Unicidad y carreras** (FR-004, FR-013, CA-001-22): no se consulta "¿existe el email?" antes de insertar. Se inserta directamente y, si Prisma lanza `P2002` (violación del índice único de `email`), `auth.service#register` lo traduce a `409 EMAIL_ALREADY_REGISTERED`. Así, dos registros simultáneos producen exactamente una cuenta y un 409, sin ventana de carrera. La transacción User + Session + RefreshToken se revierte entera.
- **Hash ficticio** (FR-015, SC-006): si en el ingreso no existe un `User` con el email, la contraseña se verifica igualmente con argon2id contra un hash ficticio, generado al arrancar con los mismos parámetros que los reales y que nunca se guarda en BD. Así, email inexistente y contraseña incorrecta cuestan el mismo tiempo de CPU y memoria.
- Es la dueña de todos los datos de usuario de features futuras: toda tabla de usuario tendrá `userId` FK → `User.id` y se consultará filtrando por el `userId` del token (principio IV).
- La eliminación (feature 006) borrará en cascada sesiones y tokens.

## Session — "Sesión"

| Campo | Tipo | Reglas |
|---|---|---|
| `id` | `uuid` PK | Va en el claim `sid` del JWT de acceso. |
| `userId` | `uuid` FK → `User.id` (`onDelete: Cascade`), indexado | |
| `createdAt` | `timestamptz` | Momento de inicio. |
| `lastUsedAt` | `timestamptz` | Se actualiza en cada solicitud autenticada válida y en cada renovación (FR-021). |
| `expiresAt` | `timestamptz` | Siempre `lastUsedAt + 30 días`. Se guarda para consultar y depurar fácil. |
| `revokedAt` | `timestamptz` nullable | Se fija al cerrar sesión o al detectar reutilización. |
| `revokedReason` | enum `SessionRevokedReason { LOGOUT, REFRESH_REUSE }` nullable | |

**Estado** (derivado, no se guarda como columna):

```text
                 ┌──────────── uso válido (touch) ────────────┐
                 ▼                                            │
 [abrir] ──► VIGENTE ─────────────────────────────────────────┘
                 │  revokedAt := now (logout | reutilización)
                 ├──────────────────────────────────────► CERRADA   (terminal)
                 │  now ≥ expiresAt
                 └──────────────────────────────────────► EXPIRADA  (terminal)
```

- `VIGENTE` ⇔ `revokedAt IS NULL AND expiresAt > now`.
- Una sesión cerrada o expirada **nunca** vuelve a vigente: un "uso" solo renueva sesiones vigentes.
- Una cuenta puede tener varias sesiones vigentes (FR-027). El logout revoca solo la sesión del token presentado.

## RefreshToken

| Campo | Tipo | Reglas |
|---|---|---|
| `id` | `uuid` PK | |
| `sessionId` | `uuid` FK → `Session.id` (`onDelete: Cascade`), indexado | |
| `tokenHash` | `char(64)` **UNIQUE** | SHA-256 hex del token opaco (32 bytes aleatorios en base64url). El token en claro solo existe en la respuesta y en el secure store del teléfono. |
| `createdAt` | `timestamptz` | |
| `usedAt` | `timestamptz` nullable | Se fija al rotar. |
| `replacedById` | `uuid` nullable, FK → `RefreshToken.id` (`onDelete: SetNull`) | El token emitido al rotar. `SetNull` como red de seguridad: borrar un reemplazo nunca deja una FK colgando. |

**Reglas de rotación** (R4), con `t` = token presentado y `S` = su sesión:

1. `t` no existe o `S` no está vigente → `401 SESSION_INVALID`.
2. `t.usedAt IS NULL` → se marca `usedAt = now`, se crea `t'`, `t.replacedById = t'.id`, `touch(S)` → `200`.
3. `t.usedAt` no nulo **y** `now − t.usedAt ≤ 30 s` **y** `t.replacedBy.usedAt IS NULL` → **gracia** (FR-033): en este orden, (a) se crea `t''`, (b) `t.replacedById = t''.id`, (c) se borra el reemplazo anterior, que ya no está referenciado → `200`.
4. Cualquier otro caso con `t.usedAt` no nulo → **reutilización** (FR-033, CA-001-31): `S.revokedAt = now`, `revokedReason = REFRESH_REUSE` → `401 SESSION_INVALID`.

Todo el proceso corre en una transacción con `SELECT … FOR UPDATE` sobre `t` para evitar carreras.

## LoginEmailThrottle — "Registro de intentos fallidos" (por email)

| Campo | Tipo | Reglas |
|---|---|---|
| `emailKey` | `char(64)` PK | `HMAC-SHA256(THROTTLE_HMAC_SECRET, emailNormalizado)` en hex. Existe aunque no haya cuenta con ese email. |
| `consecutiveFailures` | `integer` ≥ 0 | |
| `lastFailureAt` | `timestamptz` | |
| `lockedUntil` | `timestamptz` nullable | Fin de la espera vigente. |

**Reglas** (FR-016–FR-018, R5):

- Reinicio perezoso: si `lastFailureAt + 1 h ≤ now`, se trata como `consecutiveFailures = 0` y `lockedUntil = null` (FR-018; a los 59 min todavía no se reinicia, CA-001-26).
- Tras el fallo n: si n ≥ 5 → `lockedUntil = now + min(15, 2^(n−5))` minutos.
- Los intentos dentro de la espera no modifican la fila ni generan `RateLimitEvent` (FR-016, CA-001-30).
- Un ingreso exitoso **borra** la fila.
- Todos los accesos se serializan por `emailKey` con `pg_advisory_xact_lock`.

## RateLimitEvent — "Registro de intentos fallidos" (por origen)

| Campo | Tipo | Reglas |
|---|---|---|
| `id` | `bigserial` PK | |
| `scope` | enum `RateLimitScope { LOGIN_FAILURE, REGISTER_REQUEST }` | |
| `originKey` | `char(64)` | `HMAC-SHA256(THROTTLE_HMAC_SECRET, ipNormalizada)`. IPv6 se agrupa por /64. |
| `occurredAt` | `timestamptz` | |

- Índice compuesto `(scope, originKey, occurredAt)` para contar en la ventana.
- `LOGIN_FAILURE`: ventana de 15 min, umbral 20 (FR-016).
- `REGISTER_REQUEST`: ventana de 1 h, umbral 5 (FR-019). Se registran todas las solicitudes de registro salvo las rechazadas por el propio límite.
- Limpieza: al insertar se borran los eventos de la misma `(scope, originKey)` con `occurredAt < now − 1 h`.

## LoginOriginLock

| Campo | Tipo | Reglas |
|---|---|---|
| `originKey` | `char(64)` PK | |
| `lockedUntil` | `timestamptz` | `now + 15 min` al alcanzar 20 fallos en 15 min (upsert). |

- Mientras `lockedUntil > now`, todo `POST /v1/auth/login` desde ese origen → `429 LOGIN_TEMPORARILY_BLOCKED` con `retryAt = lockedUntil`, sin verificar credenciales ni tocar el contador del email (CA-001-27).
- Una fila vencida se ignora y se sobrescribe en el siguiente bloqueo.

---

## Validaciones de entrada (DTOs)

`ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })` global.

| DTO | Campo | Reglas |
|---|---|---|
| `RegisterDto` | `email` | string; `@Transform(normalizeEmail)`; no vacío; regex `^[^\s@]+@[^\s@]+\.[^\s@]+$`; ≤ 254 → `INVALID_EMAIL` |
| | `password` | string; **sin `@Transform`** (no se recorta). El servicio aplica, en este orden: tope de 1 024 bytes sobre la entrada cruda (FR-032) → NFC (FR-031) → longitud 12–128 en puntos de código, `[...pw.normalize('NFC')].length` (FR-006–FR-008) → común → igual al email (FR-009). |
| `LoginDto` | `email` | string; `@Transform(normalizeEmail)`; ≤ 254. Un formato inválido no da error 400: se trata como credenciales incorrectas (FR-015). |
| | `password` | string; sin `@Transform`. Más de 1 024 bytes → intento fallido genérico sin argon2 (FR-032). Si no, NFC y verificación. **No** se aplica la política de longitud ni de lista. |
| `RefreshDto` / `LogoutDto` | `refreshToken` | string base64url de 43 caracteres. |

Cualquier campo no declarado (por ejemplo `userId`) → `400 VALIDATION_ERROR` (FR-023).
