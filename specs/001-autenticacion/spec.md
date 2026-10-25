# Feature Specification: Autenticación (registro, ingreso y cierre de sesión)

**Feature Branch**: `001-autenticacion`

**Created**: 2026-10-25

**Status**: Draft

**Input**: User description: "La app permite a una persona controlar cada mes sus pagos recurrentes: saber cuáles ya pagó y cuáles tiene pendientes. Cada persona necesita una cuenta propia para que sus pagos estén separados de los de otros usuarios y nadie más pueda verlos. En esta feature el usuario puede: registrarse con email y contraseña, ingresar y cerrar sesión. Reglas: el email debe tener un formato válido y no puede estar registrado previamente. La contraseña debe tener mínimo 12 caracteres y permitir al menos 64; no se exigen combinaciones de mayúsculas, números o símbolos; se rechazan contraseñas comunes o filtradas. Decisión consciente: se usan 12 caracteres en lugar de los 15 que recomienda NIST SP 800-63B rev. 4, por comodidad en móvil, compensado con el bloqueo de contraseñas comunes y el límite de intentos. Por definir: qué pasa tras varios intentos fallidos de ingreso y cuánto tiempo permanece abierta la sesión. Fuera de alcance: ingreso con Google o Apple, recuperación de contraseña, verificación del email y eliminación de la cuenta (feature 006)."

## Contexto

La app sirve para que una persona controle cada mes sus pagos recurrentes (cuáles ya pagó y cuáles tiene pendientes). Esa información es financiera y personal, así que cada persona necesita una **cuenta propia**: sus datos deben quedar separados de los de cualquier otro usuario y nadie más debe poder verlos.

Esta feature entrega la puerta de entrada: crear la cuenta, ingresar y cerrar sesión. Todas las features posteriores (pagos, historial, etc.) dependen de ella.

### Decisión consciente sobre la longitud mínima de contraseña

NIST SP 800-63B rev. 4 recomienda un mínimo de 15 caracteres cuando la contraseña es el único factor. En este proyecto se fija el mínimo en **12 caracteres** por comodidad al escribir en el teclado del móvil. El riesgo se compensa con dos controles obligatorios de esta misma spec:

1. Rechazo de contraseñas comunes o filtradas (FR-009).
2. Límite de intentos fallidos de ingreso (FR-016 a FR-018).

Si alguno de estos dos controles se retira en el futuro, esta decisión debe revisarse.

### Decisiones aprobadas

Decisiones tomadas durante la redacción de la spec y aprobadas el 2026-10-25:

- **Máximo de 128 caracteres en la contraseña** (FR-007, FR-008): el requisito original pide "permitir al menos 64"; se fija 128 como máximo para dar margen a frases largas y a gestores de contraseñas, y para acotar el esfuerzo de verificación.
- **Sesiones múltiples con cierre por dispositivo** (FR-027): una misma cuenta puede tener sesión abierta en varios teléfonos a la vez; cerrar sesión afecta solo al dispositivo actual. No existe "cerrar sesión en todos los dispositivos" en esta feature.
- **Rechazo de contraseña igual al email** (FR-009): una contraseña idéntica al email de la persona se rechaza por ser demasiado predecible.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Crear una cuenta propia (Priority: P1)

Una persona que instala la app por primera vez crea su cuenta con su email y una contraseña. Al terminar queda dentro de la app, con un espacio propio que nadie más puede ver.

**Why this priority**: sin cuenta no existe ningún otro flujo; es el primer paso de cualquier usuario y la base del aislamiento de datos.

**Independent Test**: se puede probar completo registrando un email nuevo con una contraseña válida y comprobando que la persona queda con sesión abierta en su propia cuenta; también probando cada regla de rechazo (email inválido, email repetido, contraseña corta, larga o común).

**Acceptance Scenarios**:

1. **CA-001-01** — **Dado** que el email `ana@ejemplo.com` no está registrado, **Cuando** la persona se registra con ese email y una contraseña válida de 12 caracteres o más que no es común ni filtrada, **Entonces** se crea su cuenta y queda con sesión abierta.
2. **CA-001-02** — **Dado** que la persona está en el formulario de registro, **Cuando** escribe un email sin formato válido (por ejemplo `ana@`, `ana.ejemplo.com` o vacío), **Entonces** el registro se rechaza con un mensaje que indica que el email no es válido y no se crea ninguna cuenta.
3. **CA-001-03** — **Dado** que `ana@ejemplo.com` ya tiene una cuenta, **Cuando** alguien intenta registrarse con ese email (o con una variante que solo difiere en mayúsculas o espacios al inicio o final, como ` Ana@Ejemplo.com `), **Entonces** el registro se rechaza y no se crea una segunda cuenta. [El texto exacto del mensaje depende de la clarificación de FR-005.]
4. **CA-001-04** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña de 11 caracteres o menos, **Entonces** el registro se rechaza con un mensaje que indica el mínimo de 12 caracteres.
5. **CA-001-05** — **Dado** que la persona está en el formulario de registro, **Cuando** usa contraseñas válidas de exactamente 12, 64 y 128 caracteres, **Entonces** las tres se aceptan.
6. **CA-001-06** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña de 12 o más caracteres formada solo por letras minúsculas (sin mayúsculas, números ni símbolos) que no es común ni filtrada, **Entonces** se acepta.
7. **CA-001-07** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña que aparece en la lista de contraseñas comunes o filtradas (por ejemplo `contraseña123` o `123456789012`), **Entonces** el registro se rechaza con un mensaje que indica que esa contraseña es demasiado conocida y que elija otra.
8. **CA-001-08** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña de más de 128 caracteres, **Entonces** el registro se rechaza con un mensaje que indica el máximo permitido.
9. **CA-001-09** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña con espacios, tildes, `ñ` o emojis que cumple la longitud y no es común, **Entonces** se acepta, y luego puede ingresar escribiendo exactamente la misma contraseña.
10. **CA-001-21** — **Dado** que la persona se registra con el email `ana@ejemplo.com`, **Cuando** usa como contraseña `ana@ejemplo.com`, **Entonces** el registro se rechaza con un mensaje que indica que la contraseña es demasiado predecible y no se crea la cuenta.
11. **CA-001-22** — **Dado** que el email `ana@ejemplo.com` no está registrado, **Cuando** llegan dos solicitudes de registro simultáneas con ese email, **Entonces** se crea exactamente una cuenta y la otra solicitud se rechaza como email ya registrado.
12. **CA-001-23** — **Dado** que desde un mismo origen se alcanzó el límite de solicitudes de registro de FR-019, **Cuando** se envía una nueva solicitud de registro desde ese origen, **Entonces** se rechaza sin crear la cuenta y se informa a la persona que espere antes de volver a intentar. [Umbral y ventana pendientes de clarificación.]

---

### User Story 2 - Ingresar a mi cuenta (Priority: P2)

Una persona que ya tiene cuenta (por ejemplo, tras reinstalar la app o desde otro teléfono) ingresa con su email y contraseña y ve solo su propia información.

**Why this priority**: es el flujo que más se repite en el uso diario; depende de que exista la cuenta (P1).

**Independent Test**: con una cuenta ya creada, ingresar con las credenciales correctas y comprobar el acceso; ingresar con credenciales incorrectas y comprobar el rechazo sin pistas sobre qué dato falló; superar el límite de intentos y comprobar la restricción.

**Acceptance Scenarios**:

1. **CA-001-10** — **Dado** que existe la cuenta `ana@ejemplo.com` y la persona no tiene sesión abierta, **Cuando** ingresa con su email y contraseña correctos, **Entonces** queda con sesión abierta en su cuenta.
2. **CA-001-11** — **Dado** que existe la cuenta `ana@ejemplo.com`, **Cuando** la persona ingresa con ` ANA@ejemplo.com ` (otras mayúsculas o espacios alrededor) y la contraseña correcta, **Entonces** ingresa a la misma cuenta.
3. **CA-001-12** — **Dado** que existe la cuenta `ana@ejemplo.com`, **Cuando** alguien intenta ingresar con la contraseña incorrecta, **Entonces** el ingreso se rechaza con un mensaje genérico ("email o contraseña incorrectos").
4. **CA-001-13** — **Dado** que `nadie@ejemplo.com` no está registrado, **Cuando** alguien intenta ingresar con ese email, **Entonces** recibe exactamente el mismo mensaje genérico que en CA-001-12, sin revelar si el email existe.
5. **CA-001-14** — **Dado** que se ha alcanzado el límite de intentos fallidos de ingreso definido en FR-016, **Cuando** se hace un nuevo intento (incluso con la contraseña correcta), **Entonces** el sistema aplica la restricción definida en FR-017 e informa a la persona qué puede hacer. [Umbral y efecto pendientes de clarificación.]
6. **CA-001-15** — **Dado** que dos personas tienen cuenta (`ana@ejemplo.com` y `beto@ejemplo.com`), **Cuando** Ana ingresa, **Entonces** solo puede ver y operar sobre los datos asociados a su cuenta, y cualquier intento de acceder a datos de Beto se trata como si esos datos no existieran.
7. **CA-001-16** — **Dado** que la persona ingresó y cerró la app sin cerrar sesión, **Cuando** vuelve a abrir la app dentro del tiempo de vigencia definido en FR-021, **Entonces** sigue con la sesión abierta sin volver a escribir sus credenciales.
8. **CA-001-17** — **Dado** que la sesión de la persona superó el tiempo de vigencia definido en FR-021, **Cuando** intenta usar la app, **Entonces** se le pide ingresar de nuevo. [Duración pendiente de clarificación.]

---

### User Story 3 - Cerrar sesión (Priority: P3)

Una persona cierra su sesión, por ejemplo antes de prestar el teléfono o al cambiar de dispositivo, para que nadie pueda ver sus pagos desde ese teléfono.

**Why this priority**: protege la privacidad en dispositivos compartidos, pero el producto ya es usable con registro e ingreso.

**Independent Test**: con sesión abierta, cerrar sesión y comprobar que la app vuelve a la pantalla de ingreso y que la sesión anterior ya no da acceso a ningún dato.

**Acceptance Scenarios**:

1. **CA-001-18** — **Dado** que la persona tiene sesión abierta, **Cuando** elige "Cerrar sesión", **Entonces** la app vuelve a la pantalla de ingreso y no muestra ningún dato de su cuenta.
2. **CA-001-19** — **Dado** que la persona cerró sesión, **Cuando** alguien intenta reutilizar la sesión anterior (por ejemplo, reabriendo la app o repitiendo una petición con la credencial de sesión anterior), **Entonces** el acceso se rechaza y se exige ingresar de nuevo.
3. **CA-001-20** — **Dado** que la persona tiene sesión abierta en dos teléfonos, **Cuando** cierra sesión en uno, **Entonces** la sesión del otro teléfono sigue abierta.

---

### Escenarios transversales (protección de credenciales)

Aplican a las tres historias.

**Acceptance Scenarios**:

1. **CA-001-24** — **Dado** que Ana se registró con una contraseña válida, **Cuando** se inspecciona la información almacenada de su cuenta, **Entonces** la contraseña no aparece en forma legible ni puede reconstruirse a partir de lo almacenado.
2. **CA-001-25** — **Dado** que se ejecutaron un registro, un ingreso exitoso, un ingreso fallido y un cierre de sesión, **Cuando** se revisan los registros de actividad del sistema, **Entonces** no contienen ninguna contraseña ni credencial de sesión.

---

### Edge Cases

- **Email con mayúsculas o espacios**: se recortan los espacios al inicio y al final y se compara sin distinguir mayúsculas; `Ana@Ejemplo.com` y `ana@ejemplo.com` son la misma cuenta (CA-001-03, CA-001-11).
- **Contraseña con espacios al inicio o al final**: se respeta tal cual; no se recorta ni se modifica, porque cambiarla haría que la persona no pudiera ingresar con lo que escribió.
- **Contraseña en el borde del límite**: 11 caracteres se rechaza; 12, 64 y 128 se aceptan; 129 se rechaza (CA-001-04, CA-001-05, CA-001-08).
- **Caracteres especiales**: tildes, `ñ`, emojis y espacios cuentan como caracteres válidos; la longitud se mide en caracteres visibles para la persona, no en bytes (CA-001-09).
- **Doble toque en "Registrarme"**: dos envíos casi simultáneos con el mismo email nuevo crean una sola cuenta; el segundo envío se trata como email ya registrado (CA-001-22).
- **Sin conexión**: si no hay conexión al registrarse, ingresar o cerrar sesión, la app informa que no hay conexión y no da por hecha la operación; los datos escritos en el formulario (excepto la contraseña) no se pierden.
- **Cerrar sesión sin conexión**: la app borra igualmente la sesión del teléfono y vuelve a la pantalla de ingreso; la invalidación en el servidor se completa al recuperar conexión o al expirar la sesión.
- **Pegar contraseña o usar gestor de contraseñas**: está permitido en registro e ingreso.
- **Contraseña igual al email**: se rechaza como contraseña demasiado predecible (CA-001-21).

## Requirements *(mandatory)*

Los requisitos funcionales usan notación **EARS** (Easy Approach to Requirements Syntax):

| Patrón | Plantilla |
|---|---|
| Ubicuo | El sistema DEBE … |
| Por evento | CUANDO \<evento\>, el sistema DEBE … |
| Por estado | MIENTRAS \<estado\>, el sistema DEBE … |
| No deseado | SI \<condición no deseada\>, ENTONCES el sistema DEBE … |
| Opcional | DONDE \<característica\>, el sistema DEBE … |

### Functional Requirements

> **Nota:** A partir de la fase Plan los IDs FR y CA quedan congelados; los nuevos toman el siguiente número libre y los eliminados se marcan como (retirado).

**Registro**

- **FR-001** *(por evento)*: CUANDO una persona envía el formulario de registro con un email válido no registrado y una contraseña válida, el sistema DEBE crear una cuenta asociada a ese email y abrir una sesión para ella. → CA-001-01
- **FR-002** *(ubicuo)*: El sistema DEBE normalizar el email recortando espacios al inicio y al final y tratándolo sin distinción de mayúsculas, tanto al registrar como al ingresar. → CA-001-03, CA-001-11
- **FR-003** *(no deseado)*: SI el email está vacío o no tiene un formato válido (una parte local, una `@` y un dominio con al menos un punto), ENTONCES el sistema DEBE rechazar el registro, indicar que el email no es válido y no crear la cuenta. → CA-001-02
- **FR-004** *(no deseado)*: SI el email normalizado ya pertenece a una cuenta existente, ENTONCES el sistema DEBE rechazar el registro sin crear ni modificar ninguna cuenta. → CA-001-03
- **FR-005** *(no deseado)*: SI el registro se rechaza porque el email ya existe, ENTONCES el sistema DEBE responder [NEEDS CLARIFICATION: ¿se le dice explícitamente a la persona que el email ya está registrado (más claro, pero permite averiguar qué emails tienen cuenta), o se usa una respuesta que no lo revela?]. → CA-001-03
- **FR-006** *(no deseado)*: SI la contraseña tiene menos de 12 caracteres, ENTONCES el sistema DEBE rechazar el registro indicando el mínimo de 12 caracteres. → CA-001-04
- **FR-007** *(ubicuo)*: El sistema DEBE aceptar contraseñas de 12 a 128 caracteres (cumpliendo así el requisito de permitir al menos 64), contando caracteres tal como los percibe la persona. → CA-001-05
- **FR-008** *(no deseado)*: SI la contraseña tiene más de 128 caracteres, ENTONCES el sistema DEBE rechazar el registro indicando el máximo de 128 caracteres. → CA-001-08
- **FR-009** *(no deseado)*: SI la contraseña aparece en la lista de contraseñas comunes o filtradas, o es igual al email de la persona, ENTONCES el sistema DEBE rechazar el registro indicando que la contraseña es demasiado conocida o predecible y pidiendo otra. → CA-001-07, CA-001-21
- **FR-010** *(ubicuo)*: El sistema NO DEBE exigir combinaciones de mayúsculas, minúsculas, números o símbolos en la contraseña. → CA-001-06
- **FR-011** *(ubicuo)*: El sistema DEBE aceptar en la contraseña cualquier carácter imprimible, incluidos espacios, tildes, `ñ` y emojis, y NO DEBE recortarla, truncarla ni alterarla. → CA-001-09
- **FR-012** *(ubicuo)*: El sistema DEBE permitir pegar texto y usar gestores de contraseñas en los campos de contraseña de registro e ingreso. → verificado por revisión
- **FR-013** *(ubicuo)*: El sistema DEBE garantizar que dos solicitudes de registro simultáneas con el mismo email produzcan como máximo una cuenta. → CA-001-22

**Ingreso**

- **FR-014** *(por evento)*: CUANDO una persona envía su email y su contraseña correctos, el sistema DEBE abrir una sesión en su cuenta. → CA-001-10, CA-001-11
- **FR-015** *(no deseado)*: SI el email no está registrado o la contraseña no corresponde, ENTONCES el sistema DEBE rechazar el ingreso con el mismo mensaje genérico en ambos casos, y en un tiempo de respuesta que no permita distinguir un caso del otro. → CA-001-12, CA-001-13
- **FR-016** *(ubicuo)*: El sistema DEBE contar los intentos fallidos de ingreso y aplicar un límite de [NEEDS CLARIFICATION: ¿cuántos intentos fallidos, en qué ventana de tiempo, contados por cuenta, por origen o ambos, y qué efecto tienen (espera temporal creciente, bloqueo temporal de la cuenta por N minutos, u otro)?]. → CA-001-14
- **FR-017** *(no deseado)*: SI se alcanza el límite de FR-016, ENTONCES el sistema DEBE rechazar los nuevos intentos de ingreso mientras dure la restricción, incluso con la contraseña correcta, e informar a la persona cuándo puede volver a intentar, sin revelar si el email existe. → CA-001-14
- **FR-018** *(por evento)*: CUANDO una persona ingresa con éxito, el sistema DEBE reiniciar el conteo de intentos fallidos asociado a su cuenta. → CA-001-14
- **FR-019** *(ubicuo)*: El sistema DEBE limitar también la frecuencia de solicitudes de registro desde un mismo origen, para impedir la creación masiva de cuentas y el sondeo de emails, con un límite de [NEEDS CLARIFICATION: ¿cuántas solicitudes de registro se permiten desde un mismo origen y en qué ventana de tiempo?]. → CA-001-23

**Sesión**

- **FR-020** *(por estado)*: MIENTRAS la sesión esté vigente, el sistema DEBE mantener a la persona dentro de su cuenta aunque cierre y reabra la app. → CA-001-16
- **FR-021** *(por evento)*: CUANDO se cumpla el tiempo de vigencia de la sesión, el sistema DEBE dar la sesión por expirada; la sesión DEBE dejar de ser válida tras [NEEDS CLARIFICATION: ¿cuánto tiempo permanece abierta la sesión? (por ejemplo, 30 días desde el último uso, 30 días fijos desde el ingreso, o cerrar al salir de la app)]. → CA-001-17
- **FR-022** *(no deseado)*: SI la sesión expiró o fue cerrada, ENTONCES el sistema DEBE rechazar cualquier acceso a datos de la cuenta y llevar a la persona a la pantalla de ingreso. → CA-001-17, CA-001-19

**Aislamiento de datos**

- **FR-023** *(ubicuo)*: El sistema DEBE asociar cada dato de usuario a una única cuenta y DEBE determinar la cuenta solo a partir de la sesión autenticada, nunca a partir de un identificador enviado por la app. → CA-001-15
- **FR-024** *(no deseado)*: SI una persona con sesión abierta intenta acceder a un dato de otra cuenta, ENTONCES el sistema DEBE responder igual que si ese dato no existiera, sin revelar su existencia. → CA-001-15
- **FR-025** *(no deseado)*: SI una solicitud a datos de usuario llega sin sesión válida, ENTONCES el sistema DEBE rechazarla sin devolver ningún dato. → CA-001-19

**Cierre de sesión**

- **FR-026** *(por evento)*: CUANDO la persona elige "Cerrar sesión", el sistema DEBE invalidar esa sesión de forma que no pueda reutilizarse, eliminarla del teléfono y mostrar la pantalla de ingreso. → CA-001-18, CA-001-19
- **FR-027** *(ubicuo)*: El cierre de sesión DEBE afectar solo a la sesión del dispositivo donde se ejecuta; las sesiones de otros dispositivos de la misma cuenta siguen vigentes. → CA-001-20

**Protección de credenciales**

- **FR-028** *(ubicuo)*: El sistema NO DEBE guardar ni mostrar la contraseña en forma legible o recuperable. → CA-001-24
- **FR-029** *(ubicuo)*: El sistema NO DEBE incluir contraseñas ni credenciales de sesión en los registros de actividad. → CA-001-25
- **FR-030** *(ubicuo)*: El sistema DEBE guardar las credenciales de sesión en el teléfono solo en almacenamiento seguro del sistema operativo. → verificado por revisión

### Key Entities *(include if feature involves data)*

- **Cuenta de usuario**: representa a una persona. Atributos: email normalizado (único), verificador de la contraseña (no la contraseña), fecha de creación. Es la dueña de todos los datos de usuario que se creen en features posteriores.
- **Sesión**: acceso abierto de una cuenta en un dispositivo concreto. Atributos: cuenta a la que pertenece, momento de inicio, momento de expiración, estado (vigente, cerrada, expirada). Una cuenta puede tener varias sesiones (una por dispositivo).
- **Registro de intentos fallidos**: conteo de intentos de ingreso fallidos en una ventana de tiempo, usado para aplicar el límite de FR-016. Su forma exacta depende de la clarificación de FR-016.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Una persona nueva completa el registro en menos de 2 minutos desde que abre el formulario (medido en prueba e2e).
- **SC-002**: Una persona registrada ingresa a su cuenta en menos de 30 segundos (medido en prueba e2e).
- **SC-003**: El 100 % de las contraseñas de una muestra de prueba tomada de la lista de contraseñas comunes o filtradas se rechaza en el registro.
- **SC-004**: En pruebas con dos o más cuentas, el 0 % de los intentos de una cuenta por ver o modificar datos de otra tiene éxito.
- **SC-005**: Tras cerrar sesión, el 100 % de los intentos de reutilizar esa sesión se rechaza.
- **SC-006**: Las respuestas de ingreso fallido para email inexistente y para contraseña incorrecta son indistinguibles en texto y no difieren de forma consistente en tiempo de respuesta.

## Assumptions

- **Registro abre sesión**: tras registrarse, la persona queda con sesión abierta directamente, sin un ingreso adicional.
- **Lista de contraseñas prohibidas**: se usa una lista reconocida de contraseñas comunes y filtradas, disponible sin enviar la contraseña de la persona a terceros en texto claro. La elección concreta de la lista se define en el plan.
- **Formato de email**: se valida el formato básico (parte local, `@`, dominio con punto). No se verifica que el buzón exista; la verificación del email está fuera de alcance.
- **Mensajes**: todos los mensajes al usuario están en español.
- **Plataforma**: la experiencia es la app móvil; no hay versión web en esta feature.
- **Sin cambio de contraseña ni de email**: modificar la contraseña o el email de una cuenta existente no forma parte de esta feature.

### Fuera de alcance

- Ingreso con Google o Apple.
- Recuperación de contraseña.
- Verificación del email.
- Cambio de contraseña o de email.
- Eliminación de la cuenta (feature 006).
- Segundo factor de autenticación.

### Dependencias

- Ninguna feature previa. Todas las features posteriores dependen de esta para identificar al usuario y aislar sus datos.
