# Constitución — pagos-recurrentes

> Reglas permanentes del proyecto. Toda spec, plan, tarea y PR DEBE cumplirlas.
> "DEBE" = obligatorio, sin excepción. "DEBERÍA" = esperado; desviarse requiere justificación escrita en el plan.

## Principios fundamentales

### I. La spec manda (spec-anchored)
- Ninguna línea de código de producto se escribe sin `spec.md` y `plan.md` aprobados.
- Si durante la implementación algo contradice la spec, se DEBE detener el trabajo y actualizar la spec primero.
- Al cerrar una feature, la spec DEBE reflejar lo que realmente quedó construido.

*Razón:* la spec es la fuente de verdad; el código se deriva de ella, no al revés.

### II. Trazabilidad de punta a punta
- Cada criterio de aceptación tiene un ID único con formato `CA-NNN-YY` (NNN = feature, YY = criterio).
- Cada feature vive en `specs/NNN-nombre/` y se desarrolla en una rama con el mismo nombre.
- Cada PR DEBE enlazar su `spec.md` y listar los IDs de los criterios que cubre.

*Razón:* cualquier persona debe poder seguir el camino spec → tarea → test → PR.

### III. Tests como contrato
- Todo criterio de aceptación DEBE tener al menos un test automatizado cuyo nombre incluya su ID.
- Los servicios de dominio de la API DEBEN mantener una cobertura mínima del 80 %.
- Toda pantalla de la app con criterios de aceptación DEBE tener tests con React Native Testing Library.
- Los flujos críticos (registro, ingreso, crear pago) DEBERÍAN tener un test e2e con Maestro.
- Ningún PR se fusiona con CI en rojo.

*Razón:* un criterio sin test es una promesa sin verificar.

### IV. Aislamiento de datos por usuario
- Toda consulta a datos de un usuario DEBE filtrar por el `userId` obtenido del token autenticado, nunca por un valor enviado por el cliente.
- Acceder a un recurso de otro usuario DEBE responder igual que si no existiera (404), sin revelar su existencia.

*Razón:* la fuga de datos entre usuarios es el fallo más grave posible en una app financiera.

### V. Seguridad por defecto
- Las contraseñas DEBEN almacenarse con un hash adaptativo (argon2id o bcrypt); nunca en texto plano ni con cifrado reversible.
- La autenticación usa tokens de acceso de vida corta y tokens de renovación revocables.
- Los endpoints de autenticación DEBEN tener límite de intentos (rate limiting).
- Toda entrada a la API DEBE validarse con DTOs y rechazar campos no declarados (whitelist).
- Los secretos viven solo en variables de entorno; ningún secreto, token ni `.env` real entra al repositorio.
- Los logs NO DEBEN contener contraseñas, tokens ni datos personales completos.
- En el móvil, los tokens se guardan en almacenamiento seguro (`expo-secure-store`), nunca en `AsyncStorage`.
- El CI DEBE revisar dependencias vulnerables; una vulnerabilidad alta o crítica bloquea el merge.
- La API expuesta públicamente solo se sirve por HTTPS.

*Razón:* el repositorio es público y la app maneja información financiera.

### VI. Dinero y fechas sin ambigüedad
- Los montos DEBEN guardarse como enteros en la unidad menor de la moneda (centavos de COP) en columnas `BIGINT`. Nunca como `float`.
- La moneda de la v1 es COP; el usuario puede ingresar hasta 2 decimales.
- La conversión entre texto y unidad menor ocurre solo en los bordes: la entrada del formulario y la presentación en pantalla.
- Los montos se muestran con formato `es-CO` (por ejemplo, $452.318,47). Los decimales se omiten cuando son cero.
- Las fechas límite son fechas sin hora (`DATE`), interpretadas en `America/Bogota`.
- Las marcas de tiempo técnicas (creación, actualización) se guardan en UTC.

*Razón:* los errores de redondeo y de zona horaria son silenciosos y se acumulan.

### VII. Simplicidad
- Se construye solo lo que pide la spec aprobada (YAGNI). Toda abstracción extra DEBE justificarse en el plan.
- La API se organiza en módulos de NestJS por dominio. La lógica de negocio va en servicios, no en controladores.
- Los cambios de esquema de base de datos se hacen solo con migraciones de Prisma versionadas; nunca a mano.
- Los errores de la API siguen un formato único y consistente en toda la aplicación.

*Razón:* menos piezas significa menos cosas que especificar, probar y mantener.

### VIII. Staging antes que producción
- Todo cambio pasa por staging antes de llegar a producción.
- Una migración solo llega a producción si ya se aplicó con éxito en staging.
- Producción se despliega solo desde un tag de versión en `main`, con aprobación manual.
- Cada ambiente tiene su propia base de datos y sus propios secretos.

*Razón:* producción tiene datos reales y usuarios reales.

## Restricciones técnicas

| Capa | Tecnología |
|---|---|
| API | NestJS + Prisma + PostgreSQL |
| App móvil | React Native + Expo + TypeScript, builds con EAS |
| Estado y datos (app) | Zustand + TanStack Query |
| Tests | Jest + Supertest (API), Jest + RNTL (app), Maestro (e2e) |
| Infraestructura | Docker Compose, GitHub Actions, GHCR, Cloudflare Tunnel, Tailscale |

Cambiar una tecnología de esta tabla requiere un ADR en `docs/decisions/` y una enmienda a esta constitución.

## Flujo de desarrollo

- **Ciclo por feature:** Specify → Clarify → Plan → Tasks → Implement → Verify → Deploy, con aprobación humana entre fases.
- **Idioma:** identificadores de código y rutas de la API en inglés; specs, comentarios, documentación y mensajes de commit en español.
- **Commits:** Conventional Commits con la feature como alcance (por ejemplo, `feat(002): validar monto positivo`).
- **Ramas:** `main` (producción), `develop` (staging) y `NNN-nombre` (feature, sale de `develop`).
- **Revisión:** todo diff generado por un agente es revisado por una persona antes del merge.

## Gobernanza

- Esta constitución prevalece sobre cualquier otra guía del repositorio, incluido `CLAUDE.md`.
- Todo `plan.md` DEBE incluir una sección "Constitution Check" que verifique cada principio. Si un principio se incumple, se justifica ahí o se corrige el plan.
- Las enmiendas se hacen por PR, con versionado semántico:
  - **MAJOR:** se elimina o redefine un principio.
  - **MINOR:** se agrega un principio o se amplía uno existente.
  - **PATCH:** cambios de redacción sin cambio de significado.

**Versión:** 1.0.0 | **Ratificada:** 2026-10-25 | **Última enmienda:** 2026-10-25
