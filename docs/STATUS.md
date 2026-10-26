# STATUS — pagos-recurrentes

**Última actualización:** 2026-10-26
**Fase actual:** Fase 1 — feature `001-autenticacion` — Tasks y Analyze cerrados; sigue **Implement** (T001)

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

### Tasks ✅
- [x] `/speckit-tasks` → `tasks.md`: 87 tareas (Setup 10 · Foundational 11 · US1 29 · US2 25 · US3 5 · Polish 7), 50 en paralelo
- [x] Las primeras tareas son el esqueleto del monorepo y el CI mínimo (T001–T010)
- [x] Una tarea de test por CA, nombrada con su ID y anterior a su implementación (un archivo e2e por CA)
- [x] FR-012 y FR-030 con tarea de revisión (T082, T083), no de test
- [x] Dependencias (📦) y confirmaciones (⚠ migraciones, descarga de SecLists, `CLAUDE.md`) marcadas por tarea
- [x] Cambio de spec descubierto en Tasks: **FR-033** (reutilizar una credencial de renovación fuera de la gracia de 30 s cierra la sesión) → CA-001-31

### Analyze ✅
- [x] Primera pasada: 0 CRITICAL, 2 HIGH, 8 MEDIUM, 4 LOW
  - HIGH: el test de tiempos de SC-006 chocaba con el límite por email (FR-016), y la gracia de 30 s borraba una fila referenciada por FK (`replacedById`)
- [x] Todo corregido: spec (A1: la lista cubre las filtradas más frecuentes; L1: estado Approved), plan, research, data-model y tasks
- [x] Segunda pasada: 33 FR + 6 SC, 31 CA, 87 tareas, **100 % de cobertura, 0 issues**
- [x] Commits `2b33ccb` (spec), `f1a1680` (plan) y `94df942 docs(001): tareas de autenticación`, sin push
- [ ] Desviación abierta: la plantilla de PR va dentro de T009 y no como tarea propia (para no renumerar)

### Siguiente
- [ ] `/speckit-implement`, una tarea a la vez con diff revisado, empezando por T001
- [ ] Verify: Maestro (T084) y medición de SC-006 (T087) en local

## Decisiones técnicas del Plan (feature 001)

- **Hash:** argon2id (19 MiB, t=2, p=1, OWASP). Se descartó bcrypt porque trunca a 72 bytes y violaría FR-011.
- **Contraseñas comunes:** lista local (SecLists, MIT, filtrada a ≥12 caracteres) cargada en memoria. Se descartó la API de HIBP por ser un servicio externo en tiempo de ejecución. Limitación conocida: cubre contraseñas comunes, no todas las filtradas.
- **Sesión:** JWT de acceso HS256 de 15 min y token de renovación opaco, con hash SHA-256 en la BD. Rotación con detección de reutilización y gracia de 30 s (formalizada como FR-033 / CA-001-31). El guard consulta la sesión en la BD en cada solicitud para que el logout sea inmediato (SC-005); se acepta perder la ventaja de no tener estado.
- **Límites:** contadores en Postgres con claves HMAC del email y de la IP (minimización de datos). La IP se toma de `CF-Connecting-IP`, e IPv6 se agrupa por /64. Se usa `pg_advisory_xact_lock` por email.
- **Tiempo:** `Clock` inyectado; sin `@default(now())` en campos con lógica de negocio.
- **Longitud:** `[...pw.normalize('NFC')].length` en la app y en la API, nunca `.length`.
- **Ingreso:** si el email no existe, argon2 se verifica contra un hash ficticio para igualar tiempos (SC-006).
- **Registro:** el error P2002 de Prisma se traduce a `409 EMAIL_ALREADY_REGISTERED` (CA-001-22).
- **Ejecución local en Verify** (no en CI): Maestro (desviación justificada de un DEBERÍA) y la medición estadística de tiempos de SC-006. En CI, SC-006 se cubre de forma determinista: un spy confirma que argon2 se ejecuta contra el hash ficticio.
- **Cobertura:** el 80 % de `*.service.ts` se mide sobre unit + e2e juntos.
- **Rotación:** `replacedById` con `onDelete: SetNull`; en la gracia el orden es crear → reasignar → borrar.

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

000 constitución (hecha) · 001 autenticación (en curso: Implement) · 002 crear pago · 003 listar pagos · 004 cambiar estado · 005 eliminar pago · 006 eliminar cuenta

**Candidatas para la v1.1 (OpenSpec):** verificación de email (reabre FR-005), recordatorios, editar pagos.

## Aprendizajes

- Divide un requisito EARS cuando una parte puede cumplirse y la otra no.
- Fuga por comportamiento: ocultar un mensaje no sirve si el comportamiento ya revela la respuesta.
- Trazabilidad bidireccional: todo CA tiene test y todo FR tiene CA.
- El diseño revela huecos de la spec: se corrigen **primero en la spec** y en un commit separado.
- Los parámetros de una spec se revisan juntos (el reinicio de 15 min coincidía con el tope de 15 min).
- Análisis de valores límite: probar a ambos lados del umbral (59 min y 1 h), o el test no detecta una regresión.
- Las Clarifications se anotan, no se reescriben: son un registro de decisiones.
- Un comportamiento de diseño sin CA (la detección de reutilización) es un hueco de la spec: se formaliza como FR/CA antes de implementar.
- Analyze encuentra tests que chocan con otros requisitos: los límites de intentos invalidaban la medición de tiempos.
- Los tests estadísticos de tiempo son intermitentes en CI: en CI se prueba la causa (mismo trabajo en ambos caminos) y el efecto (tiempos) se mide en local.
