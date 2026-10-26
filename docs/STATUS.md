# STATUS — pagos-recurrentes

**Última actualización:** 2026-10-26
**Fase actual:** Fase 1 — feature `001-autenticacion` — Plan cerrado; sigue **Tasks**

## Fase 0 — Preparación ✅

- [x] Proyecto "SDD Lab: Pagos y Gastos" creado en Claude
- [x] Repo público `pagos-recurrentes`
- [x] Spec Kit v1.0.1 instalado (`--integration claude`, scripts `sh`)
- [x] Constitución v1.0.0 ratificada (8 principios) en `.specify/memory/constitution.md`
- [x] `CLAUDE.md` aprobado; `.gitignore` con protección de secretos
- [x] Ramas `main` y `develop`

## Fase 1 — Feature 001-autenticacion (en curso)

### Specify ✅
- [x] `/speckit-specify` → `spec.md`, dos rondas de revisión, nota de IDs congelados
- [x] Commit `539b30b docs(001): spec inicial de autenticación`
- [x] Preguntas de entrevista de cierre

### Clarify ✅
- [x] `/speckit-clarify`: 4 marcadores resueltos en 3 preguntas (sesión 2026-10-26)
- [x] CA-001-26 y CA-001-27 agregados; checklist en 16/16
- [x] Commit `0a039ce docs(001): resolver clarificaciones de autenticación`
- [x] Preguntas de entrevista de cierre

### Plan ✅
- [x] `/speckit-plan` → `plan.md`, `research.md`, `data-model.md`, `contracts/openapi.yaml`, `quickstart.md`
- [x] Constitution Check aprobado antes y después del diseño
- [x] Cambios de spec descubiertos en el Plan (aplicados a la spec **antes** de cerrar el plan):
  - FR-031: normalización NFC de la contraseña → CA-001-28
  - FR-032: tope de 1 024 bytes → CA-001-29
  - FR-016: los intentos rechazados durante una espera no cuentan como fallos → CA-001-30
  - FR-007: la longitud se mide en puntos de código Unicode tras NFC
  - FR-016/FR-018: el contador por email se reinicia tras **1 hora** sin fallos (antes 15 min, que coincidía con el tope y nunca acumulaba); CA-001-26 con bordes de 59 min y 1 h
- [x] Commits `1e2d459`, `49338df` (spec) y `13b5162 docs(001): plan de autenticación`, con push
- [ ] Preguntas de entrevista de cierre — **omitidas por decisión de Raúl** (retomar si se quiere)

### Siguiente
- [ ] Teoría de Tasks
- [ ] `/speckit-tasks` → `tasks.md`
- [ ] Implement → Verify

## Decisiones técnicas del Plan (feature 001)

- **Hash:** argon2id (19 MiB, t=2, p=1, OWASP). Se descartó bcrypt porque trunca a 72 bytes y violaría FR-011.
- **Contraseñas comunes:** lista local (SecLists, MIT, filtrada a ≥12 caracteres) cargada en memoria. Se descartó la API de HIBP por ser un servicio externo en tiempo de ejecución. Limitación conocida: cubre contraseñas comunes, no todas las filtradas.
- **Sesión:** JWT de acceso HS256 de 15 min y token de renovación opaco, con hash SHA-256 en la BD. Rotación con detección de reutilización y gracia de 30 s. El guard consulta la sesión en la BD en cada solicitud para que el logout sea inmediato (SC-005); se acepta perder la ventaja de no tener estado.
- **Límites:** contadores en Postgres con claves HMAC del email y de la IP (minimización de datos). La IP se toma de `CF-Connecting-IP`, e IPv6 se agrupa por /64. Se usa `pg_advisory_xact_lock` por email.
- **Tiempo:** `Clock` inyectado; sin `@default(now())` en campos con lógica de negocio.
- **Longitud:** `[...pw.normalize('NFC')].length` en la app y en la API, nunca `.length`.
- **Ingreso:** si el email no existe, argon2 se verifica contra un hash ficticio para igualar tiempos (SC-006).
- **Registro:** el error P2002 de Prisma se traduce a `409 EMAIL_ALREADY_REGISTERED` (CA-001-22).
- **Maestro** no corre en el CI (desviación justificada de un DEBERÍA); se ejecuta en local durante Verify.

## Decisiones de Clarify (feature 001)

- **FR-005:** el registro informa explícitamente que el email ya existe. Es un riesgo aceptado, mitigado con FR-019, y se revisa al incorporar la verificación del email.
- **FR-015:** el ingreso no revela si un email existe.
- **FR-016/017/018:** límite por email con espera creciente (1, 2, 4, 8, hasta un tope de 15 min). El contador se reinicia con un ingreso exitoso o tras 1 h sin fallos. Límite por origen: 20 fallos en 15 min → espera de 15 min.
- **FR-019:** 5 solicitudes de registro por origen por hora.
- **FR-021:** la sesión dura 30 días desde el último uso con el servidor.

## Decisiones tomadas (generales)

- Montos en centavos de COP (`BIGINT`); nunca `float`.
- Fechas límite `DATE` en `America/Bogota`; marcas de tiempo técnicas en UTC.
- Cobertura mínima del 80 % en servicios de dominio; al menos un test por criterio de aceptación.
- Código en inglés; specs, comentarios y commits en español; Conventional Commits.
- Desde el Plan, los IDs FR y CA quedan congelados: los nuevos toman el siguiente número libre.
- Sin servicios de terceros de pago en esta etapa de práctica.

## Roadmap de features (Proyecto 1)

000 constitución (hecha) · 001 autenticación (en curso: Tasks) · 002 crear pago · 003 listar pagos · 004 cambiar estado · 005 eliminar pago · 006 eliminar cuenta

**Candidatas para la v1.1 (OpenSpec):** verificación de email (reabre FR-005), recordatorios, editar pagos.

## Aprendizajes

- Divide un requisito EARS cuando una parte puede cumplirse y la otra no.
- Fuga por comportamiento: ocultar un mensaje no sirve si el comportamiento ya revela la respuesta.
- Trazabilidad bidireccional: todo CA tiene test y todo FR tiene CA.
- El diseño revela huecos de la spec: se corrigen **primero en la spec** y en un commit separado.
- Los parámetros de una spec se revisan juntos (el reinicio de 15 min coincidía con el tope de 15 min).
- Análisis de valores límite: probar a ambos lados del umbral (59 min y 1 h), o el test no detecta una regresión.
- Las Clarifications se anotan, no se reescriben: son un registro de decisiones.
