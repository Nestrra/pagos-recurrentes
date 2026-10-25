# STATUS — pagos-recurrentes

**Última actualización:** 2026-10-25
**Fase actual:** Fase 0 cerrada → siguiente: Fase 1 (feature 001 autenticación)

## Fase 0 — Preparación ✅

- [x] Proyecto "SDD Lab: Pagos y Gastos" creado en Claude
- [x] Repo público `pagos-recurrentes` creado
- [x] Spec Kit v1.0.1 instalado (`--integration claude`, scripts `sh`)
- [x] Constitución v1.0.0 ratificada (8 principios) en `.specify/memory/constitution.md`
- [x] `CLAUDE.md` aprobado (manual operativo; enlaza la constitución, no la duplica)
- [x] `.gitignore` con protección de secretos (principio V)
- [x] Ramas `main` y `develop` creadas

## Decisiones tomadas

- Montos en centavos de COP (`BIGINT`), hasta 2 decimales de entrada. Nunca `float`.
- Fechas límite como `DATE` en `America/Bogota`; marcas de tiempo técnicas en UTC.
- Cobertura mínima del 80 % en servicios de dominio; un test o más por cada criterio de aceptación.
- Código en inglés; specs, comentarios y commits en español; Conventional Commits.
- La sección Constitution Check de `plan-template.md` NO se modifica: Spec Kit la llena al correr `/speckit-plan`.

## Aprendizajes

- Los comandos de Spec Kit v1.x son skills con guion: `/speckit-specify`, no `/speckit.specify`.
- El agente detuvo una instrucción que contradecía el Scope Guard de su propia herramienta y pidió decisión humana.

## Siguiente paso

Fase 1, feature `001-autenticacion`: teoría de Specify y luego `/speckit-specify`.
El esqueleto del monorepo y el CI mínimo se crean como tareas del plan de la 001, para que también pasen por el ciclo (principio I).

## Pendientes abiertos

- Ninguno.
