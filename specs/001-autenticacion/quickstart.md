# Quickstart — validación de 001-autenticacion

Guía para comprobar de punta a punta que la feature funciona. No contiene implementación: los detalles están en [plan.md](./plan.md), [data-model.md](./data-model.md) y [contracts/openapi.yaml](./contracts/openapi.yaml).

> ⚠️ **Comandos propuestos.** Aún no existe el esqueleto del monorepo. Los nombres de scripts de abajo son los que se crearán en la tarea de esqueleto. Cuando existan y se hayan verificado, se copiarán a la sección "Comandos" de `CLAUDE.md` (con confirmación, porque modifica ese archivo).

## Prerrequisitos

- Node.js 24 LTS (`.nvmrc`) y npm.
- Docker (solo para PostgreSQL local).
- Para la app: Expo Go o un simulador o emulador iOS/Android.
- Para e2e móvil: Maestro CLI instalado en local.
- Copiar `apps/api/.env.example` a un `.env` local y completar los secretos de desarrollo. El agente nunca lee ni crea `.env` reales: este paso lo hace la persona.

## 1. Levantar el entorno local

```sh
npm ci                                              # instala todos los workspaces
docker compose -f deploy/local/compose.yml up -d    # PostgreSQL 18 (bases dev y test)
npm run -w apps/api prisma:migrate:dev              # aplica migraciones (pide confirmación al agente)
npm run -w apps/api start:dev                       # API en http://localhost:3000
npm run -w apps/mobile start                        # Expo
```

**Esperado**: `GET http://localhost:3000/health` → `200 {"status":"ok"}`.

## 2. Suite automatizada (lo mismo que corre el CI)

```sh
npm run lint
npm run typecheck
npm run -w apps/api test          # unitarios (FakeClock)
npm run -w apps/api test:e2e      # Supertest contra la BD de test
npm run -w apps/api test:cov      # falla si services < 80 %
npm run -w apps/mobile test       # Jest + RNTL
npm audit --audit-level=high
```

**Esperado**: todo en verde; un test por cada `CA-001-01` … `CA-001-30` (buscar `CA-001-` en los nombres de los tests; ningún ID debe faltar).

```sh
# comprobación rápida de trazabilidad: lista los CA sin test
for i in $(seq -w 1 30); do grep -rqs "CA-001-$i" apps/*/src apps/*/test apps/mobile/__tests__ || echo "Falta CA-001-$i"; done
```

## 3. Escenarios manuales contra la API local

Usar `curl` o cualquier cliente HTTP. Para simular orígenes, arrancar con `TRUST_CF_CONNECTING_IP=true` y enviar `CF-Connecting-IP: 203.0.113.X`.

| # | Escenario | Pasos | Resultado esperado |
|---|---|---|---|
| 1 | Registro válido | `POST /v1/auth/register` con `ana@ejemplo.com` y contraseña de 12+ caracteres no común | `201` con `accessToken`, `refreshToken` y `user.email = "ana@ejemplo.com"` |
| 2 | Email repetido con variante | Repetir con `" Ana@Ejemplo.com "` | `409 EMAIL_ALREADY_REGISTERED`, mensaje "Este email ya tiene una cuenta. Ingresa con él." |
| 3 | Contraseña común | Registro con `123456789012` | `400 PASSWORD_TOO_COMMON` |
| 4 | Contraseña = email | Registro de `beto@ejemplo.com` con contraseña `beto@ejemplo.com` | `400 PASSWORD_TOO_PREDICTABLE` |
| 5 | Ingreso genérico | Login con contraseña errónea para Ana y con `nadie@ejemplo.com` | Ambos `401` con cuerpo idéntico |
| 6 | Espera por email | 5 logins fallidos de Ana y luego login correcto | `429 LOGIN_TEMPORARILY_BLOCKED` con `retryAt` ≈ +1 min y `Retry-After` |
| 7 | Campo ajeno | `GET /v1/me?userId=<otro>` o body con `userId` | `400 VALIDATION_ERROR`; `/v1/me` sin parámetros devuelve siempre la cuenta del token |
| 8 | Logout + reutilización | `POST /v1/auth/logout` con el refresh; luego `GET /v1/me` con el access anterior y `POST /v1/auth/refresh` con el refresh anterior | `204`; después `401 SESSION_INVALID` en ambos |
| 9 | Reutilización de refresh | Refrescar con R1 → R2; esperar más de 30 s (o avanzar el reloj en test); refrescar de nuevo con R1 | `401`; R2 también queda inválido (sesión revocada) |
| 10 | Emoji fuera del BMP | Registro con `'😀'` + 127 letras (128 puntos de código, 129 unidades UTF-16) | `201` (CA-001-05) |
| 11 | NFC | Registro con `ó` descompuesta (U+006F U+0301); ingreso con `ó` compuesta (U+00F3) | `200` (CA-001-28) |
| 12 | Logs limpios | Revisar la salida de la API tras los pasos 1–11 | Sin contraseñas, tokens ni emails completos |

Los escenarios con tiempo (30 días, 15 min, 1 h) se validan con los tests e2e que usan `FakeClock`, no a mano.

## 4. App móvil (manual + Maestro)

```sh
maestro test apps/mobile/.maestro/registro.yaml
maestro test apps/mobile/.maestro/ingreso.yaml
maestro test apps/mobile/.maestro/cerrar-sesion.yaml
```

**Esperado**:

- **Registro** (SC-001 < 2 min): el formulario acepta pegar la contraseña, muestra los errores en español y al terminar lleva a Inicio con el email de la cuenta.
- **Ingreso** (SC-002 < 30 s): al cerrar y reabrir la app se sigue en Inicio (FR-020).
- **Cerrar sesión**: vuelve a Ingreso y no muestra datos. En modo avión también vuelve a Ingreso, y al recuperar conexión el servidor revoca la sesión (token pendiente).
- **Sin conexión** al registrar o ingresar: mensaje de "sin conexión", el email escrito se conserva.

## 5. Revisión manual (criterios "verificado por revisión")

- **FR-012**: los campos de contraseña permiten pegar y el gestor de contraseñas del sistema ofrece guardar/autocompletar.
- **FR-030**: los tokens solo se escriben con `expo-secure-store` (la regla de lint prohíbe `AsyncStorage`).
