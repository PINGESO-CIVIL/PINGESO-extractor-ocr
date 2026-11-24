# Modos de Falla — Servicio Extractor OCR (Grupo 5 - PT5)

Documento de resiliencia y modos de falla del microservicio **Extractor OCR** de acuerdo con el estándar técnico **DAS (Desarrollo de Microservicios)**.

---

## 1. Alcance y Contexto del Servicio

* **Propósito:** Recibir imágenes digitalizadas (directamente o referenciadas desde la Bóveda Documental), procesarlas mediante extracción óptica de caracteres (OCR) y retornar el contenido estructurado en formato JSON.
* **Dependencias Externas:** Bóveda Documental (PT7).
* **Base de Datos Propia:** PostgreSQL 17 (almacenamiento de auditoría, trazabilidad y registro de extracciones).
* **Estándar de Errores:** Todos los errores retornan payloads estructurados conforme a **RFC 9457** (`application/problem+json`), sin exponer trazas de depuración, rutas internas ni detalles de infraestructura.

---

## 2. Matriz de Modos de Falla

| ID | Modo de Falla | Causa Raíz Potencial | Mitigación / Qué la Impide | Código HTTP / Comportamiento | Cómo se Verifica |
|:---|:---|:---|:---|:---:|:---|
| **MF-01** | Base de datos PostgreSQL no disponible o caída | Problemas de red, contenedor de base de datos detenido o saturación del pool de conexiones | Conexión con timeout explícito (5s), probes independientes (`/healthz` activo, `/readyz` reporta falla) y manejo seguro de errores sin exponer trazas SQL | `503 Service Unavailable` en `/readyz`<br>`500 Internal Server Error` (RFC 9457) en operaciones de datos | `GET /readyz` devuelve 503 cuando PostgreSQL está desconectado; pruebas de integración en `config/db_test.go` |
| **MF-02** | Formato de archivo de imagen no soportado o inválido | Cliente envía archivos binarios ejecutables, documentos de texto plano u otros formatos no soportados por el motor OCR | Validación temprana de cabeceras MIME y verificación de firmas/magic bytes del archivo (solo se aceptan `image/png`, `image/jpeg`, `image/tiff`, `image/webp`) | `415 Unsupported Media Type` con mensaje RFC 9457 | Prueba unitaria enviando archivo `.txt` o `.exe` a la ruta de extracción |
| **MF-03** | Tamaño de imagen excede el límite permitido (*Payload Too Large*) | Intento de subida de imágenes gigantes o ataques de saturación de memoria | Limitación estricta de lectura de cuerpo HTTP mediante `http.MaxBytesReader` (límite máximo: 10 MB) | `413 Content Too Large` | Prueba unitaria enviando un payload superior al umbral configurado |
| **MF-04** | Imagen corrupta, vacía o con cabeceras dañadas | Transferencia de red incompleta, archivo de 0 bytes o corrupción de mapa de bits | Decodificación y verificación de integridad previa en memoria antes de invocar la lógica de extracción; captura de errores sin provocar `panic` | `422 Unprocessable Content` (o `400 Bad Request`) | Prueba con archivo `.png` truncado o bytes aleatorios que no forman una imagen válida |
| **MF-05** | Timeout o agotamiento de tiempo en procesamiento OCR | Imágenes de muy alta densidad, saturación de CPU o bloqueo en el motor OCR | Uso obligatorio de `context.WithTimeout` (máximo 5 segundos por extracción). Cancelación inmediata del procesamiento si expira | `504 Gateway Timeout` / `500 Internal Server Error` (timeout de contexto) | Prueba con simulación de motor OCR demorando más de 5s y validación de cancelación de contexto |
| **MF-06** | Indisponibilidad o timeout de dependencia externa (Bóveda Documental - PT7) | Servicio PT7 caído, error de red interna o respuesta lenta al obtener el documento original | Cliente HTTP configurado con timeout estricto de 5 segundos, propagación de cabecera `X-Request-Id` y circuit-breaking/fallback claro sin colgar la petición del usuario | `502 Bad Gateway` / `503 Service Unavailable` | Prueba de integración con mock HTTP de PT7 simulando retardo superior a 5s y error 500 |
| **MF-07** | Inyección SQL en metadatos o identificadores de documentos | Entrada maliciosa del usuario en campos de texto, nombres de archivo o etiquetas | Sentencias SQL 100% parametrizadas con marcadores nativos (`$1, $2, ...`) mediante driver `pgx/v5` / `sqlc` | Prevención total a nivel de driver; no hay concatenación de strings en queries | Verificación estática con SonarQube y pruebas unitarias con cargas de inyección SQL (`' OR '1'='1`) |
| **MF-08** | Peticiones CORS preflight no autorizadas | Solicitudes OPTIONS o cross-origin desde dominios o puertos no autorizados | Middleware centralizado de cabeceras CORS (`setCORS`) restringido estrictamente al origen configurado en `FRONTEND_URL` | `200 OK` con cabeceras correctas solo para orígenes autorizados; rechazo implícito por navegador si el origen no coincide | Prueba de petición `OPTIONS` y `GET` simulando cabecera `Origin` ajena |

---

## 3. Estrategia de Salud y Observabilidad

1. **Liveness Probe (`GET /healthz`):**
   * Determina si el proceso de la API está vivo y sirviendo peticiones HTTP.
   * Retorna `200 OK` (`ok`). No consulta dependencias externas ni bases de datos.

2. **Readiness Probe (`GET /readyz`):**
   * Verifica la conectividad activa con PostgreSQL con un timeout estricto de 2 segundos (`PingContext`).
   * Retorna `200 OK` (`ready`) si la base está operativa.
   * Retorna `503 Service Unavailable` si PostgreSQL no responde o el pool no puede entregar una conexión en el tiempo límite.

3. **Logs Estructurados y Auditoría:**
   * Utilización de `log/slog` en formato JSON a `stdout`.
   * Todo log incluye: `servicio="extractor-ocr"`, `version`, `request_id`, `metodo`, `ruta`, `estado` y `duracion_ms`.
   * Queda estrictamente prohibido registrar tokens, secretos o el payload binario completo de las imágenes.
