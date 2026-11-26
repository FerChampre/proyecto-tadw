# Proyecto final: arquitectura de microservicios, agentes y recomendaciones (DCIC-Flix26)
### Cátedra de Tópicos Avanzados de Desarrollo Web — edición 2026
**Departamento de Ciencias e Ingeniería de la Computación (DCIC)**  
**Universidad Nacional del Sur (UNS)**  
**Fecha tope de entrega:** 08/12/2026 23:59 hs

---

## 1. Contexto y visión general

En el desarrollo web moderno, las aplicaciones a gran escala han dejado de ser monolitos acoplados a una única base de datos relacional. Hoy en día, los sistemas de plataformas de contenidos y streaming (como Netflix, Spotify o Amazon) combinan:
* **Microservicios especializados**, desacoplados e independientes.
* **Bases de datos no relacionales (MongoDB)** para almacenar catálogos documentales flexibles e interacciones de usuarios.
* **Arquitecturas dirigidas por eventos (message brokers con RabbitMQ)** para absorber ráfagas masivas de actividad de los usuarios sin degradar la experiencia en el frontend.
* **Agentes de calidad de datos con integración de LLMs** para monitorear, auditar y sanear inconsistencias estructurales y semánticas en la información cruda.
* **Sistemas de recomendación de alto rendimiento**, capaces de predecir afinidades y ofrecer sugerencias contextuales con latencia mínima.

El objetivo principal de este proyecto es el **diseño y la ingeniería global del sistema**: más allá de la implementación aislada de una función o endpoint, el desafío formativo radica en concebir la arquitectura integral, articular armónicamente múltiples servicios heterogéneos, garantizar la resiliencia en el ciclo de vida de los contenedores, proteger las fronteras de red y asegurar contratos de datos e interfaces consistentes entre subsistemas.

Para materializar esta visión, el desafío consiste en diseñar e implementar **DCIC-Flix26**, una plataforma web de catálogo cinematográfico orquestada integralmente con **Docker Compose**, donde conviven el frontend, los microservicios de ingesta, el agente de curación de datos y el motor de recomendaciones.

---

## 2. Modalidad de trabajo, autoría y criterio de evaluación

* **Fecha tope de entrega:** 08/12/2026 a las 23:59 hs.
* **Modalidad:** grupos de **2 o 3 estudiantes**.
* **El rol de la IA y la responsabilidad del equipo:**  
  En esta materia se asume y alienta que gran parte del código sea generado o asistido mediante agentes de inteligencia artificial y herramientas modernas de codificación asistida. Sin embargo, **el equipo humano es el último y exclusivo responsable del resultado final.** El código entregado debe ser coherente, estructurado y plenamente comprendido por sus autores. Se evaluará que el equipo haya auditado, integrado y depurado cada componente, evitando inconsistencias estilísticas entre microservicios, código muerto o dependencias superfluas que evidencien falta de criterio de ingeniería.
* **Criterio de evaluación de la cátedra:**  
  Los docentes clonarán el repositorio del grupo y ejecutarán un único comando:
  ```bash
  docker compose up --build
  ```
  **No debe ser necesario ningún paso manual adicional** (ni migraciones manuales, ni configuración manual de colas en RabbitMQ, ni endpoints que deban ser tocados a mano para inicializar el sistema). Todo el flujo de inicialización debe estar coordinado entre los contenedores.

---

## 3. Arquitectura del sistema y seguridad de redes

El diseño de la arquitectura de microservicios y la topología de redes forman parte de la ingeniería que debe proponer el grupo, y será uno de los ejes centrales de la evaluación. La cátedra no impone un diagrama único, sino que evaluará el cumplimiento de los siguientes objetivos de diseño:

* **Infraestructura base dockerizada (imágenes oficiales de Docker Hub):** tanto MongoDB como RabbitMQ deben estar obligatoriamente contenerizados dentro del `docker-compose.yml` utilizando sus imágenes oficiales de Docker Hub. No se admiten bases de datos o message brokers externos ni instancias instaladas fuera del compose.
* **Desacoplamiento funcional:** estructurar la solución en microservicios independientes, contenerizados y orquestados mediante `docker-compose.yml`, donde cada componente posea una responsabilidad cohesiva (ingesta, curación de datos, recomendación, interfaz de usuario).
* **Aislamiento de redes y principio de mínimo privilegio:** estructurar el entorno en subredes de Docker adecuadamente aisladas. **Como requisito mandatorio de seguridad, el contenedor del frontend no debe tener conectividad de red con MongoDB ni poder resolver su nombre de host.** Cualquier intento de acceso directo desde el frontend a la base de datos debe ser rechazado a nivel de red; todo acceso o consulta debe estar mediado por la arquitectura que proponga el equipo.
* **Ingesta asincrónica y resiliencia:** las interacciones de los usuarios (votos y comentarios) no deben impactar de forma directa ni síncrona en la base de datos desde el cliente; deben canalizarse a través de RabbitMQ para garantizar absorción de ráfagas, desacoplamiento y persistencia confiable.
* **Coordinación y comunicación interservicio:** definir los contratos, eventos y mecanismos de comunicación interna necesarios para orquestar la notificación de datos limpios y la actualización del catálogo sin acoplamientos indebidos.

---

## 4. El ciclo de vida: arranque en frío vs. segundo arranque

El sistema debe gestionar con elegancia la persistencia de datos mediante volúmenes de Docker y coordinar el estado operativo entre el backend y la interfaz de usuario:

### A. Arranque en frío (cold start - primer `docker compose up`)
1. **Inicialización de MongoDB:** levanta cargando el dataset inicial provisto por la cátedra en la carpeta `dataset/` (mediante scripts en `/docker-entrypoint-initdb.d/`).
2. **Disparo del agente de limpieza:** el agente detecta que la base contiene datos crudos/sucios y comienza su diagnóstico y saneamiento.
3. **Limitación de opciones en el frontend durante el saneamiento:** mientras los datos no estén saneados y el recomendador no esté listo:
   - La interfaz web debe permitir en todo momento explorar el catálogo, ver las carátulas, leer las descripciones/sinopsis y consultar las reseñas previas de la comunidad presentes en la base.
   - Sin embargo, debe informar con claridad este estado transitorio (por ejemplo, mediante indicadores visuales o mensajes de estado sobre la curación e indexación en progreso).
   - Se deben restringir las funcionalidades dependientes del saneamiento completo: el módulo de recomendaciones debe permanecer deshabilitado o en espera, y la emisión de nuevas valoraciones/reseñas debe pausarse o condicionarse hasta que el catálogo adquiera consistencia.
4. **Notificación y barrida del motor de recomendaciones:** una vez concluida la limpieza, el agente emite la señal de datos listos (ej. mediante un evento en RabbitMQ); el motor de recomendaciones toma la señal, realiza su indexación o entrenamiento inicial y confirma su operatividad.
5. **Habilitación plena:** el frontend desbloquea automáticamente todas sus capacidades (recomendaciones personalizadas y similares, envío de votos y reseñas).

### B. Segundo arranque (warm start - reinicios posteriores)
Si la base de datos ya fue saneada y los volúmenes de Docker están persistidos, al ejecutar nuevamente `docker compose up`:
* El sistema **no debe volver a ejecutar todo el proceso pesado de limpieza ni la barrida inicial**.
* Los microservicios deben detectar el estado saneado previo y arrancar de inmediato (en pocos segundos), con el frontend plenamente habilitado desde el primer instante.

---

## 5. El desafío de calidad de datos y el agente de limpieza

La cátedra provee un dataset realista derivado de *sample_mflix*:
* **`movies`:** ~20.000 películas con poster y sinopsis.
* **`reviews`:** ~61.000 comentarios reales en inglés correlacionados con ratings de 1 a 5 estrellas, generados sobre un universo de 2.000 usuarios sintéticos (`usr_0001` a `usr_2000`).

Sin embargo, el dataset contiene **anomalías deliberadas de calidad de datos** inyectadas por la cátedra que el agente de limpieza deberá diagnosticar y sanear.

### Responsabilidades del agente de limpieza:
1. **Reglas determinísticas y validación de integridad:**
   - Detectar y deduplicar películas redundantes.
   - Normalizar tipos corruptos (años como strings `"1994"` o `"2015è"`, duraciones como `"120 min"` o negativas).
   - Homogeneizar géneros (convertir strings delimitados `"Action, Drama"` en `list[str]`).
   - Normalizar ratings de IMDb y limpiar espacios espurios en títulos.
   - Purgar o clasificar comentarios spam/basura (`"..."`, `""`, `"none"`, `"test"`).
   - Corregir fechas guardadas como strings dispares a tipo fecha ISO (`ISODate`).
   - Eliminar reseñas huérfanas cuyos `movie_id` no existan en la colección de películas.
   - Resolver colisiones de votos duplicados de un mismo usuario sobre una misma película.
2. **Integración obligatoria de LLM para ambigüedades semánticas:**  
   El agente debe incorporar un modelo de lenguaje (ya sea un modelo local pequeño como Ollama/SLM o una llamada a API externa) para **al menos una tarea de resolución semántica que las reglas rígidas no puedan resolver**.  
   *Ejemplo clave:* el dataset contiene cientos de reseñas con `rating: null` pero que incluyen un comentario detallado en inglés (elogioso o crítico). El agente debe utilizar el LLM para realizar análisis de sentimiento / inferencia y deducir la calificación entera (1 a 5 estrellas) correspondiente al texto.
3. **Auditoría visualizable:** el agente debe registrar con precisión todas las anomalías diagnosticadas y corregidas. La cátedra debe poder auditar este proceso visualmente: el frontend debe implementar de forma obligatoria la **ruta oculta `/audit`** (no enlazada en la navegación pública) que permita inspeccionar:
   - **Métricas macro:** tablero con números consolidados de anomalías detectadas y saneadas (total de películas deduplicadas, años normalizados, géneros corregidos, ratings de IMDb ajustados, comentarios basura descartados, reseñas huérfanas eliminadas, notas imputadas por el LLM, etc.).
   - **Inspección de detalles:** capacidad de explorar registros individuales modificados o depurados (por ejemplo, tablas, acordeones o paneles desplegables que muestren comparativas de antes/después o las justificaciones de inferencia emitidas por el LLM).

---

## 6. Motor de recomendaciones (libertad total y máxima creatividad)

> [!IMPORTANT]
> **El punto neurálgico de creatividad del proyecto:**  
> La cátedra otorga **libertad técnica absoluta** para la concepción del motor de recomendaciones. Pueden utilizar cualquiera de los enfoques vistos en clase o técnicas avanzadas de la industria:
> * **Filtrado colaborativo:** descomposición matricial o vecinos cercanos con `scikit-surprise` (SVD, SVD++, NMF, KNNWithMeans) aprovechando la densidad del catálogo de reviews.
> * **Filtrado basado en contenido:** normalización de metadatos (metadata soup con director, reparto, géneros y sinopsis) y similitud coseno mediante bolsa de palabras (`CountVectorizer`), TF-IDF o embeddings vectoriales.
> * **Modelos híbridos y baselines:** fórmulas de rating ponderado (ej. baseline de IMDb), esquemas de dos etapas (*retrieval + re-ranking*).
> * **Modelos basados en grafos o redes neuronales.**

### Restricciones y compromisos obligatorios:
1. **Explicación de la estrategia:** el grupo debe documentar con claridad en el `README.md` la estrategia algorítmica adoptada, por qué decidieron implementarla y cualquier otra consideración relevante.
2. **Restricción crítica de baja latencia:** el frontend no puede esperar segundos para mostrar una recomendación. Las consultas a los endpoints de recomendación deben responder de inmediato (**tiempo objetivo: < 100 ms**).  
   *Consecuencia de diseño:* el motor debe precomputar matrices, entrenar modelos o mantener índices en memoria durante el arranque o en procesos de fondo; está **terminantemente prohibido** reentrenar modelos o calcular similitudes completas contra todo el catálogo durante una petición HTTP del usuario.

---

## 7. Frontend web e ingesta asincrónica

La interfaz de usuario debe ser atractiva, funcional y responsiva (pueden usar React, Vue, Svelte, Next.js, HTML/Tailwind o la tecnología de su preferencia).

### Funcionalidades del frontend:
1. **Exploración del catálogo:** vista de películas con posters visuales, filtros por género y búsqueda por título.
2. **Ficha de detalle y lectura de críticas previas:** visualización completa de la información técnica, sinopsis (`plot` / `fullplot`), reparto, director y listado navegable de las críticas y calificaciones históricas de la comunidad presentes en la base.
3. **Selector de identidad de usuario:** un selector simple (dropdown o buscador) que permita elegir con qué usuario del catálogo (`usr_0001` a `usr_2000`) se está interactuando. Esto permite:
   - Ver las **recomendaciones personalizadas** generadas para ese perfil en base a su historial previo.
   - Emitir nuevos votos en su nombre.
4. **Interacción unificada (voto + comentario):**  
   El usuario califica (escala de 1 a 5 estrellas) y escribe su reseña en una misma acción.
   * **Flujo asincrónico:** el frontend envía esta interacción al gateway/backend, el cual **publica un mensaje en RabbitMQ**. Un microservicio consumidor desacoplado toma el mensaje de la cola y lo persiste de forma segura en MongoDB.
5. **Sección de recomendaciones:**
   - En la página principal: "Recomendadas para vos" (personalizadas según el usuario activo).
   - En la ficha de película: "Películas similares" (basadas en afinidad temática o co-visualización).
6. **Manejo del estado transitorio (arranque en frío):**  
   Durante el arranque en frío, el frontend debe permitir en todo momento explorar el catálogo, ver carátulas y leer las descripciones y reseñas previas ya existentes en la base, pero debe advertir visualmente que la curación inicial está en curso y restringir temporalmente las secciones de recomendación y el ingreso de nuevas valoraciones hasta recibir la confirmación de datos listos.
7. **Ruta oculta de auditoría (`/audit`):**  
   Una vista especializada accesible por URL directa en `/audit` (sin enlace en la barra de navegación pública) que permita a los evaluadores inspeccionar el resumen macro de la curación y visualizar el detalle de los registros depurados o inferidos por el agente.

---

## 8. Funcionalidades deseables con LLM (valor agregado / ideal)

Como objetivos de excelencia técnica y creatividad con modelos de lenguaje, se valorará positivamente la incorporación de las siguientes capacidades en la experiencia de usuario:
* **Traducción dinámica multilingüe de contenidos:**  
  Dado que el catálogo semilla cuenta con sinopsis y opiniones de usuarios en inglés, es altamente deseable que el sistema integre un LLM para traducir al español (o a un idioma seleccionado por el usuario desde una suite de idiomas) todo texto recuperado de la base de datos (descripciones, tramas y reseñas de la comunidad), preservando preferentemente los nombres propios o títulos originales de las películas.
* **Asistente cinéfilo conversacional (chat interactivo):**  
  Un chat integrado en la interfaz web donde el usuario pueda dialogar interactivamente con una IA cinéfila (*"Recomiéndame una película de ciencia ficción de los años 90 con un giro inesperado"*, *"¿Qué tienen en común Inception e Interstellar?"*), respondiendo con conocimiento del catálogo y justificando las sugerencias.

---

## 9. Entregables

El grupo deberá entregar un único **repositorio Git** (GitHub / GitLab) que contenga:
1. **`docker-compose.yml`:** orquestando todos los microservicios, volúmenes y subredes descriptas.
2. **Código fuente completo:** estructurado en carpetas por componente según la arquitectura propuesta por el equipo.
3. **Carpeta `dataset/`:** con los archivos semilla y scripts de inicialización de MongoDB provistos por la cátedra.
4. **`README.md` técnico y conciso:**
   - Instrucciones de ejecución (`docker compose up --build`).
   - Diagrama y justificación de la arquitectura de redes implementada.
   - Explicación de la estrategia de limpieza y del uso del LLM en el agente.
   - Confirmación de funcionamiento de la ruta oculta `/audit` en el frontend para acceder a la visualización de la auditoría de limpieza (métricas macro y detalle).
   - Explicación profunda de la estrategia del motor de recomendaciones y justificación del modelo adoptado.
   - Detalle de las integraciones con LLM implementadas en la interfaz (traducción multilingüe o chat conversacional cinéfilo).

---

## 10. Rúbrica de evaluación

| Criterio | Ponderación | Nivel excelente (9 - 10) | Nivel aprobado (6 - 8) | Nivel insuficiente (1 - 5) |
| :--- | :---: | :--- | :--- | :--- |
| **Orquestación y redes en Docker** | **20%** | `docker compose up` levanta todo de punta a punta. Subredes aisladas correctamente (el frontend no llega a Mongo). Manejo adecuado de cold vs. warm start con limitación de opciones en el frontend durante la curación inicial. | Levanta con compose pero con dependencias frágiles, aislamiento incompleto o falta de feedback en el frontend durante el arranque. | Falla al iniciar, requiere comandos manuales o el frontend accede directamente a la base de datos. |
| **Agente de limpieza e integración LLM** | **25%** | Sanea exitosamente las anomalías del catálogo. Integra un LLM/SLM para resolver ambigüedades (ej. imputación de rating por review). Genera auditoría clara con métricas macro y detalle accesible desde la ruta `/audit` del frontend. | Limpia la mayoría de las anomalías pero falla en casos borde o la auditoría es incompleta o inaccesible desde `/audit`. | Limpieza deficiente, no usa LLM o rompe la integridad referencial de los datos. |
| **Motor de recomendaciones (creatividad y rendimiento)** | **25%** | Enfoque algorítmico creativo y bien fundamentado. Latencia de respuesta mínima (< 100 ms). Manejo adecuado del cold start. | Implementa un recomendador funcional pero básico o con latencias perceptibles para el usuario. | No produce recomendaciones coherentes o reentrena el modelo bloqueando cada petición HTTP. |
| **Frontend, UX y funciones LLM (ideal)** | **20%** | Frontend fluido y completo (navegación del catálogo, lectura de descripciones y reviews previas, selector de usuario, votos). Incluye capacidades con LLM como traducción a español o chat interactivo cinéfilo. | Frontend funcional con catálogo y votos, pero sin integración de LLM en la UI o con interfaz rudimentaria. | Interfaz rota, navegación incompleta o errores al consultar el catálogo o enviar interacciones. |
| **Mensajería asincrónica (RabbitMQ)** | **5%** | Desacoplamiento total: el frontend envía interacciones al broker y el consumidor las guarda en Mongo sin pérdida de mensajes. | RabbitMQ funciona pero con acoplamiento innecesario o falta de manejo de errores en el consumidor. | No utiliza el message broker para las interacciones o persiste de forma síncrona. |
| **Calidad y consistencia del código** | **5%** | Código limpio, modular y coherente en todos los servicios. Se evidencia revisión, comprensión y auditoría del código generado por agentes, sin artefactos innecesarios ni código muerto. | Código funcional pero con estilos dispares entre microservicios, dependencias redundantes o comentarios genéricos de IA sin revisar. | Código desorganizado, inconsistente o con fragmentos de agentes pegados sin adaptar que evidencian falta de entendimiento. |
