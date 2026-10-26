# STATUS — pagos-recurrentes

**Última actualización:** 2026-10-26
**Fase actual:** Fase 1 — feature `001-autenticacion` — Clarify cerrado; sigue **Plan**

## Fase 0 — Preparación ✅

- [x] Proyecto "SDD Lab: Pagos y Gastos" creado en Claude
- [x] Repo público `pagos-recurrentes`
- [x] Spec Kit v1.0.1 instalado (`--integration claude`, scripts `sh`)
- [x] Constitución v1.0.0 ratificada (8 principios) en `.specify/memory/constitution.md`
- [x] `CLAUDE.md` aprobado; `.gitignore` con protección de secretos
- [x] Ramas `main` y `develop`

## Fase 1 — Feature 001-autenticacion (en curso)

### Specify ✅
- [x] Teoría de Specify
- [x] `/speckit-specify` → `specs/001-autenticacion/spec.md`
- [x] Dos rondas de revisión: EARS divididos (FR-007, FR-019, FR-028), SC-007 eliminado, nota de IDs congelados
- [x] Commit `docs(001): spec inicial de autenticación` (539b30b) y push de la rama
- [x] Preguntas de entrevista de cierre

### Clarify ✅
- [x] Teoría de Clarify
- [x] `/speckit-clarify`: 4 marcadores resueltos en 3 preguntas (sesión 2026-10-26)
- [x] CA-001-26 (reinicio del contador por email) y CA-001-27 (límite por origen) agregados
- [x] Checklist `checklists/requirements.md` en 16/16, con historial de revisiones
- [x] Commit `docs(001): resolver clarificaciones de autenticación` y push
- [x] Preguntas de entrevista de cierre

### Siguiente
- [ ] Teoría de Plan
- [ ] `/speckit-plan` (incluye el Constitution Check)
- [ ] Tasks → Implement → Verify

## Decisiones de Clarify (feature 001)

- **FR-005:** el registro informa explícitamente "Este email ya tiene una cuenta. Ingresa con él." Es un riesgo aceptado: el autoingreso tras registrarse ya revela si el email existe. Se mitiga con FR-019 y se revisa cuando entre la verificación del email.
- **FR-015:** el ingreso sigue sin revelar si un email existe.
- **FR-016/017/018:** límite por email, exista o no la cuenta. Desde el 5.º fallo seguido hay una espera de 1 min que se duplica hasta un tope de 15 min. El contador se reinicia con un ingreso exitoso o tras 15 min sin fallos. Límite por origen: 20 fallos en 15 min → espera de 15 min. Durante la espera se rechaza incluso con la contraseña correcta.
- **FR-019:** 5 solicitudes de registro por origen por hora.
- **FR-021:** la sesión dura 30 días desde el último uso. "Uso" significa que la app se comunica con el servidor con una sesión válida; abrirla sin conexión no renueva el plazo.

## Notas para el Plan (el "cómo", no van en la spec)

- **Origen detrás de Cloudflare Tunnel:** leer `CF-Connecting-IP`. Si no, todos los usuarios cuentan como un solo origen.
- **Dónde se guardan los contadores** de intentos (Postgres frente a memoria) y un reloj simulado para los tests de tiempo.
- **Rotación del token de renovación** con detección de reutilización, para compensar la expiración que se renueva con el uso (FR-021).
- **Algoritmo de hash de la contraseña** (FR-028).

## Decisiones tomadas (generales)

- Montos en centavos de COP (`BIGINT`), hasta 2 decimales de entrada; nunca `float`.
- Fechas límite `DATE` en `America/Bogota`; marcas de tiempo técnicas en UTC.
- Cobertura mínima del 80 % en servicios de dominio; al menos un test por criterio de aceptación.
- Código en inglés; specs, comentarios y commits en español; Conventional Commits.
- `plan-template.md` no se modifica: Spec Kit llena el Constitution Check al ejecutar `/speckit-plan`.
- El esqueleto del monorepo y el CI mínimo son las primeras tareas del plan de la 001.
- Contraseña: mínimo 12 caracteres y máximo 128, sin reglas de composición. Se rechazan las contraseñas comunes, filtradas o iguales al email.
- Sesiones múltiples; cerrar sesión afecta solo al dispositivo actual.
- Desde la fase Plan, los IDs FR y CA quedan congelados.
- Sin servicios de terceros de pago en esta etapa de práctica.

## Roadmap de features (Proyecto 1)

000 constitución (hecha) · 001 autenticación (en curso: Plan) · 002 crear pago · 003 listar pagos · 004 cambiar estado · 005 eliminar pago · 006 eliminar cuenta

**Candidatas para la v1.1 (OpenSpec):** verificación de email (reabre FR-005), recordatorios, editar pagos.
**Ideas para la 002 y la 003:** "etiqueta" del pago y vista del "mes actual".

## Aprendizajes

- Los comandos de Spec Kit v1.x son skills con guion: `/speckit-specify`, `/speckit-clarify`.
- Divide un requisito EARS cuando una parte puede cumplirse y la otra no. El largo no es el criterio.
- Fuga por comportamiento: ocultar un mensaje no sirve si el comportamiento del sistema ya revela la respuesta.
- Trazabilidad bidireccional: CA → test no basta; también todo FR → CA. El agente detectó que el límite por origen no tenía CA.
- Clarify no busca dejar todo "Clear": resuelve lo que cambia el plan o los tests, y lo de bajo impacto se difiere de forma documentada.
