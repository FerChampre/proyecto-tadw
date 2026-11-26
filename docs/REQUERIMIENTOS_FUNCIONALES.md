# Especificación de Requerimientos Funcionales — DCIC-Flix26

**Proyecto:** Plataforma de Catálogo Cinematográfico con Microservicios, Agentes de Calidad de Datos con LLM y Motor de Recomendaciones (**DCIC-Flix26**)  
**Cátedra:** Tópicos Avanzados de Desarrollo Web — Edición 2026 (DCIC - UNS)  
**Fuentes base:**
- Enunciado oficial: [ENUNCIADO_PROYECTO_FINAL_TADW_2026.md](file:///c:/Users/ferna/Desktop/Topicos/proyecto/ENUNCIADO_PROYECTO_FINAL_TADW_2026.md)
- Arquitectura y microservicios definidos: [Diagrama TADW.drawio](file:///c:/Users/ferna/Desktop/Topicos/proyecto/Diagrama%20TADW.drawio)

---

## 1. Alcance y Mapa de Componentes del Sistema

Este documento formaliza **todos los requerimientos funcionales (RF)** de **DCIC-Flix26**, respetando estrictamente los microservicios, componentes de infraestructura y eventos definidos en [Diagrama TADW.drawio](file:///c:/Users/ferna/Desktop/Topicos/proyecto/Diagrama%20TADW.drawio) y las exigencias funcionales y de evaluación de [ENUNCIADO_PROYECTO_FINAL_TADW_2026.md](file:///c:/Users/ferna/Desktop/Topicos/proyecto/ENUNCIADO_PROYECTO_FINAL_TADW_2026.md).

### 1.1 Componentes y Microservicios (`Diagrama TADW.drawio`)

| Componente | Tipo | Responsabilidad Funcional Principal | Prioridad |
| :--- | :--- | :--- | :--- |
| **`frontend`** | Aplicación Web | Interfaz de exploración del catálogo, selector de perfil (`usr_0001`–`usr_2000`), lectura/emisión de reseñas, visualización de recomendaciones, manejo visual de *Cold Start*, ruta oculta `/audit` y capacidades LLM de interfaz. | Alta / Obligatorio (con funciones LLM deseables) |
| **`ms_backend`** | Microservicio Gateway / BFF | Punto único de entrada para el `frontend`; media consultas hacia `catalog_api`, `ms_Recomendador` y `ms_auditor`; valida y publica votos/reseñas en `RabbitMQ`; gestiona el estado operativo (*Cold* vs. *Warm Start*). | Alta / Obligatorio |
| **`catalog_api`** | Microservicio de Catálogo e Ingesta | Expone consultas sobre películas y reseñas desde `MongoDB`, provee el catálogo saneado a `ms_Recomendador` y consume de forma asincrónica el evento/cola `reseña` desde `RabbitMQ` para persistir interacciones. | Alta / Obligatorio |
| **`ms_Sanitizador`** | Agente Worker de Curación | Detecta el estado crudo del dataset al arranque, ejecuta reglas determinísticas de limpieza en `MongoDB`, coordina la resolución semántica con `IA_sanitizador`, emite eventos `evento-reg-audit` y notifica `evento_sanitizacion_lista`. | Alta / Obligatorio |
| **`IA_sanitizador`** | Servicio / Adaptador LLM | Modelo de lenguaje (local vía Ollama/SLM o API externa) invocado por `ms_Sanitizador` para resolver ambigüedades semánticas (ej. inferencia de rating 1–5 estrellas en reseñas con `rating: null`). | Alta / Obligatorio |
| **`ms_auditor`** | Microservicio de Auditoría | Consume `evento-reg-audit` desde `RabbitMQ`, persiste trazabilidad detallada en `registros-auditoria` (`MongoDB`) y expone métricas macro y detalle antes/después para `/audit`. | Alta / Obligatorio |
| **`ms_Recomendador`** | Motor de Recomendaciones | Se activa ante `evento_sanitizacion_lista` (o carga estado previo en *Warm Start*), ingesta datos curados desde `catalog_api`, precomputa modelos/índices en memoria, emite `evento_recomendador_listo` y resuelve recomendaciones en `< 100 ms`. | Alta / Obligatorio |
| **`MongoDB`** | Base de Datos Documental | Imagen oficial Docker Hub. Aloja la BD `peliculas-proyectos` con `movies` (~20.000), `reviews` (~61.000) y `registros-auditoria` (además de estado de curación). | Alta / Obligatorio |
| **`RabbitMQ`** | Message Broker (`RabitMQ` en diagrama) | Imagen oficial Docker Hub. Desacopla la ingesta de `reseña`, el registro de `evento-reg-audit` y la coordinación del ciclo de vida (`evento_sanitizacion_lista`, `evento_recomendador_listo`). | Alta / Obligatorio |

> **Nota sobre identificadores en `Diagrama TADW.drawio`:** En el archivo `.drawio` el broker figura rotulado como `RabitMQ` y el evento de finalización del recomendador como `evento_rodomendador_listo`. En esta especificación se normalizan como `RabbitMQ` y `evento_recomendador_listo` manteniendo trazabilidad directa con los nodos y aristas del diagrama.

---

## 2. Requerimientos Funcionales por Subsistema y Microservicio

### 2.1 Orquestación, Infraestructura y Ciclo de Vida (`RF-CIC`)

| ID | Requerimiento | Prioridad | Microservicios / Actores |
| :--- | :--- | :--- | :--- |
| **RF-CIC-01** | **Arranque automatizado integral sin intervención manual:** El sistema completo debe levantarse, inicializar la base de datos con el dataset semilla, declarar automáticamente exchanges/colas en `RabbitMQ`, ejecutar la curación e indexación inicial y quedar operativo ejecutando únicamente `docker compose up --build`, sin migraciones ni pasos manuales posteriores. | Alta / Obligatorio | Todos los servicios (`docker-compose.yml`), `MongoDB`, `RabbitMQ` |
| **RF-CIC-02** | **Inicialización automática de datos semilla en `MongoDB`:** En el primer arranque con volúmenes vacíos, el contenedor oficial de `MongoDB` debe ejecutar automáticamente los scripts en `/docker-entrypoint-initdb.d/` (carpeta `dataset/`) para importar las colecciones iniciales `movies` (~20.000 documentos) y `reviews` (~61.000 documentos sobre `usr_0001` a `usr_2000`) en la base de datos `peliculas-proyectos`. | Alta / Obligatorio | `MongoDB`, `dataset/` |
| **RF-CIC-03** | **Orquestación de Arranque en Frío (*Cold Start*):** Ante una base recién inicializada con datos crudos, el sistema debe ejecutar automáticamente la secuencia coordinada: (1) `ms_Sanitizador` inicia limpieza determinística + inferencia con `IA_sanitizador` y reporta cambios por `evento-reg-audit` a `ms_auditor`; (2) `frontend` y `ms_backend` permiten explorar el catálogo pero restringen recomendaciones y nuevas reseñas informando estado transitorio; (3) al finalizar, `ms_Sanitizador` publica `evento_sanitizacion_lista`; (4) `ms_Recomendador` toma el evento, lee el catálogo limpio desde `catalog_api`, indexa/entrena sus estructuras y publica `evento_recomendador_listo`; (5) `ms_backend` y `frontend` desbloquean automáticamente todas las funcionalidades. | Alta / Obligatorio | `MongoDB`, `ms_Sanitizador`, `IA_sanitizador`, `ms_auditor`, `ms_Recomendador`, `catalog_api`, `ms_backend`, `frontend`, `RabbitMQ` |
| **RF-CIC-04** | **Orquestación de Segundo Arranque (*Warm Start*):** Al reiniciar la plataforma (`docker compose up`) con volúmenes Docker persistidos y datos previamente saneados, todos los microservicios deben detectar el estado saneado previo, omitir la re-ejecución del proceso pesado de limpieza y la barrida inicial completa del recomendador, y quedar plenamente operativos en pocos segundos con el `frontend` desbloqueado desde el primer instante. | Alta / Obligatorio | `MongoDB`, `ms_Sanitizador`, `ms_Recomendador`, `ms_backend`, `frontend` |
| **RF-CIC-05** | **Aislamiento de red y mediación obligatoria de acceso a datos:** El contenedor `frontend` debe operar aislado de la red de datos: no debe tener conectividad de red hacia `MongoDB` ni capacidad de resolver su nombre de host. Toda consulta o mutación originada en `frontend` debe ser mediada por `ms_backend` (y a través de `catalog_api` / `RabbitMQ` / `ms_auditor`). | Alta / Obligatorio | `frontend`, `ms_backend`, `MongoDB` |
| **RF-CIC-06** | **Infraestructura base contenerizada oficial:** Tanto `MongoDB` como `RabbitMQ` deben desplegarse obligatoriamente mediante sus imágenes oficiales de Docker Hub dentro de `docker-compose.yml`, utilizando volúmenes persistentes para garantizar durabilidad entre reinicios. | Alta / Obligatorio | `MongoDB`, `RabbitMQ` |

**Criterios de aceptación clave (`RF-CIC`):**
- Ejecutar `docker compose down -v && docker compose up --build` deja el sistema 100 % funcional sin intervención humana.
- Un `docker compose restart` o `docker compose down && docker compose up` posterior (conservando volúmenes) habilita recomendaciones y votos en pocos segundos sin re-sanear ni re-entrenar todo el catálogo.
- Desde el contenedor `frontend`, cualquier intento de resolución DNS o conexión TCP al puerto de `MongoDB` falla a nivel de red.

---

### 2.2 `frontend` — Interfaz Web de Usuario y Ruta `/audit` (`RF-FE`)

| ID | Requerimiento | Prioridad | Microservicios / Actores |
| :--- | :--- | :--- | :--- |
| **RF-FE-01** | **Exploración del catálogo cinematográfico:** La interfaz web debe presentar un catálogo navegable/paginado de películas con carátulas (posters), filtros por género y barra de búsqueda por título, funcionando de manera responsiva y fluida. | Alta / Obligatorio | Usuario, `frontend`, `ms_backend`, `catalog_api` |
| **RF-FE-02** | **Ficha de detalle de película y lectura de críticas previas:** Al seleccionar una película, el `frontend` debe mostrar su ficha completa: título, año, duración, géneros, rating promedio / IMDb, director, reparto, sinopsis (`plot` / `fullplot`) y el listado navegable de reseñas y calificaciones históricas de la comunidad almacenadas en la base. | Alta / Obligatorio | Usuario, `frontend`, `ms_backend`, `catalog_api` |
| **RF-FE-03** | **Selector de identidad de usuario (`usr_0001` a `usr_2000`):** La interfaz debe ofrecer un selector simple (dropdown o buscador) que permita elegir el usuario activo del catálogo con el cual se interactúa, actualizando en consecuencia las recomendaciones personalizadas y asociando los nuevos votos/reseñas a dicho perfil. | Alta / Obligatorio | Usuario, `frontend`, `ms_backend` |
| **RF-FE-04** | **Interacción unificada de voto y comentario:** Dentro de la ficha de película, el usuario activo debe poder emitir una calificación entera en escala de 1 a 5 estrellas junto con un texto de reseña en una única acción/envío hacia `ms_backend`, recibiendo confirmación visual de recepción asincrónica sin bloquear la navegación. | Alta / Obligatorio | Usuario, `frontend`, `ms_backend` |
| **RF-FE-05** | **Sección de recomendaciones personalizadas ("Recomendadas para vos"):** En la página principal, una vez que el sistema esté operativo, el `frontend` debe mostrar una sección de películas recomendadas específicamente para el usuario seleccionado en el selector de identidad en función de su historial. | Alta / Obligatorio | Usuario, `frontend`, `ms_backend`, `ms_Recomendador` |
| **RF-FE-06** | **Sección de películas similares ("Películas similares"):** En la vista de detalle de cada película, una vez operativo el sistema, el `frontend` debe desplegar sugerencias de títulos afines por contenido temático o co-visualización. | Alta / Obligatorio | Usuario, `frontend`, `ms_backend`, `ms_Recomendador` |
| **RF-FE-07** | **Gestión visual del estado transitorio (*Cold Start*) y desbloqueo automático:** Mientras `ms_Sanitizador` y `ms_Recomendador` no hayan completado su proceso inicial: (a) el `frontend` debe permitir explorar el catálogo, ver posters, leer sinopsis y consultar reseñas previas; (b) debe mostrar indicadores visuales claros informando que la curación e indexación están en curso; (c) debe mantener deshabilitado o en espera el módulo de recomendaciones y pausada/restringida la emisión de nuevas valoraciones; (d) debe desbloquear automáticamente todas las funciones en cuanto el backend confirme la operatividad plena. | Alta / Obligatorio | `frontend`, `ms_backend` |
| **RF-FE-08** | **Ruta oculta de auditoría (`/audit`):** El `frontend` debe implementar la ruta `/audit`, accesible únicamente por URL directa (sin enlaces visibles en menús o barras de navegación públicas), donde el evaluador pueda consultar: (1) tablero con métricas macro consolidadas de anomalías detectadas y saneadas; y (2) explorador de detalles de registros modificados/depurados (tablas, acordeones o paneles con comparativa antes/después y justificaciones de inferencia emitidas por `IA_sanitizador`). | Alta / Obligatorio | Evaluador, `frontend`, `ms_backend`, `ms_auditor` |

**Criterios de aceptación clave (`RF-FE`):**
- Durante el *Cold Start*, navegar el catálogo y abrir fichas funciona sin errores; las recomendaciones y el formulario de voto muestran estado de espera claro y se habilitan automáticamente al finalizar el saneamiento + indexación sin requerir intervención manual.
- No existe ningún enlace visible hacia `/audit` en la navegación pública, pero ingresar manualmente `/audit` muestra métricas macro y detalles antes/después incluyendo casos imputados por el LLM.

---

### 2.3 `ms_backend` — Gateway / BFF y Coordinación hacia el Cliente (`RF-BE`)

| ID | Requerimiento | Prioridad | Microservicios / Actores |
| :--- | :--- | :--- | :--- |
| **RF-BE-01** | **Mediación de consultas de catálogo y reseñas:** `ms_backend` debe exponer los endpoints REST consumidos por `frontend` para listar, buscar y filtrar películas, obtener fichas de detalle y consultar el historial de reseñas, delegando las lecturas en `catalog_api`. | Alta / Obligatorio | `frontend`, `ms_backend`, `catalog_api` |
| **RF-BE-02** | **Orquestación de consultas de recomendación:** `ms_backend` debe exponer hacia el `frontend` los endpoints de *"Recomendadas para vos"* (por `user_id`) y *"Películas similares"* (por `movie_id`), consultando a `ms_Recomendador` y manteniendo el presupuesto de baja latencia (`< 100 ms`). | Alta / Obligatorio | `frontend`, `ms_backend`, `ms_Recomendador` |
| **RF-BE-03** | **Exposición del módulo de auditoría (`/audit`):** `ms_backend` debe proveer endpoints dedicados para servir al `frontend` las métricas macro consolidadas y el listado paginado/filtrable de registros detallados de auditoría consultando a `ms_auditor`. | Alta / Obligatorio | `frontend`, `ms_backend`, `ms_auditor` |
| **RF-BE-04** | **Publicación asincrónica de interacciones (voto + reseña):** Al recibir una nueva valoración unificada desde `frontend`, `ms_backend` debe validar el formato de entrada (`user_id`, `movie_id`, `rating` 1–5 entero, comentario no vacío/válido) y publicar el evento/mensaje **`reseña`** en `RabbitMQ`, respondiendo de inmediato sin realizar escrituras directas ni síncronas en `MongoDB`. | Alta / Obligatorio | `frontend`, `ms_backend`, `RabbitMQ` |
| **RF-BE-05** | **Seguimiento de estado del ciclo de vida y desbloqueo:** `ms_backend` debe escuchar los eventos de ciclo de vida en `RabbitMQ` (`evento_sanitizacion_lista` y `evento_recomendador_listo`) en *Cold Start* y consultar el estado persistido en *Warm Start*, exponiendo un endpoint de estado (polling o SSE/WebSocket) para que el `frontend` conozca en todo momento si el sistema está saneando, indexando o completamente listo. | Alta / Obligatorio | `ms_backend`, `RabbitMQ`, `frontend` |
| **RF-BE-06** | **Control de acceso en servidor durante el *Cold Start*:** Mientras el sistema no alcance el estado de operatividad plena (sanitización + recomendador listo), `ms_backend` debe rechazar con un código HTTP controlado (ej. `503 Service Unavailable`) cualquier petición de emisión de nuevas reseñas o solicitud de recomendaciones, permitiendo únicamente la lectura del catálogo, reseñas previas y estado del sistema. | Alta / Obligatorio | `ms_backend`, `frontend` |

**Criterios de aceptación clave (`RF-BE`):**
- `ms_backend` nunca abre conexiones directas hacia `MongoDB`.
- Enviar un `POST` de reseña cuando el sistema está en *Cold Start* es rechazado en el servidor; cuando el sistema está listo, encola en `RabbitMQ` y retorna respuesta de aceptación inmediata.

---

### 2.4 `catalog_api` — Servicio de Catálogo, Reseñas e Ingesta Asincrónica (`RF-CAT`)

| ID | Requerimiento | Prioridad | Microservicios / Actores |
| :--- | :--- | :--- | :--- |
| **RF-CAT-01** | **Consultas del catálogo de películas:** `catalog_api` debe consultar la colección `movies` en `MongoDB` para ofrecer listado paginado, filtrado por géneros, búsqueda por título y obtención de ficha individual completa, garantizando respuestas consistentes incluso durante el *Cold Start* mediante parseo tolerante de campos aún no saneados (ej. años o duraciones como strings). | Alta / Obligatorio | `ms_backend`, `catalog_api`, `MongoDB` |
| **RF-CAT-02** | **Consultas de reseñas e historial de usuarios:** `catalog_api` debe consultar la colección `reviews` en `MongoDB` para devolver de forma paginada las reseñas de una película determinada y las interacciones previas de un usuario (`usr_0001`–`usr_2000`), excluyendo/tolerando registros corruptos durante la etapa transitoria. | Alta / Obligatorio | `ms_backend`, `catalog_api`, `MongoDB` |
| **RF-CAT-03** | **Provisión de datos curados a `ms_Recomendador`:** `catalog_api` debe exponer un endpoint interno (o interfaz de exportación eficiente en lotes/streaming) para que `ms_Recomendador` obtenga el catálogo completo de películas y el historial limpio de reseñas al recibir `evento_sanitizacion_lista`. | Alta / Obligatorio | `ms_Recomendador`, `catalog_api`, `MongoDB` |
| **RF-CAT-04** | **Consumo asincrónico del mensaje `reseña` e ingesta en `MongoDB`:** `catalog_api` debe operar un consumidor suscrito a la cola de `reseña` en `RabbitMQ` que tome cada interacción emitida por `ms_backend`, verifique la existencia de la película (`movie_id`), normalice la fecha a `ISODate`, persista la reseña en `reviews` (gestionando idempotencia/duplicados del mismo usuario sobre la misma película según la regla de negocio definida) y actualice los contadores/promedios necesarios en `movies`. | Alta / Obligatorio | `RabbitMQ`, `catalog_api`, `MongoDB` |
| **RF-CAT-05** | **Confiabilidad sin pérdida de mensajes:** El consumidor de `reseña` en `catalog_api` debe confirmar (`ACK`) únicamente tras persistir exitosamente en `MongoDB`, manejando errores transitorios mediante reintentos o cola de mensajes muertos (DLQ) para evitar pérdida silenciosa de votos ante ráfagas o caídas temporales. | Alta / Obligatorio | `RabbitMQ`, `catalog_api`, `MongoDB` |

**Criterios de aceptación clave (`RF-CAT`):**
- Al enviar múltiples reseñas en ráfaga desde el `frontend`, todas se encolan en `RabbitMQ` y terminan persistidas en `MongoDB` sin bloqueos ni duplicación errónea.
- Durante el *Cold Start*, listar películas con datos sucios (ej. año `"2015è"` o género `"Action, Drama"`) no produce errores `500` en `catalog_api`.

---

### 2.5 `ms_Sanitizador` e `IA_sanitizador` — Calidad de Datos y Resolución Semántica (`RF-SAN`)

| ID | Requerimiento | Prioridad | Microservicios / Actores |
| :--- | :--- | :--- | :--- |
| **RF-SAN-01** | **Detección de estado crudo vs. saneado (*Cold* vs. *Warm Start*):** Al iniciar, `ms_Sanitizador` debe verificar en `MongoDB` si el dataset ya completó el proceso de saneamiento. Si ya fue saneado, debe omitir toda limpieza y llamadas a `IA_sanitizador`, asegurando un arranque rápido e idempotente; si detecta datos crudos, debe ejecutar el flujo completo de curación. | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB` |
| **RF-SAN-02** | **Deduplicación de películas redundantes:** `ms_Sanitizador` debe detectar películas duplicadas en la colección `movies`, consolidar el registro canónico preservando la información más completa y reasignar o compatibilizar las referencias de `reviews` (`movie_id`) para no romper la integridad referencial antes de eliminar los duplicados. | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB` |
| **RF-SAN-03** | **Normalización de tipos corruptos (años y duraciones):** Debe convertir años almacenados como cadenas corruptas (ej. `"1994"` o `"2015è"`) a valores numéricos enteros válidos, y normalizar duraciones en formato texto (ej. `"120 min"`) o inválidas/negativas a enteros positivos consistentes (o `null` sanitizado cuando no sea recuperable). | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB` |
| **RF-SAN-04** | **Homogeneización de géneros:** Debe transformar géneros almacenados como strings delimitados (ej. `"Action, Drama"`) u otros formatos heterogéneos en arreglos de cadenas normalizados (`list[str]`), limpiando espacios y duplicados por película. | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB` |
| **RF-SAN-05** | **Normalización de ratings IMDb y limpieza de títulos:** Debe normalizar los puntajes de IMDb a formato numérico homogéneo dentro del rango válido y eliminar espacios en blanco espurios (al inicio, fin o múltiples intermedios) en los títulos de `movies`. | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB` |
| **RF-SAN-06** | **Purga o clasificación de comentarios spam/basura:** Debe detectar reseñas cuyo texto corresponda a ruido o basura (ej. `"..."`, `""`, `"none"`, `"test"`, caracteres sin contenido semántico) y depurarlas/purgarlas del conjunto activo de reseñas válidas, registrando cada acción para auditoría. | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB` |
| **RF-SAN-07** | **Estandarización de fechas a `ISODate`:** Debe parsear fechas almacenadas en `reviews` (o `movies`) bajo formatos de string dispares y convertirlas a tipo fecha estándar `ISODate` de MongoDB. | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB` |
| **RF-SAN-08** | **Eliminación de reseñas huérfanas:** Debe identificar y eliminar de `reviews` todos los documentos cuyo `movie_id` no corresponda a ninguna película existente en `movies` (tras resolver deduplicaciones), garantizando 100 % de integridad referencial. | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB` |
| **RF-SAN-09** | **Resolución de colisiones de votos duplicados:** Debe detectar múltiples reseñas emitidas por un mismo `user_id` sobre una misma `movie_id` y resolver la colisión bajo un criterio determinístico y auditable (ej. conservar la más reciente o la más informativa/completa, descartando el duplicado). | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB` |
| **RF-SAN-10** | **Resolución semántica obligatoria con `IA_sanitizador` (imputación de `rating: null`):** Para reseñas válidas que posean comentario en inglés pero `rating: null` (o faltante), `ms_Sanitizador` debe invocar a `IA_sanitizador` (modelo LLM local como Ollama/SLM o API externa) para ejecutar análisis de sentimiento/inferencia semántica y deducir: (1) una calificación entera obligatoria entre `1` y `5` estrellas; y (2) una breve justificación textual del criterio inferido. Debe incluir validación estricta del rango `1..5` y manejo de reintentos/errores frente a respuestas inválidas del LLM. | Alta / Obligatorio | `ms_Sanitizador`, `IA_sanitizador`, `MongoDB` |
| **RF-SAN-11** | **Emisión de trazabilidad hacia auditoría (`evento-reg-audit`):** Por cada anomalía detectada, corregida, descartada o imputada por `IA_sanitizador`, `ms_Sanitizador` debe publicar un mensaje `evento-reg-audit` en `RabbitMQ` (individual o agrupado en lotes controlados) incluyendo: categoría de anomalía, colección/ID afectado, estado anterior (`before`), estado corregido (`after`), timestamp y justificación del LLM cuando corresponda. | Alta / Obligatorio | `ms_Sanitizador`, `RabbitMQ` |
| **RF-SAN-12** | **Persistencia de estado saneado y emisión de `evento_sanitizacion_lista`:** Al concluir todas las fases de curación y garantizar la consistencia de `movies` y `reviews`, `ms_Sanitizador` debe registrar la marca persistente de dataset saneado en `MongoDB` y publicar el evento `evento_sanitizacion_lista` en `RabbitMQ` para habilitar el arranque del recomendador. | Alta / Obligatorio | `ms_Sanitizador`, `MongoDB`, `RabbitMQ` |

**Criterios de aceptación clave (`RF-SAN`):**
- Tras finalizar el agente, cero documentos de `reviews` quedan con `rating: null` o `movie_id` inexistente, y no existen pares `(user_id, movie_id)` duplicados.
- Todos los campos `year`, `runtime`, `genres` y fechas cumplen su tipo objetivo en `MongoDB`.
- Los registros imputados por `IA_sanitizador` contienen calificación entera `1..5` y justificación trazable en auditoría.

---

### 2.6 `ms_auditor` — Servicio de Registro y Consulta de Auditoría (`RF-AUD`)

| ID | Requerimiento | Prioridad | Microservicios / Actores |
| :--- | :--- | :--- | :--- |
| **RF-AUD-01** | **Consumo asincrónico de `evento-reg-audit`:** `ms_auditor` debe suscribirse a `evento-reg-audit` en `RabbitMQ` para recibir de forma desacoplada y sin pérdida de mensajes todos los hallazgos y correcciones emitidos por `ms_Sanitizador`. | Alta / Obligatorio | `RabbitMQ`, `ms_auditor` |
| **RF-AUD-02** | **Persistencia en `registros-auditoria` (`MongoDB`):** `ms_auditor` debe almacenar en `MongoDB` (colección `registros-auditoria`) cada evento de curación recibido con esquema estructurado: tipo de anomalía, colección (`movies` / `reviews`), identificador de documento, valor anterior, valor posterior, fuente (`regla_deterministica` vs. `IA_sanitizador`), justificación semántica (si aplica) y fecha de procesamiento. | Alta / Obligatorio | `ms_auditor`, `MongoDB` |
| **RF-AUD-03** | **Consolidación y consulta de métricas macro:** `ms_auditor` debe calcular y exponer un resumen consolidado con los contadores totales por categoría de anomalía saneada (películas deduplicadas, años normalizados, duraciones corregidas, géneros homogeneizados, ratings IMDb ajustados, títulos limpiados, comentarios spam descartados, fechas corregidas, reseñas huérfanas eliminadas, votos duplicados resueltos y notas imputadas por el LLM). | Alta / Obligatorio | `ms_auditor`, `ms_backend`, `MongoDB` |
| **RF-AUD-04** | **Consulta detallada e inspeccionable para `/audit`:** `ms_auditor` debe exponer endpoints de consulta paginada y filtrable por tipo de anomalía o entidad que permitan al evaluador inspeccionar en `/audit` casos individuales con comparativa antes/después y las justificaciones generadas por `IA_sanitizador`. | Alta / Obligatorio | `ms_auditor`, `ms_backend`, `frontend` |

**Criterios de aceptación clave (`RF-AUD`):**
- Los contadores del tablero macro en `/audit` coinciden exactamente con los registros almacenados en `registros-auditoria`.
- En *Warm Start*, las métricas y detalles de auditoría siguen disponibles de inmediato desde `MongoDB`.

---

### 2.7 `ms_Recomendador` — Motor de Recomendaciones de Baja Latencia (`RF-REC`)

| ID | Requerimiento | Prioridad | Microservicios / Actores |
| :--- | :--- | :--- | :--- |
| **RF-REC-01** | **Activación por `evento_sanitizacion_lista` y carga desde `catalog_api`:** En *Cold Start*, `ms_Recomendador` debe esperar la recepción de `evento_sanitizacion_lista` desde `RabbitMQ`; al recibirlo, debe obtener el catálogo limpio y las reseñas saneadas desde `catalog_api` para ejecutar su fase inicial de entrenamiento/indexación. | Alta / Obligatorio | `RabbitMQ`, `ms_Recomendador`, `catalog_api` |
| **RF-REC-02** | **Precomputación de modelos y emisión de `evento_recomendador_listo`:** Durante su fase de inicialización, `ms_Recomendador` debe precomputar las matrices de afinidad, embeddings/vectores de similitud o modelos en memoria RAM (y persistir los artefactos/estado para futuros reinicios) y publicar `evento_recomendador_listo` en `RabbitMQ` una vez apto para servir consultas. | Alta / Obligatorio | `ms_Recomendador`, `RabbitMQ` |
| **RF-REC-03** | **Inicialización acelerada en *Warm Start*:** Ante reinicios posteriores con artefactos persistidos y estado ya saneado, `ms_Recomendador` debe recargar sus estructuras precomputadas en memoria en pocos segundos sin volver a ejecutar la barrida completa inicial, notificando disponibilidad inmediata. | Alta / Obligatorio | `ms_Recomendador`, `ms_backend` |
| **RF-REC-04** | **Generación de recomendaciones personalizadas y por similitud:** Debe exponer consultas diferenciadas para: (a) **Recomendadas para un usuario** (`usr_0001`–`usr_2000`) en base a su historial de calificaciones/preferencias, contemplando estrategia de respaldo (*fallback* por popularidad ponderada o contenido) si el usuario tiene pocas o nulas interacciones; y (b) **Películas similares** dada una película de referencia, basadas en afinidad temática o patrones de co-visualización. | Alta / Obligatorio | `ms_backend`, `ms_Recomendador` |
| **RF-REC-05** | **Garantía de baja latencia (`< 100 ms`):** Todas las peticiones HTTP de recomendación deben resolverse directamente sobre índices, matrices o listas precomputadas en memoria con tiempo de respuesta objetivo **`< 100 ms`**. Queda estrictamente prohibido reentrenar modelos o calcular similitudes completas contra todo el catálogo durante el ciclo de una petición HTTP. | Alta / Obligatorio | `ms_Recomendador`, `ms_backend` |

**Criterios de aceptación clave (`RF-REC`):**
- Los endpoints de recomendación personalizada y películas similares responden en `< 100 ms` tras la inicialización.
- Cuando un usuario emite una nueva reseña (ingesta asincrónica vía `RabbitMQ` -> `catalog_api`), el recomendador puede actualizar o incorporar progresivamente la señal en segundo plano (sin bloquear peticiones HTTP), o mantener coherencia de recomendaciones sin degradar latencia.

---

### 2.8 Funcionalidades Deseables con LLM en la Experiencia de Usuario (`RF-LLM`)

*(Categoría: Valor Agregado / Ideal según Sección 8 y Rúbrica de Evaluación del Enunciado)*

| ID | Requerimiento | Prioridad | Microservicios / Actores |
| :--- | :--- | :--- | :--- |
| **RF-LLM-01** | **Traducción dinámica multilingüe de contenidos:** El sistema debe integrar un LLM que permita traducir dinámicamente al español (u otro idioma seleccionable desde la interfaz) los textos recuperados en inglés de la base de datos —específicamente descripciones, sinopsis (`plot` / `fullplot`) y reseñas de la comunidad—, preservando preferentemente los títulos originales de las películas y nombres propios. | Deseable / Valor Agregado | Usuario, `frontend`, `ms_backend`, componente LLM (`IA_sanitizador` o adaptador LLM) |
| **RF-LLM-02** | **Asistente cinéfilo conversacional (chat interactivo):** El `frontend` debe incorporar una interfaz de chat donde el usuario pueda realizar consultas en lenguaje natural (ej. *"Recomiéndame una película de ciencia ficción de los años 90 con un giro inesperado"*, *"¿Qué tienen en común Inception e Interstellar?"*) y recibir respuestas fundamentadas con conocimiento del catálogo real de DCIC-Flix26 y justificación de las sugerencias. | Deseable / Valor Agregado | Usuario, `frontend`, `ms_backend`, `catalog_api` / `ms_Recomendador`, componente LLM |

---

## 3. Matriz de Trazabilidad (Requerimientos ↔ Microservicios ↔ Eventos ↔ Rúbrica)

| Grupo de RF | Microservicios / Componentes Involucrados | Eventos / Colas `RabbitMQ` | Criterio de Rúbrica Asociado (Peso) |
| :--- | :--- | :--- | :--- |
| **`RF-CIC-01` .. `RF-CIC-06`** | `docker-compose`, `MongoDB`, `RabbitMQ`, `frontend`, `ms_backend`, `ms_Sanitizador`, `ms_Recomendador` | `evento_sanitizacion_lista`, `evento_recomendador_listo` | **Orquestación y redes en Docker (20%)** |
| **`RF-SAN-01` .. `RF-SAN-12`** | `ms_Sanitizador`, `IA_sanitizador`, `MongoDB` | `evento-reg-audit`, `evento_sanitizacion_lista` | **Agente de limpieza e integración LLM (25%)** |
| **`RF-AUD-01` .. `RF-AUD-04`** | `ms_auditor`, `MongoDB`, `ms_backend`, `frontend` (`/audit`) | `evento-reg-audit` | **Agente de limpieza e integración LLM (25%)** *(auditoría `/audit`)* |
| **`RF-REC-01` .. `RF-REC-05`** | `ms_Recomendador`, `catalog_api`, `ms_backend` | `evento_sanitizacion_lista`, `evento_recomendador_listo` | **Motor de recomendaciones (25%)** |
| **`RF-FE-01` .. `RF-FE-08`** | `frontend`, `ms_backend`, `catalog_api`, `ms_Recomendador`, `ms_auditor` | `evento_sanitizacion_lista`, `evento_recomendador_listo` | **Frontend, UX y funciones LLM (20%)** |
| **`RF-LLM-01` .. `RF-LLM-02`** | `frontend`, `ms_backend`, módulo LLM, `catalog_api` | — | **Frontend, UX y funciones LLM (20%)** *(nivel excelente 9–10)* |
| **`RF-BE-04`, `RF-CAT-04` .. `RF-CAT-05`** | `ms_backend`, `RabbitMQ`, `catalog_api`, `MongoDB` | `reseña` | **Mensajería asincrónica RabbitMQ (5%)** |

---

## 4. Observaciones Técnicas sobre [Diagrama TADW.drawio](file:///c:/Users/ferna/Desktop/Topicos/proyecto/Diagrama%20TADW.drawio) para Cumplir Todos los RF

Al contrastar el diagrama actual de microservicios contra los requerimientos funcionales anteriores, existen **conexiones y detalles estructurales en [Diagrama TADW.drawio](file:///c:/Users/ferna/Desktop/Topicos/proyecto/Diagrama%20TADW.drawio) que deberán completarse al implementar o actualizar el diagrama**:

1. **Conexión `frontend <-> ms_backend`:** Actualmente el nodo `frontend` no tiene flecha hacia `ms_backend`; requiere comunicación HTTP/REST (y SSE/WebSocket o polling para el estado del *Cold Start*).
2. **Conexión `ms_backend -> ms_Recomendador`:** Es indispensable para cumplir `RF-FE-05`, `RF-FE-06` y `RF-BE-02` (*"Recomendadas para vos"* y *"Películas similares"*).
3. **Conexión `ms_backend -> ms_auditor`:** Es indispensable para cumplir `RF-FE-08`, `RF-BE-03` y `RF-AUD-03/04` (alimentar `/audit` con métricas macro y detalle antes/después).
4. **Consumo de `evento_recomendador_listo` por `ms_backend`:** En el diagrama `ms_backend` solo lee `evento_sanitizacion_lista`, pero `RF-CIC-03` y `RF-FE-07` exigen mantener bloqueadas las recomendaciones y reseñas hasta que el recomendador también haya completado su indexación inicial (`evento_recomendador_listo`).
5. **Integración LLM para funcionalidades deseables (`RF-LLM-01` y `RF-LLM-02`):** En el diagrama actual el nodo `IA_sanitizador` solo se conecta con `ms_Sanitizador`. Si se implementan la traducción dinámica al español y/o el chat cinéfilo para aspirar al puntaje máximo en Frontend/UX, `ms_backend` (o un conector LLM dedicado) deberá comunicarse con el proveedor LLM y consultar el contexto en `catalog_api`/`ms_Recomendador`.
6. **Subredes Docker y nombres de eventos:** Es recomendable delimitar en el diagrama las subredes Docker (`frontend` aislado de `MongoDB`, `RF-CIC-05`) y corregir las etiquetas `RabitMQ` -> `RabbitMQ` y `evento_rodomendador_listo` -> `evento_recomendador_listo`.
