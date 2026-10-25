# CLAUDE.md — pagos-recurrentes

Manual de operación para Claude Code en este repositorio.
Las **reglas** del proyecto NO están aquí: viven en la constitución. Este archivo explica **cómo trabajar**.

## Fuentes de verdad (en este orden)

1. `.specify/memory/constitution.md`: reglas permanentes. Prevalece sobre este archivo.
2. `specs/NNN-nombre/spec.md`: qué construir en cada feature.
3. `specs/NNN-nombre/plan.md` y `tasks.md`: cómo construirlo y en qué orden.

Si algo de este archivo contradice la constitución, manda la constitución.

## Flujo obligatorio

Ciclo por feature, con aprobación humana entre cada fase:
`/speckit-specify` → `/speckit-clarify` → `/speckit-plan` → `/speckit-tasks` → `/speckit-implement` → Verify → Deploy.

- No escribas código de producto sin `spec.md` y `plan.md` aprobados.
- Si al implementar algo contradice la spec: **detente**, explica el conflicto y propone el cambio a la spec. No improvises.
- Implementa **una tarea de `tasks.md` a la vez** y muestra el diff antes de seguir.

## Estado actual

- Fase 0: constitución v1.0.0 ratificada. Aún no existe código ni carpetas `apps/`.
- El progreso se registra en `docs/STATUS.md`.
- Las secciones "Estructura" y "Comandos" se completan cuando exista el esqueleto del monorepo. Hasta entonces no inventes comandos.

## Estructura (objetivo)

```
apps/api/      NestJS + Prisma + PostgreSQL
apps/mobile/   React Native + Expo + TypeScript
specs/         una carpeta por feature (spec, plan, tasks, checklist)
deploy/        compose de staging y producción
docs/          STATUS.md y decisiones (ADRs)
```

## Comandos

_Pendiente: se documentan al crear el esqueleto (instalar, test, lint, migraciones, levantar local)._

## Cómo escribir código aquí

- Tests: el nombre incluye el ID del criterio. Ejemplo: `it('CA-002-02: rechaza monto 0 o negativo', ...)`.
- Comentarios en español y abundantes; identificadores en inglés.
- Commits: `tipo(NNN): descripción en español`. Ejemplo: `feat(002): validar monto positivo`.
- Diffs pequeños y enfocados en la tarea actual. Nada de refactors no pedidos.

## Límites (pide confirmación antes de…)

- Instalar o actualizar dependencias.
- Crear o aplicar migraciones de base de datos.
- Ejecutar cualquier comando contra algo que no sea el entorno local.
- Modificar la constitución, una spec aprobada o este archivo.

Nunca leas, crees ni modifiques archivos `.env` reales; usa solo `.env.example`.
