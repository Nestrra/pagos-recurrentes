# Specification Quality Checklist: Autenticación (registro, ingreso y cierre de sesión)

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-10-25
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Notes

- No quedan marcadores [NEEDS CLARIFICATION]. En la spec inicial hubo 4 (FR-005, FR-016, FR-019 y FR-021), uno más que el máximo de 3 que sugiere la plantilla, por decisión explícita del usuario. Se resolvieron en `/speckit-clarify` el 2026-10-26.
- Todos los FR son testables con umbrales concretos.
- Todo FR tiene un CA asociado, excepto FR-012 (pegar contraseña o usar gestor) y FR-030 (almacenamiento seguro en el teléfono), marcados como "verificado por revisión".
- Revisión 2026-10-25:
  - Se dividieron FR-007, FR-019 (el anterior) y FR-028 en un comportamiento por requisito y se renumeraron los FR (ahora FR-001 a FR-030).
  - Se agregaron CA-001-21 a CA-001-25.
  - Se eliminó SC-007.
  - Se agregó la sección "Decisiones aprobadas".
- Revisión 2026-10-26 (`/speckit-clarify`, ver la sección "Clarifications" de la spec):
  - FR-005: el registro informa explícitamente que el email ya existe. Es una decisión consciente, mitigada con FR-019.
  - FR-016 y FR-018: límite por email con espera creciente (se reinicia con un ingreso exitoso o tras 15 min sin fallos) y límite por origen.
  - FR-019: 5 solicitudes de registro por origen por hora.
  - FR-021: la sesión dura 30 días desde el último uso con el servidor; abrir la app sin conexión no renueva el plazo.
  - Se agregaron CA-001-26 (reinicio del contador por email) y CA-001-27 (límite por origen).
- Revisión 2026-10-26 (fase Plan, `/speckit-plan`; IDs congelados, se usan los siguientes números libres):
  - FR-031 (nuevo): normalización Unicode NFC de la contraseña al registrar y al ingresar. CA-001-28.
  - FR-032 (nuevo): tope de 1 024 bytes de la contraseña; en el registro se rechaza como demasiado larga y en el ingreso como intento fallido genérico. CA-001-29.
  - FR-007: la longitud de la contraseña (mínimo y máximo) se cuenta en puntos de código Unicode después de NFC; reemplaza "tal como la percibe la persona". Edge Case "Caracteres especiales" alineado.
  - FR-011: se aclara que la normalización NFC no se considera alterar la contraseña.
  - FR-016: los intentos rechazados durante una espera vigente no cuentan como fallos (ni por email ni por origen) y no prolongan la espera. CA-001-30.
  - Se agregaron a "Decisiones aprobadas" las cuatro decisiones de la fase Plan y dos Edge Cases (misma contraseña con distinta codificación; contraseña de más de 1 024 bytes).
  - La checklist sigue en 16/16: todo FR nuevo tiene CA y ningún cambio introduce detalles de implementación.
- Items marked incomplete require spec updates before `/speckit-clarify` or `/speckit-plan`
