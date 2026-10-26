# Research — 001-autenticacion

**Fecha**: 2026-10-26 | **Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Cada sección sigue el formato **Decisión → Por qué → Alternativas consideradas**. Las dos primeras (R1 y R2) son las investigaciones pedidas explícitamente; el resto resuelve los puntos abiertos del Technical Context y de las "Notas para el Plan" de `docs/STATUS.md`.

> Las versiones exactas de las dependencias se fijan en la tarea de esqueleto, con confirmación previa (CLAUDE.md, "Límites"). Aquí se decide **qué** usar y **por qué**.

---

## R1. Algoritmo de hash de contraseñas (FR-028, CA-001-24)

### Decisión

**Argon2id** con los parámetros mínimos recomendados por OWASP (Password Storage Cheat Sheet):

| Parámetro | Valor |
|---|---|
| Memoria (`m`) | 19 456 KiB (19 MiB) |
| Iteraciones (`t`) | 2 |
| Paralelismo (`p`) | 1 |
| Salt | 16 bytes aleatorios por contraseña (lo genera la librería) |
| Salida | 32 bytes, guardado en formato PHC (`$argon2id$v=19$m=19456,t=2,p=1$<salt>$<hash>`) |

Librería: **`argon2`** (node-argon2), que usa la implementación de referencia en C y publica binarios precompilados (glibc y musl), así que no requiere toolchain en CI ni en Docker.

Los parámetros quedan en una constante del módulo de autenticación. Como el formato PHC guarda los parámetros junto al hash, subirlos en el futuro no rompe los hashes existentes.

### Por qué

1. **La constitución (V) solo admite argon2id o bcrypt.** Entre esos dos se elige.
2. **bcrypt trunca la entrada a 72 bytes.** La spec permite hasta 128 caracteres con tildes y emojis (FR-007, FR-011). Un emoji ocupa 4 bytes en UTF-8 o más si lleva modificadores, así que una contraseña válida de 128 caracteres puede pasar de 500 bytes. Con bcrypt, dos contraseñas que comparten los primeros 72 bytes serían equivalentes. Eso contradice FR-011 ("NO DEBE truncarla"). Para evitarlo habría que pre-hashear (SHA-256 + base64) antes de bcrypt, lo cual agrega una construcción propia que es fácil equivocar (por ejemplo, los bytes nulos en la salida binaria).
3. **Argon2id es memory-hard.** El costo de memoria encarece los ataques con GPU/ASIC sobre una base de datos robada. Es el ganador de la Password Hashing Competition, la recomendación principal de OWASP y una de las opciones de RFC 9106.
4. **Parámetros de OWASP y no los de RFC 9106 (64 MiB, t=3, p=4).** El servidor es modesto (detrás de Cloudflare Tunnel, sin escalado horizontal). 19 MiB por verificación aguanta varios ingresos concurrentes sin presionar la RAM. El riesgo de DoS por hashing se acota además con los límites de FR-016 y FR-019. Objetivo: cada verificación en menos de 250 ms en el servidor de staging. Si se queda muy por debajo, se sube `t` (se mide en la tarea correspondiente).

### Alternativas consideradas

| Alternativa | Por qué se descarta |
|---|---|
| **bcrypt** (cost 12) | Trunca a 72 bytes (incompatible con FR-011 sin pre-hash). No es memory-hard. |
| **bcrypt con pre-hash SHA-256** | Funciona, pero es una construcción propia con trampas conocidas. No aporta nada sobre argon2id. |
| **scrypt** (`node:crypto`, sin dependencia) | Memory-hard y nativo. La constitución no lo admite: cambiarlo requeriría ADR + enmienda. |
| **PBKDF2** | No es memory-hard y la constitución no lo admite. |
| **`crypto.argon2()` nativo de Node 24** | Evita la dependencia, pero la API es experimental y no trae codificación/verificación PHC. Se puede reevaluar cuando sea estable. |
| **`@node-rs/argon2`** (Rust + napi) | Técnicamente equivalente. Se prefiere `argon2` por adopción y mantenimiento más largos. Se puede cambiar sin migrar datos (ambos usan PHC). |
| **Pepper (secreto global añadido al hash)** | Añade un secreto que rotar y, si se pierde, invalida todas las contraseñas. Con el alcance actual el beneficio es marginal. YAGNI (principio VII). |

---

## R2. Detección de contraseñas comunes o filtradas sin servicios de pago (FR-009, CA-001-07, SC-003)

### Decisión

**Lista local de contraseñas prohibidas, versionada en el repo y cargada en memoria al arrancar la API.**

- **Fuente**: listas públicas de [SecLists](https://github.com/danielmiessler/SecLists) (licencia MIT), derivadas de filtraciones reales:
  - `100k-most-used-passwords-NCSC` (top 100 000 de Have I Been Pwned publicado por el NCSC del Reino Unido).
  - `10-million-password-list-top-1000000` (compilación de filtraciones de xato.net).
- **Complemento propio en español** (`common-passwords.es.txt`): contraseñas previsibles en Colombia o en español que las listas en inglés no cubren. Ejemplos: `contraseña123`, `contrasena123`, `colombia2026`, `123456789012`, `teamo123456789`, nombres de equipos + año. Esta lista se mantiene a mano y crece con el tiempo.
- **Filtrado al generar**: cada entrada se normaliza a NFC y se pasa a minúsculas; se quedan solo las de 12 a 128 puntos de código (las de menos de 12 ya las rechaza FR-006). Se eliminan duplicados. Así el archivo resultante es pequeño (decenas de miles de líneas, objetivo < 2 MB).
- **Script reproducible** `apps/api/scripts/build-password-blocklist.ts`. Toma las fuentes descargadas a mano, aplica el filtro y escribe `apps/api/src/auth/password-policy/data/blocklist.txt`. El commit SHA de SecLists usado y su licencia quedan anotados en la cabecera del archivo generado.
- **Comparación**: `blocklist.has(password.normalize('NFC').toLowerCase())` sobre un `Set<string>` en memoria (FR-031; las entradas de la lista también se normalizan a NFC al generarla). Se compara sin distinguir mayúsculas: `Contraseña123` también se rechaza, y eso es más estricto que lo que pide la spec, pero no la contradice.
- **Contraseña igual al email** (CA-001-21): se compara `password.normalize('NFC').trim().toLowerCase()` con el email normalizado. El recorte solo se usa para la comparación; la contraseña guardada no se altera (FR-011).
- **SC-003**: el test toma una muestra fija (semilla determinista) de la lista y verifica que el 100 % se rechaza vía `POST /v1/auth/register`.

### Por qué

1. **La contraseña nunca sale del servidor** (supuesto de la spec: "sin enviar la contraseña de la persona a terceros").
2. **Cero servicios nuevos** (restricción del plan) y cero latencia o fallos externos en el registro.
3. **Determinista y testeable**: el mismo archivo en CI, staging y producción. SC-003 se verifica sin mocks.
4. **Proporcional al riesgo aceptado**: la spec compensa los 12 caracteres con bloqueo de contraseñas comunes + límite de intentos. Las contraseñas más usadas son las que un atacante en línea prueba primero, y con FR-016 solo puede probar unas pocas por cuenta. La cola larga de filtraciones aporta poco frente a ataques en línea.

### ⚠️ Limitación explícita: cubre contraseñas **comunes**, no **todas las filtradas**

La spec habla de "contraseñas comunes o filtradas" (FR-009). Esta solución **no detecta cualquier contraseña filtrada**. Detecta solo las contraseñas **más frecuentes** que aparecen en filtraciones conocidas (unos cientos de miles antes de filtrar por longitud).

- Have I Been Pwned tiene más de 900 millones de contraseñas filtradas únicas. Una contraseña que apareció una sola vez en una filtración (por ejemplo `MiPerroFirulais2019`) **se acepta** si no está en la lista local.
- Esto es una **limitación aceptada**, no un cumplimiento total de "filtradas". Se justifica porque los ataques que nos afectan son en línea y el límite de intentos (FR-016) solo deja probar unas pocas contraseñas por cuenta. Ese atacante prueba primero las más comunes, que son justamente las que bloquea la lista. La cola larga solo sirve en ataques fuera de línea, y contra esos protege argon2id (R1).
- **SC-003** se cumple sobre una muestra **tomada de la lista local**. No es una garantía sobre el conjunto completo de filtraciones.
- **Cuándo revisar**: si se acepta un servicio externo, la API de rangos de HIBP (ver alternativas) cerraría la brecha sin enviar la contraseña.
- La lista hay que regenerarla de vez en cuando. Queda como tarea de mantenimiento, no de esta feature.

### Alternativas consideradas

| Alternativa | Por qué se descarta (por ahora) |
|---|---|
| **API de rangos de HIBP Pwned Passwords** (k-anonimato: se envían los 5 primeros caracteres del SHA-1) | Es gratuita y no envía la contraseña. Pero es un **servicio externo en tiempo de ejecución**: el registro dependería de su disponibilidad, latencia y términos de uso, y habría que decidir qué hacer si cae (¿aceptar? ¿rechazar?). Va contra la restricción "no agregar servicios". Es la mejor candidata para cuando se acepte una dependencia externa. |
| **Dataset completo de HIBP descargado** (~2 000 M hashes SHA-1/NTLM, decenas de GB) | Desproporcionado para el servidor y para el repo. Requiere un proceso de descarga/actualización. |
| **Filtro de Bloom del dataset HIBP** (~1–2 GB con 0,1 % de falsos positivos) | Reduce el tamaño, pero sigue siendo grande. Los falsos positivos rechazarían contraseñas válidas sin explicación posible, y habría que generarlo y distribuirlo fuera del repo. |
| **Tabla en PostgreSQL con la lista** | Funciona, pero agrega una migración con decenas de miles de filas y una consulta por registro, para datos que nunca cambian en tiempo de ejecución. Un `Set` en memoria es más simple. |
| **zxcvbn / zxcvbn-ts** (estimador de fortaleza) | Mide "adivinabilidad", no pertenencia a una lista. Rechazaría contraseñas que la spec acepta (FR-010: sin reglas de composición). Agrega una dependencia grande. |
| **RockYou completo** (14 M) filtrado a ≥ 12 caracteres | Más cobertura, pero el resultado pesa decenas de MB y aumenta memoria y tamaño del repo. Se puede reconsiderar si la lista elegida se queda corta. |

---

## R3. Longitud, normalización NFC y tope en bytes (FR-006–FR-008, FR-011, FR-031, FR-032, CA-001-05, CA-001-09, CA-001-28, CA-001-29)

> Revisado el 2026-10-26 tras aprobar FR-031 y FR-032. La versión anterior de esta sección proponía contar grafemas con `Intl.Segmenter`; la spec fija ahora **puntos de código Unicode después de NFC**.

### Decisión

Orden de tratamiento de la contraseña en la API, igual en registro e ingreso:

1. **Tope en bytes sobre la entrada cruda** (FR-032): `Buffer.byteLength(password, 'utf8') > 1024`. Se comprueba **antes** de normalizar, hashear o consultar la lista.
   - En el registro → `400 PASSWORD_TOO_LONG` (mismo mensaje de FR-008).
   - En el ingreso → se registra como intento fallido (contadores de FR-016) y se responde `401 INVALID_CREDENTIALS`, **sin** verificar con argon2. Ninguna cuenta puede tener esa contraseña, así que la respuesta no revela si el email existe. El trabajo ahorrado es el mismo para emails existentes e inexistentes, así que FR-015 se mantiene.
2. **Normalización** (FR-031): `const normalized = password.normalize('NFC')`. Desde aquí solo se usa `normalized`.
3. **Longitud** (FR-006–FR-008, solo en el registro): `[...normalized].length` entre 12 y 128.
4. **Lista prohibida** (FR-009): `blocklist.has(normalized.toLowerCase())`. El script que genera la lista también aplica `normalize('NFC')` y `toLowerCase()` a cada entrada.
5. **Igual al email** (FR-009): `normalized.trim().toLowerCase() === emailNormalizado`.
6. **Hash y verificación**: argon2id sobre `normalized`, en registro y en ingreso.

**App**: aplica la misma medida de longitud (pasos 2 y 3) para avisar antes de enviar. El servidor sigue siendo la fuente de verdad. La app **no** aplica la lista ni el tope en bytes.

### 📌 Nota de implementación: cómo medir la longitud

> La longitud de la contraseña se mide **siempre** con `[...password.normalize('NFC')].length`, en la app y en la API. **Nunca** con `password.length`.

- `String.prototype.length` cuenta **unidades UTF-16**, no caracteres. Todo carácter fuera del BMP (U+10000 en adelante: emojis como 😀 U+1F600, algunos ideogramas CJK) ocupa **2** unidades (un par sustituto), así que `'😀'.length === 2`.
- El spread `[...str]` itera por **puntos de código**, así que `[...'😀'].length === 1`.
- Hay que normalizar **antes** de contar: `'á'` compuesta (U+00E1) es 1 punto de código, pero la forma descompuesta (U+0061 U+0301) son 2. Sin NFC, la misma contraseña tendría longitudes distintas según el teclado.
- Para no repetir la expresión, cada proyecto define una sola función (`passwordLength(password: string): number`) y una regla de lint/revisión prohíbe `password.length` en el código de validación. No es un paquete compartido (YAGNI): son dos funciones de una línea, cada una con sus tests.
- **Verificar en el esqueleto**: que el motor JS de la app (Hermes) implementa `String.prototype.normalize('NFC')`. Los tests con RNTL corren en Node y no lo detectarían, así que se comprueba en un dispositivo o emulador (flujo de Maestro de registro con tildes). Si Hermes no lo soportara, la app omite la prevalidación de longitud y muestra el error del servidor.

### Datos de prueba obligatorios para la longitud

Para que un test detecte el uso accidental de `.length`, los casos de borde **deben incluir caracteres fuera del BMP**:

| CA | Caso | Puntos de código (tras NFC) | `.length` (UTF-16) | Esperado |
|---|---|---|---|---|
| CA-001-05 | 12 caracteres con 1 emoji: `'😀' + 'gatoazulsol'` | 12 | 13 | Se acepta |
| CA-001-05 | 64 caracteres con emojis: `'🦜'.repeat(4) + 60 letras` | 64 | 68 | Se acepta |
| CA-001-05 | **128 caracteres con emoji**: `'😀' + 127 letras` | 128 | 129 | Se acepta (con `.length` se rechazaría por error) |
| CA-001-08 | 129 caracteres: `'😀' + 128 letras` | 129 | 130 | Se rechaza |
| CA-001-04 | **11 caracteres con 6 emojis**: `'😀'.repeat(6) + 'abcde'` | 11 | 17 | Se rechaza (con `.length` se aceptaría por error) |
| CA-001-09 | Compuesto: `'👨‍👩‍👧'` (U+1F468 ZWJ U+1F469 ZWJ U+1F467) + 7 letras | 12 | 15 | Se acepta e ingresa con la misma cadena |
| CA-001-28 | `'canción' + …` con `ó` descompuesta (U+006F U+0301) en el registro, compuesta en el ingreso | igual en ambas formas | distinto | Ingresa |
| CA-001-29 | `'a'.repeat(1025)` y `'😀'.repeat(257)` (1 028 bytes) | > 128 | — | Registro: `PASSWORD_TOO_LONG`; ingreso: `INVALID_CREDENTIALS` y cuenta como fallo |

Las letras de relleno se generan con un patrón no trivial (por ejemplo, una frase repetida) para que ninguna cadena coincida con la lista prohibida. Un test aparte comprueba que ninguna de estas cadenas está en la lista.

### Observación: el tope en bytes no se alcanza dentro del rango válido

Un punto de código ocupa como máximo 4 bytes en UTF-8, así que una contraseña válida de 128 puntos de código pesa ≤ 512 bytes. En el registro, FR-032 solo actúa sobre entradas que FR-008 rechazaría de todos modos. Lo que aporta es rechazarlas **antes** de normalizar. Donde sí es imprescindible es en el **ingreso**, que no aplica política de longitud y sin el tope procesaría contraseñas de cualquier tamaño (cuerpo limitado por el servidor HTTP).

### Alternativas consideradas

- **Grafemas con `Intl.Segmenter`** (la propuesta original): coincide mejor con "lo que ve la persona", pero Hermes no lo soporta de forma fiable y el resultado depende de la versión de Unicode del motor. La spec eligió puntos de código.
- **Unidades UTF-16 (`str.length`)**: incorrecto para caracteres fuera del BMP (ver nota).
- **NFKC en vez de NFC**: NFKC también pliega compatibilidades (`ﬁ` → `fi`, dígitos de ancho completo → ASCII). Así acepta más variantes como iguales, pero altera más la contraseña de lo que pide FR-031. NFC solo unifica formas canónicamente equivalentes.
- **Normalizar solo en el servidor**: obligaría a la app a contar sin normalizar, y la app y la API discreparían en contraseñas con tildes descompuestas.

---

## R4. Modelo de sesión: token de acceso + token de renovación opaco con rotación (FR-020–FR-022, FR-026, FR-027, SC-005)

### Decisión

| Pieza | Diseño |
|---|---|
| **Token de acceso** | JWT HS256 firmado con `JWT_ACCESS_SECRET`, vigencia **15 min**. Claims: `sub` (userId), `sid` (sessionId), `iat`, `exp`. Librería `@nestjs/jwt`. |
| **Validación del acceso** | `AccessTokenGuard` global (deny by default; `@Public()` para excepciones). Verifica firma y `exp` con el **reloj inyectado** (`clockTimestamp`). Luego **consulta la sesión por `sid`** (PK) y exige `revokedAt IS NULL`, `expiresAt > now` y `userId = sub`. |
| **"Uso" (FR-021)** | En cada solicitud autenticada válida (acceso o renovación) se actualiza `lastUsedAt = now` y `expiresAt = now + 30 días`. Es un `UPDATE` por PK por solicitud, aceptable a esta escala y exacto respecto a la spec. |
| **Token de renovación** | 32 bytes aleatorios (`crypto.randomBytes`) en base64url. En BD solo se guarda su **SHA-256** (índice único). Como tiene 256 bits de entropía, un hash rápido basta; no hace falta argon2. |
| **Rotación** | Cada `POST /v1/auth/refresh` marca el token presentado como usado (`usedAt`, `replacedById`) y emite un par nuevo. |
| **Detección de reutilización** | Si llega un token ya usado, **se revoca toda la sesión** (`revokedReason = REFRESH_REUSE`) y se responde 401. Excepción: **ventana de gracia de 30 s**. Si el token usado fue reemplazado hace menos de 30 s y su reemplazo **no se ha usado**, se revoca ese reemplazo y se emite otro par. |
| **Cierre de sesión** | `POST /v1/auth/logout` con el token de renovación. Marca `revokedAt` (`LOGOUT`) solo en **esa** sesión (FR-027). Es idempotente (204 siempre). Desde ese momento el guard rechaza también los accesos vigentes de esa sesión (SC-005). |
| **Varias sesiones** | Cada registro o ingreso crea una `Session` nueva (una por dispositivo). |

### Por qué

- **Lo exige la constitución (V)**: "tokens de acceso de vida corta y tokens de renovación revocables".
- **Consultar la sesión en cada solicitud** es necesario por SC-005 y CA-001-19: "repitiendo una petición con la credencial anterior → se rechaza". Un JWT puramente stateless seguiría siendo válido hasta 15 min después del logout. La consulta por PK es barata y además permite registrar el "uso" exacto de FR-021.
- **Rotación + reutilización**: la expiración por inactividad deslizante (FR-021) significa que una sesión robada podría vivir indefinidamente si el atacante la sigue usando. Rotar y detectar la reutilización hace que, si el token se filtra, el primero que lo reutilice (el atacante o la persona) provoque la revocación, y la persona lo note al tener que ingresar de nuevo.
- **Ventana de gracia**: en móvil es normal que la respuesta de un `refresh` se pierda (cambio de red, app suspendida). Sin gracia, el reintento con el token anterior parecería reutilización y cerraría la sesión sin motivo, lo que rompe FR-020. Con 30 s y "reemplazo no usado" un atacante tiene muy poco margen. Es el mismo enfoque del "reuse interval" de proveedores como Auth0.
- **Single-flight en la app**: el cliente garantiza una sola renovación en curso (promesa compartida). Así varias consultas que reciben 401 a la vez no disparan renovaciones paralelas, que el servidor tomaría como reutilización.

### Alternativas consideradas

| Alternativa | Por qué se descarta |
|---|---|
| **JWT stateless sin consultar BD** | No cumple SC-005 (el acceso sigue válido tras el logout) y no puede registrar el "uso" exacto. |
| **Solo tokens opacos (acceso también opaco)** | Funcionaría igual de bien con la consulta por solicitud, pero se aparta del modelo "acceso de vida corta + renovación" pedido. El JWT permite rechazar sin ir a BD los tokens mal firmados o vencidos. |
| **Token de renovación como JWT** | No hace falta que sea autocontenido: igual se consulta en BD para rotar y revocar. Opaco es más simple y no expone claims. |
| **Rotación sin ventana de gracia** | Cierres de sesión espurios en redes móviles inestables. |
| **Passport (`@nestjs/passport` + `passport-jwt`)** | Agrega dos dependencias para algo que `@nestjs/jwt` + un guard propio resuelven. Además, inyectar el reloj en `passport-jwt` es más incómodo. |
| **Cookies HttpOnly** | No aplican a una app nativa; la spec exige almacenamiento seguro del sistema operativo (FR-030). |

---

## R5. Contadores de límites en PostgreSQL (FR-016–FR-019)

### Decisión

Tres tablas (detalle en [data-model.md](./data-model.md)):

1. **`login_email_throttle`**: una fila por email normalizado (exista o no la cuenta), con clave `emailKey = HMAC-SHA256(THROTTLE_HMAC_SECRET, email)`. Guarda `consecutiveFailures`, `lastFailureAt` y `lockedUntil`.
2. **`rate_limit_event`**: eventos con marca de tiempo (`LOGIN_FAILURE`, `REGISTER_REQUEST`) por `originKey`, para las ventanas deslizantes (20 fallos en 15 min; 5 registros en 1 h).
3. **`login_origin_lock`**: bloqueo de 15 min por origen, que se activa al llegar al 20.º fallo.

**Reglas exactas** (todas con `now = clock.now()`):

- *Espera por email* tras el fallo n (n ≥ 5): `min(15, 2^(n−5))` minutos, es decir 1, 2, 4, 8, 15, 15… Ej.: 5.º fallo → 1 min.
- *Reinicio* (FR-018): si `lastFailureAt + 1 h ≤ now`, el contador se considera en 0 antes de evaluar. Un ingreso exitoso borra la fila. (Antes eran 15 min; se cambió en la spec el 2026-10-26, ver el riesgo de abajo.)
- *Intentos durante la espera* (FR-016, CA-001-30; ya está en la spec): se rechazan con 429 **sin verificar la contraseña** y **no suman** fallos, ni al contador del email ni a los eventos `LOGIN_FAILURE` del origen. Tampoco cambian `lockedUntil`. "Cada fallo adicional" (CA-001-14) es cada fallo de verificación tras terminar la espera vigente. Un intento de más de 1 024 bytes fuera de una espera **sí** cuenta como fallo (FR-032).
- *Origen*: antes de verificar se mira `login_origin_lock`. Tras cada fallo se inserta un evento y, si los eventos de los últimos 15 min son ≥ 20, se hace upsert del bloqueo hasta `now + 15 min`.
- *Registro* (FR-019): se cuentan **todas** las solicitudes a `POST /v1/auth/register`, incluidas las rechazadas por validación (para que no sirvan para sondear emails), salvo las rechazadas por el propio límite. Si en la última hora ya hay 5 → 429. Se aplica en un **guard**, que corre antes del `ValidationPipe` de NestJS, para contar también los cuerpos inválidos.
- *Limpieza*: al insertar un evento se borran los eventos de esa clave con más de 1 h. No hace falta cron ni otro servicio.

**Concurrencia**: el ingreso de un email se serializa con `pg_advisory_xact_lock(hashtextextended(emailKey, 0))` dentro de una transacción que cubre leer el contador → verificar → actualizar. Sin esto, 100 intentos en paralelo pasarían todos antes de que el contador llegara a 5. El conteo por origen no se serializa (eso bloquearía a todos los usuarios detrás de una misma IP), así que acepta un pequeño exceso acotado por la concurrencia.

### Por qué

- La restricción del plan exige Postgres (sin Redis).
- Ventana deslizante con eventos: es lo que dice la spec ("20 en una ventana de 15 minutos") sin el efecto borde de las ventanas fijas.
- HMAC en vez del email o la IP en claro: no se guardan datos personales de personas sin cuenta ni IPs (principio V). Un SHA-256 simple sería reversible por fuerza bruta sobre emails conocidos.

### Riesgos anotados (no son conflictos con la spec)

- **Reinicio tras 1 hora (resuelto en la spec)**: con la regla original (reinicio a los 15 min), el reinicio coincidía con la espera tope de 15 min. Al terminar cada espera tope el contador volvía a 0 y el atacante recuperaba 5 intentos libres por ciclo. Con el reinicio a 1 h el contador sí acumula: tras alcanzar el tope, cada fallo impone otros 15 min, así que el atacante consigue como máximo **1 intento cada 15 min por email** (~96 al día) mientras no pare 1 h completa. Costo para la persona legítima: si olvidó la contraseña y llegó al tope, cada nuevo intento fallido la hace esperar 15 min, hasta que pase 1 h sin fallos o ingrese bien.
- **CGNAT de operadores móviles**: muchas personas pueden compartir IP pública. Con 20 fallos por origen en 15 min, una IP compartida muy activa podría bloquear a usuarios legítimos. Se acepta por la escala del proyecto y se vigila en staging.

### Alternativas consideradas

- **`@nestjs/throttler`**: limita por petición y por ventana fija, en memoria o con un storage a medida. No modela "fallos consecutivos con espera exponencial ni reinicio por éxito". Habría que escribir el storage para Postgres igualmente.
- **Contadores en memoria**: se pierden al reiniciar y no se comparten entre réplicas. Además, la restricción lo prohíbe.
- **Ventana fija (contador por bucket de 15 min)**: más simple, pero permite 2× el límite en el borde entre buckets.
- **Bloqueo `SELECT … FOR UPDATE` sobre la fila del contador**: no funciona cuando la fila aún no existe (primer fallo). El advisory lock cubre ese caso.

---

## R6. Origen detrás de Cloudflare Tunnel

### Decisión

`OriginResolver` (servicio en `common/`):

- Si `TRUST_CF_CONNECTING_IP=true` (staging y producción) y llega la cabecera `CF-Connecting-IP` con una IP válida, usa esa IP.
- Si no, usa la dirección del socket (`req.socket.remoteAddress`). En local y en tests la flag está apagada, y los tests inyectan la cabecera con la flag encendida para simular orígenes distintos (CA-001-27).
- Si la flag está encendida y la cabecera falta, usa el socket y registra un aviso (sin la IP). Ese caso indica mala configuración, porque todo el tráfico se vería como un único origen.
- **IPv6 se agrupa por prefijo /64**, porque un solo cliente suele controlar un /64 completo y rotar direcciones dentro de él para evadir el límite.
- `originKey = HMAC-SHA256(THROTTLE_HMAC_SECRET, ipNormalizada)`.

### Por qué

- Detrás del túnel, `req.ip` es siempre la IP de `cloudflared`. Sin esta cabecera todos los usuarios contarían como un solo origen (nota de STATUS).
- `CF-Connecting-IP` es la cabecera que Cloudflare garantiza con una sola IP. `X-Forwarded-For` se puede encadenar o falsificar.
- La API no expone puertos públicos: solo se alcanza por el túnel (o por Tailscale para administración), así que la cabecera no puede venir de un cliente que se salte Cloudflare.

### Alternativas consideradas

- **`app.set('trust proxy', …)` + `req.ip`**: depende de `X-Forwarded-For` y de contar saltos. Es más frágil que leer la cabecera específica de Cloudflare.
- **Confiar en la cabecera siempre**: en local, cualquiera podría falsificarla. La flag por entorno lo evita.

---

## R7. Tiempo inyectable

### Decisión

- Interfaz `Clock { now(): Date }` con token de inyección `CLOCK`.
- `SystemClock` en producción y `FakeClock` en tests (`set(date)`, `advance(ms)`).
- **Toda** comparación temporal usa `clock.now()` en la aplicación y pasa ese instante como parámetro a las consultas. Nunca `now()` ni `CURRENT_TIMESTAMP` de Postgres, ni `@default(now())` / `@updatedAt` de Prisma para campos con lógica de negocio.
- La verificación del JWT recibe `clockTimestamp: Math.floor(clock.now().getTime() / 1000)`.
- En la app no hace falta reloj inyectable: la expiración la decide el servidor. Los tests de RNTL simulan las respuestas 401.

### Por qué

Los tests de CA-001-14, -16, -17, -23, -26 y -27 recorren minutos, horas o 30 días. Con el reloj falso son instantáneos y deterministas. El reloj de la BD se descarta porque no se puede controlar desde el test.

### Alternativas consideradas

- **`jest.useFakeTimers()` con fecha simulada**: no afecta a Postgres ni a librerías nativas, y mezcla temporizadores con fechas. Es frágil en tests e2e con Supertest.
- **Pasar `now` como argumento a cada función**: es explícito, pero contamina todas las firmas. Inyectarlo por DI es lo idiomático en NestJS.

---

## R8. Igualdad de respuesta en ingreso fallido (FR-015, SC-006)

### Decisión

- Si el email no existe, se verifica la contraseña contra un **hash ficticio** (argon2id con los mismos parámetros, generado una vez al arrancar). El costo de CPU y memoria es el mismo que con un email existente.
- Ambos casos pasan por el mismo camino: advisory lock, contador por email (que existe aunque no haya cuenta), evento por origen y la misma respuesta `401 INVALID_CREDENTIALS` con el mismo cuerpo.
- La espera por límite también es idéntica para emails existentes e inexistentes (CA-001-14).
- Test: compara cuerpos byte a byte y comprueba que la mediana de N tiempos de respuesta no difiere más de un umbral (por ejemplo, 20 %) entre ambos casos.

### Alternativas consideradas

- **Retardo artificial fijo**: no iguala el trabajo de CPU, que se puede medir bajo carga, y alarga todos los ingresos.

---

## R9. Registro de actividad sin credenciales (FR-029, CA-001-25)

### Decisión

- `Logger` integrado de NestJS más un `RequestLoggingInterceptor` que registra solo método, ruta **plantilla** (`/v1/auth/login`, no la URL con query), código de estado, duración y `requestId`. **Nunca** cuerpo, cabeceras ni query.
- El log de Prisma solo registra `warn` y `error`, nunca `query` (incluiría parámetros).
- El filtro global de errores registra el `code` y el stack de los errores 5xx, nunca el cuerpo de la solicitud.
- CA-001-25: el test e2e captura la salida del logger durante registro, ingreso exitoso, ingreso fallido y logout. Luego busca la contraseña, el token de acceso, el token de renovación y el email usados, y verifica que no aparezcan.

### Alternativas consideradas

- **pino / nestjs-pino con `redact`**: es útil a más escala, pero es una dependencia más. Con "no registrar cuerpos ni cabeceras" no hay nada que redactar. Se puede reconsiderar en Deploy si hace falta log estructurado JSON.

---

## R10. Monorepo, app y CI

### Decisión

- **npm workspaces** (`apps/api`, `apps/mobile`). No agrega herramientas, Expo los soporta de forma nativa y el CI solo necesita `npm ci`.
- **Node 24 LTS**, **PostgreSQL 18**, **TypeScript `strict: true`** en ambos proyectos.
- **App**: Expo (SDK estable vigente al crear el esqueleto) con **Expo Router**:
  - Grupo `(auth)` (ingreso, registro) y grupo `(app)` (inicio con "Cerrar sesión"), con redirección según el estado de la sesión en Zustand.
  - **TanStack Query** para las mutaciones (register, login, logout) y la consulta `me`.
  - **expo-secure-store** para los tokens (FR-030). Sin NetInfo: "sin conexión" se detecta cuando `fetch` lanza un error de red.
- **Sin paquete compartido** de tipos entre API y app (YAGNI). El contrato es [contracts/openapi.yaml](./contracts/openapi.yaml) y la app declara sus tipos a mano a partir de él.
- **CI mínimo (GitHub Actions)**, en cada PR y push a `develop` o `main`:
  1. `api`: `npm ci` → lint → typecheck → `prisma migrate deploy` contra un servicio Postgres del job → tests unitarios + e2e (Supertest) con umbral de cobertura del 80 % en `src/**/*.service.ts`.
  2. `mobile`: lint → typecheck → Jest + RNTL.
  3. `audit`: `npm audit --audit-level=high` (falla con vulnerabilidades altas o críticas, principio V).
- **Maestro** queda fuera del CI en esta feature: los flujos viven en `apps/mobile/.maestro/` y se ejecutan en local durante Verify. Ver la justificación en el Constitution Check (es un "DEBERÍA").

### Alternativas consideradas

| Alternativa | Por qué se descarta |
|---|---|
| **pnpm / Turborepo / Nx** | Herramientas adicionales. pnpm con Expo requiere `node-linker=hoisted`. Con dos proyectos, npm workspaces basta. |
| **React Navigation directo** | Expo Router es el estándar actual de Expo y da rutas tipadas y redirecciones declarativas. |
| **react-hook-form + zod en la app** | Dos formularios de dos campos. `useState` alcanza (YAGNI). |
| **`@nestjs/config`** | Una función `loadConfig()` propia que valida `process.env` al arrancar es suficiente con 5 variables. |
| **Maestro en CI con emulador** | Los emuladores en GitHub Actions son lentos e inestables. Se reevalúa con EAS Workflows o Maestro Cloud (de pago, excluido por ahora). |
