# Feature Specification: Autenticación (registro, ingreso y cierre de sesión)

**Feature Branch**: `001-autenticacion`

**Created**: 2026-10-25

**Status**: Approved (Plan, 2026-10-26)

**Input**: User description: "La app permite a una persona controlar cada mes sus pagos recurrentes: saber cuáles ya pagó y cuáles tiene pendientes. Cada persona necesita una cuenta propia para que sus pagos estén separados de los de otros usuarios y nadie más pueda verlos. En esta feature el usuario puede: registrarse con email y contraseña, ingresar y cerrar sesión. Reglas: el email debe tener un formato válido y no puede estar registrado previamente. La contraseña debe tener mínimo 12 caracteres y permitir al menos 64; no se exigen combinaciones de mayúsculas, números o símbolos; se rechazan contraseñas comunes o filtradas. Decisión consciente: se usan 12 caracteres en lugar de los 15 que recomienda NIST SP 800-63B rev. 4, por comodidad en móvil, compensado con el bloqueo de contraseñas comunes y el límite de intentos. Por definir: qué pasa tras varios intentos fallidos de ingreso y cuánto tiempo permanece abierta la sesión. Fuera de alcance: ingreso con Google o Apple, recuperación de contraseña, verificación del email y eliminación de la cuenta (feature 006)."

## Contexto

La app sirve para que una persona controle cada mes sus pagos recurrentes (cuáles ya pagó y cuáles tiene pendientes). Esa información es financiera y personal, así que cada persona necesita una **cuenta propia**: sus datos deben quedar separados de los de cualquier otro usuario y nadie más debe poder verlos.

Esta feature entrega la puerta de entrada: crear la cuenta, ingresar y cerrar sesión. Todas las features posteriores (pagos, historial, etc.) dependen de ella.

### Decisión consciente sobre la longitud mínima de contraseña

NIST SP 800-63B rev. 4 recomienda un mínimo de 15 caracteres cuando la contraseña es el único factor. En este proyecto se fija el mínimo en **12 caracteres** por comodidad al escribir en el teclado del móvil. El riesgo se compensa con dos controles obligatorios de esta misma spec:

1. Rechazo de contraseñas comunes o filtradas (FR-009). La lista cubre las contraseñas filtradas **más frecuentes**, no todas las que se han filtrado alguna vez: una contraseña que apareció en una sola filtración puede aceptarse. Basta para ataques en línea, que el límite de intentos restringe a pocas pruebas por cuenta y donde se prueban primero las más comunes.
2. Límite de intentos fallidos de ingreso (FR-016 a FR-018) y de solicitudes de registro (FR-019).

Si alguno de estos dos controles se retira en el futuro, esta decisión debe revisarse.

### Decisiones aprobadas

Decisiones tomadas durante la redacción de la spec y aprobadas el 2026-10-25:

- **Máximo de 128 caracteres en la contraseña** (FR-007, FR-008): el requisito original pide "permitir al menos 64"; se fija 128 como máximo para dar margen a frases largas y a gestores de contraseñas, y para acotar el esfuerzo de verificación.
- **Sesiones múltiples con cierre por dispositivo** (FR-027): una misma cuenta puede tener sesión abierta en varios teléfonos a la vez; cerrar sesión afecta solo al dispositivo actual. No existe "cerrar sesión en todos los dispositivos" en esta feature.
- **Rechazo de contraseña igual al email** (FR-009): una contraseña idéntica al email de la persona se rechaza por ser demasiado predecible.

Decisiones aprobadas el 2026-10-26 durante la fase Plan:

- **Normalización Unicode NFC de la contraseña** (FR-031): la misma letra con tilde puede llegar codificada de dos formas según el teclado o el sistema operativo; se normaliza a NFC al registrar y al ingresar para que la persona pueda ingresar desde cualquier teléfono escribiendo lo mismo. Normalizar no se considera alterar la contraseña (FR-011).
- **Longitud medida en puntos de código Unicode tras NFC** (FR-007): reemplaza la medida "tal como la percibe la persona". Es exacta y reproducible en el servidor y en la app; como consecuencia, un emoji compuesto (por ejemplo 👨‍👩‍👧) cuenta como varios caracteres.
- **Tope de 1 024 bytes en la contraseña** (FR-032): acota el esfuerzo de procesar entradas anómalas, tanto en el registro como en el ingreso.
- **Los intentos rechazados durante una espera no cuentan como fallos** (FR-016): así nadie puede prolongar indefinidamente la espera de otra persona enviando intentos mientras está bloqueada.
- **El contador por email se reinicia tras 1 hora sin fallos, no 15 minutos** (FR-016, FR-018): con 15 minutos el reinicio coincidía con el tope de espera, así que al terminar una espera de 15 minutos el contador volvía a cero y nunca acumulaba; un atacante recuperaba 5 intentos libres en cada ciclo.
- **Cierre de sesión al reutilizar una credencial de renovación** (FR-033): la sesión se mantiene abierta renovando su credencial con el uso, y cada credencial de renovación sirve una sola vez. Si alguien presenta una ya usada, es señal de que pudo haberse copiado, así que se cierra la sesión entera y la persona debe ingresar de nuevo. Para no cerrar sesiones por cortes de red habituales en el móvil (la respuesta de una renovación que nunca llegó al teléfono), se admite una ventana de gracia de 30 segundos.

### Decisión consciente sobre revelar emails registrados

El registro informa explícitamente que un email ya tiene cuenta (FR-005). Con el alcance actual no es posible ocultarlo: el registro exitoso abre sesión de inmediato (FR-001) y la verificación del email está fuera de alcance, así que cualquier respuesta distinta ya revela la existencia del email. Se acepta el riesgo de que alguien averigüe qué emails tienen cuenta, mitigado con el límite de solicitudes de registro (FR-019). El ingreso sigue sin revelarlo (FR-015). Esta decisión debe revisarse cuando se incorpore la verificación del email.

## Clarifications

### Session 2026-10-26

- Q: Cuando alguien intenta registrarse con un email que ya tiene cuenta, ¿la app debe decirle explícitamente "este email ya está registrado"? → A: Sí, mensaje explícito ("Este email ya tiene una cuenta. Ingresa con él."); riesgo aceptado y mitigado con FR-019; se revisa al incorporar la verificación del email.
- Q: ¿Qué límites aplicamos a los intentos fallidos de ingreso y a las solicitudes de registro, y qué pasa al alcanzarlos? → A: Por email normalizado (exista o no): desde el 5.º fallo consecutivo, espera de 1 min que se duplica en cada fallo siguiente hasta un tope de 15 min; el contador se reinicia con un ingreso exitoso o tras 15 min sin fallos *(modificado en la fase Plan el 2026-10-26: tras 1 hora sin fallos; ver FR-016)*. Por origen: 20 ingresos fallidos en 15 min → espera de 15 min. Registro: 5 solicitudes por origen por hora.
- Q: ¿Cuánto tiempo debe seguir abierta la sesión en un teléfono antes de pedir de nuevo email y contraseña? → A: 30 días desde el último uso; "uso" es cualquier comunicación con el servidor con una sesión válida. Abrir la app sin conexión no renueva el plazo.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Crear una cuenta propia (Priority: P1)

Una persona que instala la app por primera vez crea su cuenta con su email y una contraseña. Al terminar queda dentro de la app, con un espacio propio que nadie más puede ver.

**Why this priority**: sin cuenta no existe ningún otro flujo; es el primer paso de cualquier usuario y la base del aislamiento de datos.

**Independent Test**: se puede probar completo registrando un email nuevo con una contraseña válida y comprobando que la persona queda con sesión abierta en su propia cuenta; también probando cada regla de rechazo (email inválido, email repetido, contraseña corta, larga o común).

**Acceptance Scenarios**:

1. **CA-001-01** — **Dado** que el email `ana@ejemplo.com` no está registrado, **Cuando** la persona se registra con ese email y una contraseña válida de 12 caracteres o más que no es común ni filtrada, **Entonces** se crea su cuenta y queda con sesión abierta.
2. **CA-001-02** — **Dado** que la persona está en el formulario de registro, **Cuando** escribe un email sin formato válido (por ejemplo `ana@`, `ana.ejemplo.com` o vacío), **Entonces** el registro se rechaza con un mensaje que indica que el email no es válido y no se crea ninguna cuenta.
3. **CA-001-03** — **Dado** que `ana@ejemplo.com` ya tiene una cuenta, **Cuando** alguien intenta registrarse con ese email (o con una variante que solo difiere en mayúsculas o espacios al inicio o final, como ` Ana@Ejemplo.com `), **Entonces** el registro se rechaza con el mensaje "Este email ya tiene una cuenta. Ingresa con él." y no se crea una segunda cuenta.
4. **CA-001-04** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña de 11 caracteres o menos, **Entonces** el registro se rechaza con un mensaje que indica el mínimo de 12 caracteres.
5. **CA-001-05** — **Dado** que la persona está en el formulario de registro, **Cuando** usa contraseñas válidas de exactamente 12, 64 y 128 caracteres, **Entonces** las tres se aceptan.
6. **CA-001-06** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña de 12 o más caracteres formada solo por letras minúsculas (sin mayúsculas, números ni símbolos) que no es común ni filtrada, **Entonces** se acepta.
7. **CA-001-07** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña que aparece en la lista de contraseñas comunes o filtradas (por ejemplo `contraseña123` o `123456789012`), **Entonces** el registro se rechaza con un mensaje que indica que esa contraseña es demasiado conocida y que elija otra.
8. **CA-001-08** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña de más de 128 caracteres, **Entonces** el registro se rechaza con un mensaje que indica el máximo permitido.
9. **CA-001-09** — **Dado** que la persona está en el formulario de registro, **Cuando** usa una contraseña con espacios, tildes, `ñ` o emojis que cumple la longitud y no es común, **Entonces** se acepta, y luego puede ingresar escribiendo exactamente la misma contraseña.
10. **CA-001-21** — **Dado** que la persona se registra con el email `ana@ejemplo.com`, **Cuando** usa como contraseña `ana@ejemplo.com`, **Entonces** el registro se rechaza con un mensaje que indica que la contraseña es demasiado predecible y no se crea la cuenta.
11. **CA-001-22** — **Dado** que el email `ana@ejemplo.com` no está registrado, **Cuando** llegan dos solicitudes de registro simultáneas con ese email, **Entonces** se crea exactamente una cuenta y la otra solicitud se rechaza como email ya registrado.
12. **CA-001-23** — **Dado** que desde un mismo origen se enviaron 5 solicitudes de registro en la última hora, **Cuando** se envía una 6.ª solicitud de registro desde ese origen dentro de esa hora, **Entonces** se rechaza sin crear la cuenta y se informa a la persona que espere antes de volver a intentar.
13. **CA-001-28** — **Dado** que Ana se registró con una contraseña que contiene `á` escrita como un único carácter (U+00E1), **Cuando** ingresa con la misma contraseña escrita con `a` seguida de la tilde combinante (U+0061 U+0301), **Entonces** ingresa a su cuenta; y viceversa (registro con la forma descompuesta, ingreso con la compuesta).
14. **CA-001-29** — **Dado** que la persona está en el formulario de registro, **Cuando** envía una contraseña que ocupa más de 1 024 bytes, **Entonces** el registro se rechaza con el mismo mensaje de máximo permitido de FR-008 y no se crea la cuenta; y **Dado** que existe la cuenta `ana@ejemplo.com`, **Cuando** alguien intenta ingresar con una contraseña de más de 1 024 bytes, **Entonces** recibe el mismo mensaje genérico de CA-001-12 y el intento cuenta como fallido.

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
5. **CA-001-14** — **Dado** que para `ana@ejemplo.com` se registraron 5 intentos fallidos consecutivos, **Cuando** se hace un nuevo intento antes de que pase 1 minuto (incluso con la contraseña correcta), **Entonces** se rechaza indicando cuándo puede volver a intentar; y cada fallo adicional duplica la espera (2, 4, 8 min…) hasta un máximo de 15 minutos. Con `nadie@ejemplo.com` (no registrado) se obtiene exactamente la misma respuesta.
6. **CA-001-15** — **Dado** que dos personas tienen cuenta (`ana@ejemplo.com` y `beto@ejemplo.com`), **Cuando** Ana ingresa, **Entonces** solo puede ver y operar sobre los datos asociados a su cuenta, y cualquier intento de acceder a datos de Beto se trata como si esos datos no existieran.
7. **CA-001-16** — **Dado** que la persona ingresó y cerró la app sin cerrar sesión, **Cuando** vuelve a abrir la app con conexión 29 días después de su último uso, **Entonces** sigue con la sesión abierta sin volver a escribir sus credenciales, y el plazo de 30 días vuelve a contar desde ese momento.
8. **CA-001-17** — **Dado** que pasaron más de 30 días desde el último uso de la sesión (aunque en ese tiempo la persona haya abierto la app sin conexión), **Cuando** intenta usar la app con conexión, **Entonces** se le pide ingresar de nuevo.
9. **CA-001-26** — **Dado** que para `ana@ejemplo.com` hay 4 intentos fallidos consecutivos, **Cuando** pasa 1 hora sin nuevos fallos y luego se produce 1 fallo más, **Entonces** no se aplica ninguna espera (el contador se reinició y ese fallo cuenta como el primero); y **Dado** el mismo estado de 4 fallos, **Cuando** la persona ingresa con éxito y después falla 1 vez, **Entonces** tampoco se aplica espera. Y **Dado** el mismo estado de 4 fallos, **Cuando** pasan 59 minutos sin fallos y se produce 1 fallo más, **Entonces** se aplica la espera de 1 minuto (5.º fallo).
10. **CA-001-27** — **Dado** que desde un mismo origen se registraron 20 ingresos fallidos en los últimos 15 minutos, repartidos entre emails distintos, **Cuando** se envía un nuevo ingreso desde ese origen (incluso con el email y la contraseña correctos de `ana@ejemplo.com`), **Entonces** se rechaza indicando cuándo puede volver a intentar, sin revelar si el email existe; desde otro origen, Ana ingresa sin restricción; y desde el origen restringido se vuelve a aceptar el ingreso cuando terminan los 15 minutos de espera.
11. **CA-001-30** — **Dado** que para `ana@ejemplo.com` hay 5 intentos fallidos consecutivos y una espera vigente de 1 minuto, **Cuando** se envían 3 intentos más durante esa espera (todos rechazados) y, al terminar la espera, se produce 1 fallo más, **Entonces** la nueva espera es de 2 minutos (6.º fallo), no la que correspondería a 9 fallos; y los intentos rechazados durante la espera no cambian el momento en que esta termina.
12. **CA-001-31** — **Dado** que la sesión de Ana renovó su credencial de renovación C1 y recibió C2, **Cuando** alguien presenta C1 más de 30 segundos después de esa renovación, **Entonces** se rechaza, la sesión se cierra (C2 tampoco sirve ya) y se pide ingresar de nuevo; **Dado** el mismo estado, **Cuando** C1 se presenta dentro de los 30 segundos y C2 todavía no se ha usado, **Entonces** se acepta y la sesión sigue abierta; y **Dado** que C2 ya se usó, **Cuando** se presenta C1 (dentro o fuera de los 30 segundos), **Entonces** se rechaza y la sesión se cierra.

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
- **Caracteres especiales**: tildes, `ñ`, emojis y espacios cuentan como caracteres válidos; la longitud se mide en puntos de código Unicode después de normalizar a NFC, no en bytes (CA-001-09, FR-007). Un emoji compuesto puede contar como varios caracteres.
- **Misma contraseña, distinta codificación**: una tilde escrita como carácter único o como letra + tilde combinante se trata como la misma contraseña (FR-031, CA-001-28).
- **Contraseña desmesurada**: una contraseña de más de 1 024 bytes se rechaza en el registro como demasiado larga y en el ingreso como credenciales incorrectas (FR-032, CA-001-29).
- **Doble toque en "Registrarme"**: dos envíos casi simultáneos con el mismo email nuevo crean una sola cuenta; el segundo envío se trata como email ya registrado (CA-001-22).
- **Sin conexión**: si no hay conexión al registrarse, ingresar o cerrar sesión, la app informa que no hay conexión y no da por hecha la operación; los datos escritos en el formulario (excepto la contraseña) no se pierden.
- **Cerrar sesión sin conexión**: la app borra igualmente la sesión del teléfono y vuelve a la pantalla de ingreso; la invalidación en el servidor se completa al recuperar conexión o al expirar la sesión.
- **Abrir la app sin conexión**: no renueva el plazo de la sesión (FR-021); solo cuenta como uso la comunicación con el servidor con una sesión válida (CA-001-17).
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
- **FR-005** *(no deseado)*: SI el registro se rechaza porque el email ya existe, ENTONCES el sistema DEBE informar explícitamente que el email ya tiene una cuenta e invitar a ingresar con él (ver "Decisión consciente sobre revelar emails registrados"). → CA-001-03
- **FR-006** *(no deseado)*: SI la contraseña tiene menos de 12 caracteres, ENTONCES el sistema DEBE rechazar el registro indicando el mínimo de 12 caracteres. → CA-001-04
- **FR-007** *(ubicuo)*: El sistema DEBE aceptar contraseñas de 12 a 128 caracteres (cumpliendo así el requisito de permitir al menos 64). En esta spec, la longitud de la contraseña (mínimo de FR-006, máximo de FR-007 y FR-008) se cuenta en puntos de código Unicode después de la normalización NFC de FR-031. → CA-001-05
- **FR-008** *(no deseado)*: SI la contraseña tiene más de 128 caracteres, ENTONCES el sistema DEBE rechazar el registro indicando el máximo de 128 caracteres. → CA-001-08
- **FR-009** *(no deseado)*: SI la contraseña aparece en la lista de contraseñas comunes o filtradas, o es igual al email de la persona, ENTONCES el sistema DEBE rechazar el registro indicando que la contraseña es demasiado conocida o predecible y pidiendo otra. → CA-001-07, CA-001-21
- **FR-010** *(ubicuo)*: El sistema NO DEBE exigir combinaciones de mayúsculas, minúsculas, números o símbolos en la contraseña. → CA-001-06
- **FR-011** *(ubicuo)*: El sistema DEBE aceptar en la contraseña cualquier carácter imprimible, incluidos espacios, tildes, `ñ` y emojis, y NO DEBE recortarla, truncarla ni alterarla; la normalización NFC de FR-031 no se considera una alteración. → CA-001-09
- **FR-012** *(ubicuo)*: El sistema DEBE permitir pegar texto y usar gestores de contraseñas en los campos de contraseña de registro e ingreso. → verificado por revisión
- **FR-013** *(ubicuo)*: El sistema DEBE garantizar que dos solicitudes de registro simultáneas con el mismo email produzcan como máximo una cuenta. → CA-001-22

**Ingreso**

- **FR-014** *(por evento)*: CUANDO una persona envía su email y su contraseña correctos, el sistema DEBE abrir una sesión en su cuenta. → CA-001-10, CA-001-11
- **FR-015** *(no deseado)*: SI el email no está registrado o la contraseña no corresponde, ENTONCES el sistema DEBE rechazar el ingreso con el mismo mensaje genérico en ambos casos, y en un tiempo de respuesta que no permita distinguir un caso del otro. → CA-001-12, CA-001-13
- **FR-016** *(ubicuo)*: El sistema DEBE contar los intentos fallidos de ingreso de dos formas y aplicar estos límites:
  - **Por email normalizado** (exista o no una cuenta con ese email): a partir del 5.º fallo consecutivo, impone una espera de 1 minuto que se duplica con cada fallo siguiente (1, 2, 4, 8 min…) hasta un máximo de 15 minutos. El contador se reinicia con un ingreso exitoso o cuando pasa 1 hora desde el último fallo. → CA-001-14, CA-001-26
  - **Por origen**: si desde un mismo origen se acumulan 20 ingresos fallidos en una ventana de 15 minutos, impone una espera de 15 minutos a los ingresos desde ese origen. → CA-001-27
  - **Intentos durante una espera**: los intentos de ingreso rechazados por una espera vigente (FR-017) NO cuentan como fallos, ni para el conteo por email ni para el conteo por origen, y no prolongan la espera. → CA-001-30
- **FR-017** *(no deseado)*: SI se alcanza el límite de FR-016, ENTONCES el sistema DEBE rechazar los nuevos intentos de ingreso mientras dure la restricción, incluso con la contraseña correcta, e informar a la persona cuándo puede volver a intentar, sin revelar si el email existe. → CA-001-14, CA-001-27
- **FR-018** *(por evento)*: CUANDO una persona ingresa con éxito, o pasa 1 hora desde el último intento fallido con un email, el sistema DEBE reiniciar el conteo de intentos fallidos de ese email. → CA-001-26
- **FR-019** *(ubicuo)*: El sistema DEBE limitar también la frecuencia de solicitudes de registro desde un mismo origen, para impedir la creación masiva de cuentas y el sondeo de emails, con un límite de 5 solicitudes de registro por origen en una ventana de 1 hora; las solicitudes que superen el límite se rechazan sin crear cuenta. → CA-001-23

**Sesión**

- **FR-020** *(por estado)*: MIENTRAS la sesión esté vigente, el sistema DEBE mantener a la persona dentro de su cuenta aunque cierre y reabra la app. → CA-001-16
- **FR-021** *(por evento)*: CUANDO se cumpla el tiempo de vigencia de la sesión, el sistema DEBE dar la sesión por expirada; la sesión DEBE dejar de ser válida tras 30 días sin uso. "Uso" es cualquier comunicación de la app con el servidor con esa sesión válida; cada uso reinicia el plazo de 30 días. Abrir la app sin conexión NO cuenta como uso. → CA-001-16, CA-001-17
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

**Tratamiento de la contraseña (registro e ingreso)**

- **FR-031** *(ubicuo)*: El sistema DEBE normalizar la contraseña a la forma Unicode NFC antes de medir su longitud, compararla con la lista de FR-009 y con el email, y verificarla, tanto al registrar como al ingresar. → CA-001-28
- **FR-032** *(no deseado)*: SI la contraseña recibida ocupa más de 1 024 bytes, ENTONCES el sistema DEBE rechazarla sin procesarla: en el registro, con el mensaje de máximo permitido de FR-008; en el ingreso, como un intento fallido con el mensaje genérico de FR-015. → CA-001-29

**Renovación de la sesión**

- **FR-033** *(no deseado)*: SI se presenta una credencial de renovación ya usada fuera de la ventana de gracia, ENTONCES el sistema DEBE cerrar esa sesión y pedir ingresar de nuevo. La ventana de gracia son los 30 segundos siguientes a la renovación en que se usó esa credencial, y solo aplica mientras la credencial emitida en esa renovación no se haya usado. → CA-001-31

### Key Entities *(include if feature involves data)*

- **Cuenta de usuario**: representa a una persona. Atributos: email normalizado (único), verificador de la contraseña (no la contraseña), fecha de creación. Es la dueña de todos los datos de usuario que se creen en features posteriores.
- **Sesión**: acceso abierto de una cuenta en un dispositivo concreto. Atributos: cuenta a la que pertenece, momento de inicio, momento del último uso, momento de expiración (último uso + 30 días), estado (vigente, cerrada, expirada). Una cuenta puede tener varias sesiones (una por dispositivo).
- **Registro de intentos fallidos**: conteo usado para aplicar los límites de FR-016 y FR-019. Por email normalizado (exista o no una cuenta): fallos consecutivos, momento del último fallo y fin de la espera vigente. Por origen: fallos de ingreso y solicitudes de registro dentro de su ventana de tiempo. No guarda contraseñas.

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
