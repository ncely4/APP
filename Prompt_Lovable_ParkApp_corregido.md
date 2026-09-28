# PROMPT PARA LOVABLE — ParkApp (Gestión inteligente de parqueaderos comunales)

> Cómo usarlo: pega el **PROMPT MAESTRO** (Parte 1) en Lovable. Cuando termine de construir, pega los **PROMPTS DE SEGUIMIENTO** (Parte 2) uno por uno, en orden. Lovable rinde mejor construyendo por fases que con un solo prompt gigante.

---

# PARTE 1 — PROMPT MAESTRO

## 1. Rol y objetivo

Actúa como un arquitecto de software y desarrollador full-stack senior. Construye **ParkApp**, una aplicación web (responsive, usable desde celular y computador) para la **gestión de cupos de parqueaderos comunales** de un conjunto residencial en Engativá, Bogotá (Colombia): la Asociación de Unidades Residenciales Santa Cecilia (**AUSACE**), Zona G, lotes 12 a 15.

Problema que resuelve: hoy la asignación de cupos y el control del dinero se llevan a mano en Excel y Word, lo que genera cupos duplicados, errores de cálculo, reparto injusto entre anillos y desconfianza en la administración.

La app cubre tres ejes: **gestión de cupos**, **control financiero** y **comunicación con residentes**.

Toda la interfaz debe estar **en español (Colombia)**: fechas `DD/MM/AAAA`, moneda en pesos colombianos (`$50.000`), textos claros y sin jerga técnica. Todas las fechas y cálculos por mes usan la zona horaria **America/Bogota**.

## 2. Stack técnico

- React + Vite + TypeScript + Tailwind CSS + shadcn/ui.
- Backend: Lovable Cloud / Supabase (PostgreSQL, Auth, Storage, Edge Functions, tareas programadas).
- Seguridad: Row Level Security (RLS) en TODAS las tablas. Los roles van en una tabla separada `user_roles` (nunca en el perfil del usuario), con una función `has_role()` de tipo SECURITY DEFINER.
- Gráficos: Recharts.
- Reporte Word: librería `docx` en el cliente para generar el archivo `.docx` descargable.
- IA: Lovable AI Gateway (modelo de visión principal `google/gemini-2.5-flash`, con respaldo automático a `openai/gpt-5-mini` del mismo gateway; si alguno ya no está disponible en el gateway, usa el equivalente vigente más cercano y dime cuál usaste — ver sección 8.0) invocada **solo desde Edge Functions**, nunca desde el navegador. No pidas ni expongas API keys en el código del cliente ni integres proveedores de IA distintos a los del gateway de Lovable.

## 3. Roles y permisos

| Rol | Puede hacer |
|---|---|
| `administrador` | Todo: CRUD de cupos, configuración, importar/exportar, ver alertas, escáner de placas, gestionar usuarios y roles |
| `tesorero` | Registrar pagos, ver módulo financiero y reportes, enviar recordatorios de pago, consultar cupos (no elimina cupos) |
| `guarda` | Solo el escáner de placas, registrar visitas y consultar si una placa está autorizada |
| `residente` | Solo lectura: ver sus propios cupos, su estado de pago y solicitar turno de cupo rotativo |

Autenticación con correo y contraseña.

- El **primer usuario** registrado queda como `administrador`. **Después de ese primero, el registro público se desactiva**: los demás usuarios solo entran por invitación del administrador (el administrador crea la cuenta o envía la invitación y asigna el rol).
- Pantalla "Usuarios y roles" donde el administrador asigna roles y **vincula a cada residente con sus cupos** (llenando `cupos.user_id`). Un residente sin cupos vinculados ve un mensaje claro: "Aún no tienes cupos asociados. Comunícate con la administración."

## 4. Modelo de datos (PostgreSQL)

**cupos** — un registro por vehículo/cupo:
`id`, `anillo` (int), `apto` (text), `placa` (text, MAYÚSCULAS sin espacios), `nombre`, `tipo_identificacion` (CC | NIT | C.EXTRANJERIA | T.IDENTIDAD), `numero_identificacion`, `tipo_tenencia` (PROPIETARIO | ARRENDATARIO), `tipo_cupo` (FIJO | SEGUNDO CUPO | ROTATIVO), `correo`, `telefono` (10 dígitos, inicia en 3, opcional), `estado` (ACTIVO | INACTIVO | ELIMINADO), `cuota_mensual` (numeric), `estado_pago` (AL_DIA | PENDIENTE | MOROSO | NO_APLICA), `fecha_ultimo_pago` (date, nullable), `comentarios`, `fecha_registro`, `fecha_eliminacion`, `user_id` (opcional, vincula al residente).

**visitas** — los visitantes NO son cupos (así una placa puede visitar varias veces sin chocar con la regla de placa única):
`id`, `placa`, `anillo`, `apto` (apartamento que recibe la visita), `nombre_visitante` (opcional), `lectura_id` (nullable, referencia a `lecturas_placa`), `fecha_ingreso`, `fecha_salida` (nullable), `registrado_por`.

**solicitudes_turno** — cola de solicitudes para cupos rotativos:
`id`, `anillo`, `apto`, `placa`, `solicitado_por`, `fecha_solicitud`, `estado` (EN_ESPERA | ASIGNADO | CANCELADO), `fecha_asignacion` (nullable), `cupo_id` (nullable, el cupo rotativo asignado).

**configuracion_anillos**: `anillo`, `num_apartamentos`, `comentarios`, `fecha_actualizacion`.
**configuracion_general**: `total_cupos` (int), `total_cupos_rotativos` (int), `cuota_fijo` (default 50000), `cuota_segundo_cupo` (default 50000), `cuota_rotativo` (default 30000), `max_cupos_por_apto` (default 2), umbrales del detector de abuso (ver sección 6.4).
**movimientos** (bitácora de auditoría, solo inserción): `id`, `cupo_id`, `accion` (CREAR | EDITAR | ELIMINAR | PAGO | IMPORTAR | ASIGNAR_TURNO | NOTIFICAR), `justificacion`, `usuario_id`, `datos_antes` (jsonb), `datos_despues` (jsonb), `fecha`.
**pagos**: `id`, `cupo_id`, `periodo` (AAAA-MM), `valor`, `fecha_pago`, `registrado_por`.
**lecturas_placa**: `id`, `imagen_path`, `placa_detectada`, `placa_confirmada`, `confianza` (0–1), `tipo_vehiculo`, `modelo_usado`, `resultado` (AUTORIZADA | NO_REGISTRADA | INACTIVA | EN_MORA | ILEGIBLE), `cupo_id` (nullable), `corregida_por_usuario` (bool), `usuario_id`, `fecha`. **Cada lectura confirmada cuenta como un ingreso del vehículo.**
**notificaciones**: `id`, `cupo_id`, `canal` (CORREO | WHATSAPP), `motivo` (RECORDATORIO_PAGO | MORA | TURNO_ASIGNADO | OTRO), `mensaje`, `enviado_por`, `fecha`.
**alertas_uso_indebido**: `id`, `tipo_regla`, `placa`, `cupo_id`, `detalle`, `estado` (ABIERTA | REVISADA | DESCARTADA), `comentario_revision`, `fecha`.

Datos semilla (editables): anillos 12, 13, 14 y 15 con 34, 30, 25 y 20 apartamentos; `total_cupos` = 90; `total_cupos_rotativos` = 10; ~40 cupos de ejemplo **totalmente ficticios** (nombres, placas y teléfonos inventados, con mezcla de estados y de estados de pago), ~15 visitas, ~5 solicitudes de turno en espera, y algunas lecturas de placa, de forma que el detector de uso indebido genere al menos una alerta de cada regla.

## 5. Módulos y pantallas

1. **Login, usuarios y roles** (invitación, asignación de rol, vínculo residente–cupos).
2. **Dashboard** (ver sección 7).
3. **Cupos**: tabla con búsqueda, filtros (anillo, tipo, estado, estado de pago) y paginación; botón "Nuevo vehículo"; editar; eliminar; interruptor "Mostrar eliminados (historial)".
4. **Escáner de placas con IA** (ver sección 8) — módulo estrella.
5. **Visitas**: registro y listado de visitantes por fecha, placa y apartamento.
6. **Rotación de cupos compartidos** (ver 6.3).
7. **Alertas de uso indebido** (ver 6.4).
8. **Finanzas**: registrar pago, cartera por estado, recaudo esperado vs. recaudado del mes, exportar reporte.
9. **Comunicación con residentes** (ver 6.6).
10. **Reporte en Word** (ver 6.7).
11. **Configuración**: número de apartamentos por anillo, total de cupos, cupos rotativos, cuotas por tipo, máximo de cupos por apartamento, umbrales de alertas.
12. **Importar / Exportar CSV**.
13. **Historial de movimientos** (auditoría, solo lectura, filtrable).

## 6. Reglas de negocio (obligatorias)

### 6.1 Validaciones de registro
- La **placa** debe ser única entre cupos activos. Normalízala (mayúsculas, sin espacios ni guiones). Formatos válidos en Colombia: carro `ABC123` (3 letras + 3 números), moto `ABC12D` (3 letras + 2 números + 1 letra) y moto antigua `ABC12` (3 letras + 2 números). Si es duplicada, muestra el error indicando en qué anillo/apto ya está activa.
- **El primer cupo de un apartamento es FIJO. Un apartamento no puede tener dos cupos FIJO.** Si el apartamento ya tiene un cupo y se registra otra placa distinta, esa placa solo puede ser de tipo `SEGUNDO CUPO` o `ROTATIVO`: en ese caso **autocompleta** los datos del residente (nombre, identificación, correo, teléfono, anillo, apto) desde el cupo existente; solo se piden la placa nueva y el tipo.
- Un apartamento no puede superar `max_cupos_por_apto` cupos activos; si se intenta, bloquea el registro con un mensaje claro.
- Todos los nombres en MAYÚSCULAS; el anillo y el apto deben existir en la configuración.
- **Nunca se borra un registro de verdad**: "eliminar" = marcar `ELIMINADO` (borrado lógico) y **exigir una justificación de mínimo 5 caracteres**. Toda edición o eliminación queda en `movimientos` con antes/después. La justificación no es editable después.

### 6.2 Reparto equitativo por anillo (equidad ESPACIAL)
Implementa el **método del residuo mayor (Hamilton)** para calcular cuántos cupos le corresponden a cada anillo:
1. `cupo_exacto = (aptos_del_anillo / total_aptos) × total_cupos`.
2. Cada anillo recibe primero la parte entera (`floor`).
3. Los cupos que sobran se entregan a los anillos con mayor parte decimal.

Muestra en el dashboard, por anillo: cupos equitativos, cupos activos, diferencia y semáforo (verde = dentro del reparto, rojo = excedido). Al registrar un cupo FIJO en un anillo que ya alcanzó o superó su cupo equitativo, **muestra una advertencia**; el administrador puede continuar solo escribiendo una justificación (queda en la bitácora).

### 6.3 Rotación equitativa (equidad TEMPORAL)
Los apartamentos (o el administrador en su nombre) crean solicitudes en `solicitudes_turno`. Cuando se libera un cupo rotativo, la pantalla **"Cola de turnos"** lista las solicitudes `EN_ESPERA` **ordenadas por antigüedad de uso**:
1. Primero quien nunca ha tenido un cupo rotativo.
2. Luego quien hace más tiempo no lo usa, tomando la fecha más reciente entre: `fecha_eliminacion` de sus cupos rotativos anteriores, `fecha_asignacion` de sus turnos anteriores y sus lecturas de placa como rotativo.
3. En empate, gana la `fecha_solicitud` más antigua.

Cada fila muestra la fecha del último uso y una explicación de por qué le toca ("hace 45 días sin cupo rotativo" o "nunca ha tenido cupo rotativo"). El botón **"Asignar turno"** del primero de la fila crea el cupo ROTATIVO, marca la solicitud como `ASIGNADO` y registra `ASIGNAR_TURNO` en la bitácora. Saltarse el orden exige justificación.

### 6.4 Detección de uso indebido (auditoría inteligente)
Una Edge Function (programada cada día y también ejecutable con un botón "Analizar ahora") aplica estas reglas y crea filas en `alertas_uso_indebido` (sin duplicar alertas ya abiertas):
1. **Visitante recurrente**: una misma placa con más de **N registros en `visitas` en 30 días** (N configurable, por defecto 3) → posible cupo fijo disfrazado.
2. **Cupos de más**: un apartamento con más cupos activos que `max_cupos_por_apto`.
3. **Eliminar y recrear**: un cupo eliminado y vuelto a crear con la misma placa en menos de **7 días** (configurable) → posible intento de esquivar la trazabilidad.
4. **Mora con ingreso**: placa de un cupo en estado `MOROSO` con al menos una lectura confirmada en `lecturas_placa` en los últimos **7 días** (configurable).

Cada alerta se puede marcar como Revisada o Descartada con un comentario obligatorio.

### 6.5 Finanzas
`cuota_mensual` según tipo: FIJO $50.000, SEGUNDO CUPO $50.000, ROTATIVO $30.000 (cuotas editables en configuración). Estados de pago:
- `AL_DIA`: pagó el mes actual.
- `PENDIENTE`: pagó hasta el mes anterior.
- `MOROSO`: más de un mes sin pagar.
- `NO_APLICA`: cupos eliminados o inactivos.

El `estado_pago` se recalcula **en dos momentos**: (a) al registrar un pago, y (b) con una **tarea programada diaria** (a las 00:05 hora de Bogotá), para que al cambiar de mes los cupos pasen solos de AL_DIA a PENDIENTE y de PENDIENTE a MOROSO aunque nadie haya registrado nada.

### 6.6 Comunicación con residentes
Sin integrar proveedores externos ni API keys:
- **WhatsApp**: botón que abre un enlace `https://wa.me/57XXXXXXXXXX?text=...` con el mensaje ya redactado (solo si el cupo tiene teléfono válido).
- **Correo**: botón que abre un enlace `mailto:` con asunto y cuerpo ya redactados.
- Plantillas editables: recordatorio de pago, aviso de mora (con el valor adeudado y los meses) y turno rotativo asignado.
- Acción masiva "Recordar a morosos": lista todos los cupos morosos con su botón de WhatsApp/correo para enviarlos uno por uno.
- Cada envío queda en `notificaciones` y en `movimientos` (acción `NOTIFICAR`).

### 6.7 Reporte en Word
Botón "Generar reporte (.docx)" en Finanzas y Dashboard que descarga un documento con: fecha y periodo, resumen de cupos por anillo y tipo, reparto equitativo vs. actual, estado de cartera y recaudo del mes, lista de morosos, alertas abiertas y movimientos del periodo. Formato limpio, en español.

## 7. Dashboard (12 indicadores, todos calculados en tiempo real)

1. Total de cupos activos (tarjeta KPI).
2. Cupos por anillo (barras).
3. Índice de ocupación por anillo = cupos asignados ÷ apartamentos (barras comparativas) y reparto equitativo (residuo mayor) vs. actual.
4. Distribución por tipo de cupo (dona).
5. Distribución por tenencia propietario/arrendatario (dona).
6. Alertas de duplicados (tarjeta + tabla).
7. Apartamentos con más de un cupo (tabla).
8. Estado de los cupos: activo / inactivo / eliminado (dona).
9. Movimientos por mes con su motivo (líneas + tabla del historial).
10. Cobertura de contacto: % de registros con teléfono válido (KPI).
11. Estado de cartera: al día / pendiente / moroso (dona).
12. Recaudo mensual: esperado vs. recaudado (tarjeta comparativa).

Extras: alertas de uso indebido abiertas, solicitudes de turno en espera, visitas del mes, y **precisión del lector de placas** (% de lecturas que el usuario no tuvo que corregir).

## 8. MÓDULO DE IA: RECONOCIMIENTO AUTOMÁTICO DE PLACAS (ALPR)

Es la función estrella de ParkApp. Construye lo siguiente.

### 8.0 Herramienta de IA elegida (y por qué)

Usa **Lovable AI Gateway con `google/gemini-2.5-flash` como modelo principal y `openai/gpt-5-mini` como respaldo automático** (o sus equivalentes vigentes en el gateway si alguno ya no está disponible). Ambos se llaman igual (mismo gateway, mismo formato de salida estructurada), así que el respaldo es solo un segundo intento, no una integración distinta. No integres ningún otro proveedor de IA para este módulo. Guarda en `lecturas_placa.modelo_usado` cuál modelo dio la respuesta final.

Justificación (déjala tal cual, no la cambies por otra herramienta):
- **Ya está integrado en Lovable, sin cuenta ni tarjeta aparte.** No hay que crear una cuenta nueva, pagar una mensualidad a un tercero ni pedir una API key adicional — es coherente con el diferencial de "licencia única, sin depender de un proveedor externo" que ya sustenta el proyecto.
- **Soporta visión + salida estructurada (JSON/tool calling)** de forma nativa, que es exactamente lo que necesita el post-procesamiento (placa, tipo de vehículo, confianza, legibilidad).
- **Es rápido y económico** para una tarea simple de lectura de texto en una foto (no se necesita un modelo grande ni costoso); Gemini Flash está pensado justo para este tipo de tareas de alto volumen y baja latencia.
- **Con respaldo automático a un segundo modelo del mismo gateway**, se gana tolerancia a fallos sin duplicar integraciones.

Por qué se descartaron otras opciones:
- **APIs especializadas en placas (Plate Recognizer, OpenALPR, Regula, etc.):** mayor precisión "de fábrica" para placas, pero cobran una mensualidad o por consulta y exigen una cuenta y una API key de un proveedor externo — contradice directamente el argumento de "bajo costo total, sin depender de terceros" que ya está en el documento del proyecto.
- **APIs de visión genéricas fuera de Lovable (Google Cloud Vision, AWS Rekognition, Azure AI Vision):** son OCR genérico, no reconocen placas como concepto (habría que programar más lógica para aislar el texto de la placa), y exigen crear y facturar una cuenta cloud aparte, con sus propias credenciales que hay que proteger.
- **Integrar Claude (Anthropic) u otro proveedor por fuera del gateway de Lovable:** viable técnicamente, pero obliga a gestionar una API key propia por fuera de Lovable, guardarla como secreto, y mantener una integración manual — más trabajo y un proveedor más que administrar, sin ganancia real de precisión para esta tarea.

Si en la sustentación te preguntan "¿por qué esta IA y no otra?", la respuesta corta es: **porque es la que ya viene dentro de la misma plataforma que construyó el proyecto, no agrega un proveedor ni un costo nuevo, y es suficientemente precisa para leer una placa en una foto clara — que es exactamente lo que la app necesita, ni más ni menos.**

### 8.1 Pantalla "Escáner de placas"
Optimizada para celular (la usa el guarda en la portería):
- Botón grande **"Tomar foto"** (abre la cámara trasera con `<input type="file" accept="image/*" capture="environment">`) y opción **"Subir imagen"** desde la galería. Muestra vista previa antes de enviar y un indicador de carga.
- Esta misma acción debe estar disponible como botón **"Escanear placa"** dentro del formulario "Nuevo vehículo" y del formulario de visitas, para autocompletar el campo placa.

### 8.2 Edge Function `reconocer-placa`
1. Recibe la imagen (sube primero al bucket privado `placas` de Storage y trabaja con la ruta/URL firmada). Rechaza archivos que no sean imagen o superen 5 MB; comprime/redimensiona en el cliente a máx. 1280 px antes de subir.
2. Llama al modelo principal con un prompt de sistema como este:
   *"Eres un lector de placas vehiculares colombianas. Analiza la imagen y devuelve SOLO la placa del vehículo principal. Formatos válidos: carro ABC123 (3 letras + 3 números), moto ABC12D (3 letras + 2 números + 1 letra) o moto antigua ABC12 (3 letras + 2 números). Si no es legible, indícalo; no inventes caracteres."*
   Usa **salida estructurada (tool calling / JSON schema)** con los campos: `placa` (string o null), `tipo_vehiculo` (CARRO | MOTO | OTRO), `confianza` (0 a 1), `legible` (boolean), `observaciones` (string corto).
   **Umbral único: 0.70.** Si el modelo principal falla, tarda más de ~8 s o devuelve confianza menor a 0.70, reintenta **una vez** con el modelo de respaldo. Si después del respaldo la confianza sigue por debajo de 0.70 o el formato es inválido, el resultado es `ILEGIBLE`. Si ambos modelos responden, quédate con la lectura de mayor confianza.
3. **Post-procesamiento en el servidor**: pasar a mayúsculas, quitar espacios y guiones; corregir confusiones típicas según la posición:
   - Posiciones 1 a 3 (siempre letras): `0→O`, `1→I`, `8→B`, `5→S`.
   - Posiciones 4 y 5 (siempre números): `O→0`, `I→1`, `B→8`, `S→5`.
   - Posición 6 (depende del vehículo): si `tipo_vehiculo` = CARRO, corrige a número; si = MOTO, corrige a letra; si no se sabe, no la corrijas.

   Luego valida con expresión regular: `^[A-Z]{3}\d{3}$` (carro), `^[A-Z]{3}\d{2}[A-Z]$` (moto) o `^[A-Z]{3}\d{2}$` (moto antigua). Si no cumple ningún formato, marca el resultado como no válido y baja la confianza por debajo de 0.70.
4. **Cruza la placa con la base de datos** y devuelve uno de estos resultados:
   - `AUTORIZADA` → cupo activo (muestra anillo, apto, nombre del responsable y tipo de cupo) con un aviso verde grande **"ACCESO AUTORIZADO"**.
   - `EN_MORA` → cupo activo pero moroso: aviso amarillo con el detalle de la deuda.
   - `INACTIVA` → existe pero está inactiva o eliminada: aviso rojo con el motivo del historial.
   - `NO_REGISTRADA` → no tiene cupo: aviso naranja con dos botones: **"Registrar visita"** (crea una fila en `visitas`, pide anillo y apto de destino, NO crea un cupo) y **"Crear cupo nuevo"** (precarga el formulario con la placa detectada). Si la placa ya tiene visitas previas, muestra cuántas lleva en los últimos 30 días.
   - `ILEGIBLE` → pide repetir la foto con consejos (más luz, más cerca, sin reflejos) y ofrece **ingreso manual**.
5. Guarda **siempre** la lectura en `lecturas_placa` (imagen, placa detectada, confianza, modelo usado, resultado, usuario).

### 8.3 Humano en el circuito
Antes de confirmar, muestra la placa detectada en un campo editable. Si el usuario la corrige, guarda `placa_confirmada`, marca `corregida_por_usuario = true` y vuelve a hacer el cruce con la placa corregida (así se mide la precisión real del modelo). Nunca bloquees el acceso ni tomes decisiones automáticas sin confirmación humana.

### 8.4 Historial de lecturas
Tabla con miniatura, placa, confianza, modelo usado, resultado, fecha y usuario, con filtros.

### 8.5 Privacidad (Ley 1581 de 2012 – Habeas Data)
Bucket privado con URLs firmadas de corta duración; eliminación automática de imágenes a los **30 días** con una tarea programada (dejando solo el texto de la placa en `lecturas_placa`); aviso visible de tratamiento de datos en el escáner, en el registro de cupos y en el de visitas; solo los roles administrador y guarda ven las imágenes.

### 8.6 Manejo de errores
Si la IA falla, hay límite de uso (429) o falta de crédito (402), muestra un mensaje claro en español y permite el ingreso manual de la placa. Registra el error, sin exponer detalles internos al usuario.

## 9. Importar / Exportar CSV
- **Exportar** todos los cupos a CSV con separador `;` (para que Excel en español lo separe en columnas), con codificación UTF-8 con BOM.
- **Importar**: detectar automáticamente el separador (`;`, `,` o tabulador), mapear encabezados sin importar mayúsculas/tildes/espacios, y hacer **upsert por placa**: si la placa ya existe activa, actualiza; si no, crea. Aplica las mismas validaciones de 6.1 (las filas que no cumplan se omiten y se listan con el motivo). Al final muestra "X creados, Y actualizados, Z omitidos". Todo queda en la bitácora con acción `IMPORTAR`.
- Botón **"Eliminar duplicados"**: para placas activas repetidas, conserva el registro más reciente y marca los demás como eliminados con justificación automática (borrado lógico, con confirmación previa).

## 10. Diseño y experiencia
- Diseño limpio y profesional, con **paleta de tonos azules** (azul principal + azul oscuro para acentos/botones primarios + neutros grises) y modo claro con opción de modo oscuro. No uses verde, teal ni morado como color principal de marca; el azul debe ser el color dominante en la barra lateral/superior, los botones primarios y los gráficos del dashboard (usa variaciones de azul para las series de las gráficas; reserva el rojo/ámbar solo para alertas y advertencias, y el verde solo para el aviso "ACCESO AUTORIZADO").
- Navegación lateral en escritorio y menú inferior en móvil. Todas las pantallas deben verse bien en 375 px de ancho.
- Estados vacíos, carga (skeletons), y errores siempre visibles y en español. Confirmación antes de acciones destructivas.
- Accesibilidad: contraste suficiente, etiquetas en los campos, foco visible.

## 11. Criterios de aceptación
- No es posible registrar una placa activa duplicada ni dos cupos FIJO en el mismo apartamento, ni superar el máximo de cupos por apartamento.
- Un segundo vehículo del mismo apartamento se registra como SEGUNDO CUPO con los datos del residente autocompletados.
- Eliminar exige justificación y queda en el historial; nada se borra físicamente.
- Solo el primer usuario se registra solo; los demás entran por invitación.
- Un `residente` solo ve sus propios cupos (verificado por RLS, no solo por la interfaz), y un residente sin cupos vinculados ve el mensaje correspondiente.
- El dashboard muestra los 12 indicadores con los datos semilla, incluido el reparto por residuo mayor.
- Subir la foto de una placa devuelve la placa detectada, la confianza y el resultado en menos de ~10 segundos.
- Una misma placa puede registrarse como visita varias veces sin error de duplicado.
- La cola de turnos ordena por antigüedad de uso y explica el porqué.
- Al cambiar de mes, los estados de pago se actualizan solos.
- El detector de uso indebido genera al menos una alerta de cada una de las 4 reglas con los datos semilla.
- Los botones de WhatsApp y correo abren el mensaje ya redactado y quedan registrados.
- El reporte Word se descarga y abre correctamente.

Comienza por la **Fase 1** (autenticación con registro por invitación, roles, vínculo residente–cupos, modelo de datos completo con RLS, módulo de Cupos con las validaciones de 6.1 y datos semilla). Al terminar, dime qué construiste y espera mi siguiente instrucción.

---

# PARTE 2 — PROMPTS DE SEGUIMIENTO (pégalos uno por uno)

## Fase 2 — Auditoría, finanzas, comunicación y CSV
> Construye ahora: (1) la bitácora `movimientos` conectada a toda creación/edición/eliminación con justificación obligatoria y la pantalla de historial; (2) el módulo de Finanzas (registrar pago, recálculo de `estado_pago` al pagar **y con la tarea programada diaria en hora de Bogotá**, recaudo esperado vs. recaudado); (3) el módulo de Comunicación con enlaces `wa.me` y `mailto:`, plantillas editables, "Recordar a morosos" y registro en `notificaciones`; (4) el reporte descargable en Word (.docx); (5) Importar/Exportar CSV con separador `;`, autodetección de delimitador, upsert por placa con validaciones y el botón "Eliminar duplicados". Todo según las secciones 6.5, 6.6, 6.7 y 9 del documento de requisitos.

## Fase 3 — Módulo de IA: reconocimiento de placas
> Construye ahora el **módulo de reconocimiento de placas con IA** (sección 8): la pantalla "Escáner de placas" optimizada para celular, el bucket privado `placas`, la Edge Function `reconocer-placa` usando Lovable AI Gateway con **`google/gemini-2.5-flash` como principal y `openai/gpt-5-mini` como respaldo** (o sus equivalentes vigentes; dime cuáles usaste), **umbral único de confianza 0.70**, salida estructurada, el post-procesamiento con corrección por posición (la posición 6 según tipo de vehículo) y regex de placas colombianas (carro, moto y moto antigua), el cruce con la tabla `cupos` con los 5 resultados posibles, "Registrar visita" guardando en la tabla `visitas` (no en `cupos`), la confirmación humana editable con nuevo cruce si se corrige, el guardado en `lecturas_placa`, el historial de lecturas, el manejo de errores 429/402 con ingreso manual de respaldo y la eliminación automática de imágenes a los 30 días. Agrega también el botón "Escanear placa" dentro de los formularios "Nuevo vehículo" y de visitas. Si algo requiere configuración que no puedes hacer tú (por ejemplo un secreto), dime exactamente qué debo hacer.

## Fase 4 — Equidad y detección de abuso (el diferencial)
> Construye ahora los tres mecanismos diferenciales: (1) el reparto equitativo por anillo con el **método del residuo mayor** y la advertencia al registrar un FIJO en un anillo excedido (6.2); (2) la **cola de turnos por antigüedad de uso** usando la tabla `solicitudes_turno`, con explicación de por qué le toca a cada uno y el botón "Asignar turno" (6.3); (3) la Edge Function del **detector de uso indebido** con las 4 reglas (visitas recurrentes desde la tabla `visitas`, cupos de más, eliminar y recrear, mora con ingreso según `lecturas_placa`), umbrales configurables, ejecución diaria y botón "Analizar ahora", más la pantalla de alertas con comentario obligatorio al revisar (6.4).

## Fase 5 — Dashboard y pulido
> Construye el Dashboard con los 12 indicadores más los extras (alertas abiertas, turnos en espera, visitas del mes y precisión del lector de placas), todos con datos reales de la base. Luego revisa: responsive en 375 px, modo oscuro, textos en español, estados vacíos y de carga, y que las políticas RLS impidan que un `residente` o un `guarda` accedan a datos fuera de su rol. Recorre uno por uno los criterios de aceptación de la sección 11 y dime cuáles cumple y cuáles no. Lista al final cualquier pendiente o limitación.

---

# NOTAS PARA TI (no se las pegues a Lovable)

- Lovable construye una **aplicación web en la nube** (no un programa de escritorio portable ni un APK offline). Para la entrega, el argumento de "sin internet" ya no aplica a esta versión; tu diferencial es el de los 3 pilares (reparto por residuo mayor, rotación temporal, detección de uso indebido) más el lector de placas con IA.
- Si Lovable se queda corto de créditos, pega las fases en orden y no todo junto.
- Para la demo del lector de placas, prueba con fotos **ficticias o de tu propio vehículo**; no subas fotos de placas de terceros.
- **Cambio de modelo de datos:** VISITANTE ya no es un tipo de cupo sino una tabla aparte (`visitas`). Los tipos de cupo quedan FIJO / SEGUNDO CUPO / ROTATIVO. Si tu Word dice fijo/rotativo/visitante, explícalo así: "el visitante no ocupa un cupo asignado, se registra como visita, y eso es lo que permite detectar a quien usa la figura de visitante para parquear todos los días".
- La comunicación por WhatsApp y correo se hace con enlaces que abren la app del celular con el mensaje listo: no cuesta nada y no depende de terceros, coherente con el resto del argumento. La limitación es que no es envío automático; alguien debe presionar "enviar".
- Si Lovable te dice que usó otro modelo de IA porque los nombrados ya no existen, actualiza la tabla de abajo con el nombre real antes de la sustentación.

## Tabla comparativa de IA para el módulo de placas (para tu sustentación)

Si la profe pregunta por qué esta IA y no otra, esta es la comparación completa que respalda la elección de **Gemini 2.5 Flash vía Lovable AI Gateway**:

| Opción | Precisión para leer placas | Costo | Integración con Lovable | ¿Coherente con "bajo costo, sin depender de terceros"? |
|---|---|---|---|---|
| **Gemini 2.5 Flash (Lovable AI) — elegida** | Alta en fotos legibles; modelo de visión reciente de uso general | Incluido en Lovable AI, muy económico para tareas simples | Nativa, sin API key ni cuenta aparte | Sí — mismo proveedor de toda la app |
| GPT-5-mini (Lovable AI) — respaldo | Similar a Gemini Flash | Incluido en Lovable AI | Nativa, mismo gateway | Sí |
| Plate Recognizer / OpenALPR (especializado en placas) | Muy alta, entrenado solo para placas | Mensualidad o pago por consulta, plan aparte | Requiere cuenta y API key externas | No — agrega una mensualidad y un proveedor externo |
| Google Cloud Vision / AWS Rekognition (OCR genérico) | Media — no distingue la placa del resto del texto de la foto | Pago por uso, cuenta cloud aparte | Requiere configurar credenciales fuera de Lovable | No |
| Claude / Anthropic (visión) por fuera del gateway | Alta | Requiere clave y facturación propias, aparte de Lovable | Requiere integración manual (Edge Function llamando a otra API) | Parcial — viable, pero suma un proveedor más que administrar |

**Conclusión para la sustentación:** se eligió Gemini 2.5 Flash porque ya viene dentro de la misma plataforma que construyó el proyecto (Lovable), no agrega cuentas, tarjetas ni proveedores nuevos, soporta salida estructurada de forma nativa, y es suficientemente preciso para una tarea acotada (leer una placa en una foto clara) — usar una API especializada y más costosa sería sobre-ingeniería para lo que el proyecto necesita.
