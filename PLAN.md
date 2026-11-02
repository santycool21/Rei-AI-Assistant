# Mika — Plan de desarrollo

> Asistente virtual de IA personal: conversa en español rioplatense, recuerda, te acompaña mientras usás la PC y participa en tus streams.
>
> **Estado:** Etapa 0 pendiente de aprobación. Este documento se actualiza al cerrar cada etapa.

---

## 1. Análisis del repositorio

El repositorio (`Rei-AI-Assistant`) está vacío: solo tiene un `README.md` con una línea de descripción. No hay código, dependencias ni decisiones previas que respetar, así que la arquitectura se diseña desde cero.

- El repo se llama **Rei**, pero la asistente se llama **Mika**. Propongo mantener el nombre del repo (renombrarlo en GitHub es trivial y no rompe nada) y usar `mika` como nombre interno de paquetes y carpetas. El nombre visible de la asistente sale de la configuración, no del código, así que cambiarlo después cuesta lo mismo que editar un archivo.

---

## 2. Principios de diseño

1. **La nube piensa, la PC percibe y muestra.** Todo lo pesado (modelos de lenguaje, reconocimiento de voz, síntesis de voz, memoria, integraciones con Twitch/YouTube) corre en un servidor. El cliente de Windows es liviano: captura micrófono, pantalla y webcam, reproduce audio y dibuja el avatar.
2. **Nada está atado a un proveedor.** LLM, STT, TTS, embeddings y visión se acceden a través de interfaces propias. Cambiar de Claude a Gemini, de ElevenLabs a Azure, etc., es un cambio de configuración, no de código.
3. **Modular por diseño.** Cada capacidad (voz, visión, chat de Twitch, juegos…) es un módulo que se conecta a un núcleo común mediante eventos. Agregar una capacidad no exige tocar las demás.
4. **Vos tenés el control.** Pantalla, webcam, micrófono y control de teclado/mouse están **apagados por defecto**. Los permisos se aplican en el cliente (el servidor no los puede saltear), hay un indicador visible cuando algo está activo y un atajo de "apagado de emergencia".
5. **Las claves nunca salen del servidor.** El cliente de Windows no guarda claves de proveedores; solo un token de dispositivo propio, revocable.
6. **Gratis o barato, pero con límites contemplados.** Se asume que todo plan gratuito tiene cuotas. El sistema registra el uso y el costo estimado, permite poner topes y cambia a un proveedor de respaldo cuando uno falla o se agota.
7. **Paso a paso.** Cada etapa entrega algo usable y se revisa antes de seguir.

---

## 3. Arquitectura general

```
┌──────────────────────────── PC con Windows ─────────────────────────────┐
│                                                                          │
│  Cliente Mika (Tauri)                                                    │
│  ┌───────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐ │
│  │ Micrófono │ │ Pantalla │ │  Webcam  │ │ Atajos de │ │ Avatar       │ │
│  │ + VAD     │ │ (a pedido)│ │(opcional)│ │ teclado   │ │ (overlay/OBS)│ │
│  │ + wake    │ └────┬─────┘ └────┬─────┘ └─────┬─────┘ └──────▲───────┘ │
│  │   word    │      │            │             │              │         │
│  └─────┬─────┘      ▼            ▼             ▼              │         │
│        └──────► Guardián de permisos (local, no se puede saltear) ◄──┐  │
│                         │                                        │  │   │
│                         ▼                     Reproductor de audio┘  │   │
│                 Conexión WebSocket segura (token de dispositivo)     │   │
└─────────────────────────┬────────────────────────────────────────────┘   │
                          │  eventos JSON + audio                          │
┌─────────────────────────▼──────────────── Nube ─────────────────────────┐
│  Mika Server (Python / FastAPI)                                          │
│                                                                          │
│  Gateway WS/HTTP ──► Bus de eventos ──► Orquestador ("el cerebro")       │
│                          ▲                 │                             │
│   Entradas:              │                 ├─► Constructor de contexto   │
│   · texto / voz          │                 │     personalidad + memoria  │
│   · chat Twitch/YouTube ─┤                 │     + eventos recientes     │
│   · capturas de pantalla │                 ├─► LLM (vía adaptador)       │
│   · eventos de juegos   ─┘                 ├─► Herramientas (tools)      │
│                                            └─► Salidas: texto, emoción,  │
│                                                 TTS, chat, acciones      │
│                                                                          │
│  Adaptadores de proveedores: LLM · STT · TTS · Embeddings · Visión       │
│  Memoria: corto plazo (sesión) + largo plazo (Postgres + pgvector)       │
│  Panel web de configuración (React) · Registro de uso y costos           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.1 El núcleo: bus de eventos + orquestador

Todo lo que pasa se representa como un **evento tipado** (`user.message`, `voice.transcript`, `twitch.chat`, `screen.snapshot`, `game.event`, `mika.reply`, `mika.emotion`, etc.). Los módulos publican y se suscriben a eventos; no se llaman entre sí directamente.

El **orquestador** decide *cuándo* y *a qué* responde Mika:

- Mantiene una cola de entradas con prioridades (vos hablándole > chat del stream > comentario espontáneo sobre la pantalla).
- Arma el contexto: personalidad, recuerdos relevantes, conversación reciente y estado actual (qué juego, si estás en vivo, etc.).
- Llama al LLM en modo *streaming*, interpreta la respuesta (texto, etiquetas de emoción, llamadas a herramientas) y emite eventos de salida.
- Permite interrupciones: si empezás a hablar mientras Mika habla, corta el audio y te escucha.

Al principio el bus es en memoria (asyncio). Si algún día hace falta escalar o repartir procesos, se reemplaza por Redis sin cambiar los módulos.

### 3.2 Protocolo cliente ↔ servidor

- **WebSocket** único y persistente para eventos JSON y audio (binario, Opus).
- Los esquemas de eventos se definen una sola vez (modelos Pydantic en Python) y se exportan a JSON Schema → tipos TypeScript, para que cliente, panel y servidor nunca se desincronicen.
- Versionado del protocolo desde el día uno (`protocol_version` en el saludo inicial).

### 3.3 Capa de proveedores intercambiables

Cada tipo de servicio tiene una interfaz propia (`LLMProvider`, `STTProvider`, `TTSProvider`, `EmbeddingProvider`, `VisionProvider`). Las implementaciones concretas son adaptadores.

- Para LLMs se usa **LiteLLM** *detrás* de nuestra interfaz: con una sola librería se accede a Claude, OpenAI, Gemini, Groq, DeepSeek, Mistral, OpenRouter y modelos locales vía Ollama. Si algún día LiteLLM molesta, se cambia el adaptador y el resto del código ni se entera.
- Se configuran **perfiles de modelo** por tarea, porque no todo necesita el modelo más caro:
  - `chat` — conversación principal (calidad + personalidad).
  - `fast` — respuestas rápidas al chat del stream, clasificaciones.
  - `vision` — análisis de capturas.
  - `memory` — extraer y resumir recuerdos (barato).
- Cada perfil tiene una **cadena de respaldo** (si falla o se agota la cuota del primero, se usa el siguiente).
- Se registra uso de tokens/caracteres/minutos y costo estimado por proveedor, con topes diarios/mensuales configurables.

### 3.4 Memoria

| Nivel | Qué guarda | Dónde |
|---|---|---|
| **Corto plazo** | Últimos mensajes de la sesión + resumen continuo de lo anterior | Memoria del proceso; se persiste al cerrar la sesión |
| **Episódica** | Resúmenes de conversaciones/streams pasados ("el martes jugamos Hollow Knight y te frustraste con un jefe") | Postgres + embeddings |
| **Semántica** | Hechos sobre vos y tu mundo ("tu gata se llama Luna", "no te gusta el spoiler") | Postgres + embeddings, con nivel de confianza |
| **Personalidad/estado** | Rasgos configurados, estado de ánimo actual, chistes internos | Configuración + Postgres |

- Después de cada intercambio, un modelo barato extrae hechos candidatos; se guardan, se actualizan o se descartan si contradicen algo más confiable.
- Al armar el contexto se recuperan los recuerdos más relevantes por similitud semántica + recencia + importancia.
- **Todo recuerdo se puede ver, editar y borrar desde el panel.** Opción de "modo incógnito" (no recordar nada de esta sesión).
- Recuerdos de espectadores del stream separados de los tuyos (y con política más conservadora).

### 3.5 Voz

```
Mic → VAD (local) → [wake word o tecla] → audio Opus → STT (nube) → texto
texto de respuesta → se corta por oraciones → TTS (nube, streaming) → audio → parlante + lip sync del avatar
```

- **Activación**: tecla (push-to-talk o toggle) desde la primera versión; palabra clave ("Mika") después, con **openWakeWord** corriendo localmente (liviano, gratuito, entrenable con un nombre propio).
- **VAD** (detección de voz) local con Silero VAD: evita mandar silencio a la nube → ahorra cuota y latencia.
- **STT**: Groq (Whisper, rápido y con plan gratuito limitado) como primera opción; Deepgram u OpenAI como alternativas.
- **TTS**: Azure Speech (tiene voces neuronales de **español argentino** como `es-AR-ElenaNeural`, con cuota mensual gratuita) como opción base; ElevenLabs/Cartesia/Fish Audio para una voz más expresiva o clonada si querés invertir.
- **Latencia**: el texto del LLM llega en streaming y se envía al TTS oración por oración, para que Mika empiece a hablar antes de terminar de "pensar".

### 3.6 Avatar

Se avanza por niveles, todos sobre la misma interfaz `AvatarRenderer` (recibe emoción, nivel de boca, mirada):

1. **PNGTuber**: imágenes por emoción + boca abierta/cerrada según volumen. Sirve desde el primer día.
2. **Live2D**: `pixi.js` + `pixi-live2d-display`. Expresiones mapeadas desde las emociones que emite el LLM; lip sync por amplitud (y por visemas si el TTS los provee, p. ej. Azure).
3. **3D (VRM)**: `three.js` + `@pixiv/three-vrm`.

El avatar es una página web, así que puede mostrarse:
- como **ventana transparente siempre visible** dentro del cliente de Windows, y
- como **Browser Source de OBS** (URL local servida por el cliente), sin capturar ventanas.

### 3.7 Visión (pantalla y webcam)

- **Solo a pedido** o en un modo "mirá conmigo" que vos activás, con indicador visible y posibilidad de excluir ventanas/monitores.
- El cliente captura, reduce resolución y comprime antes de enviar (menos costo y menos datos).
- Se analiza con un modelo multimodal barato (perfil `vision`), y opcionalmente detecta cambios para no mandar capturas repetidas.
- Webcam: misma lógica, apagada por defecto, nunca se graba ni se guarda la imagen salvo que lo pidas.

### 3.8 Streaming (Twitch y YouTube)

- **Twitch**: EventSub por WebSocket para leer chat y eventos (follows, subs, raids) y la API Helix para enviar mensajes. Corre en el servidor, así funciona aunque la PC esté ocupada.
- **YouTube**: YouTube Live Streaming API (`liveChatMessages`). **Ojo**: la cuota diaria de la API es limitada y leer el chat la consume; el módulo ajusta la frecuencia de consulta según la cuota restante.
- Filtros: moderación de entrada (spam, toxicidad, intentos de "jailbreak" desde el chat), lista de palabras prohibidas, límites de frecuencia de respuesta, y Mika no repite datos privados tuyos frente al chat.
- Modo stream: Mika responde una selección del chat (no todo), saluda a raids/subs y comenta lo que pasa.

### 3.9 Herramientas e integraciones

- Las capacidades de "hacer cosas" (abrir apps, buscar en internet, controlar música, leer estado de un juego) son **herramientas** que el LLM puede invocar mediante *tool calling*.
- Se usa **MCP (Model Context Protocol)** como formato de herramientas: es un estándar abierto, hay muchos servidores MCP ya hechos y permite agregar integraciones sin tocar el núcleo.
- Herramientas que actúan en tu PC corren en el cliente y **piden confirmación** según el nivel de riesgo (leer = libre; escribir/teclear/hacer clic = confirmación o lista blanca).
- Juegos: adaptadores por juego cuando existe API o mod (p. ej. Minecraft vía Mineflayer, juegos con telemetría/log), y visión de pantalla como alternativa genérica.

### 3.10 Seguridad y credenciales

- Claves de proveedores solo en el servidor, como variables de entorno / secretos de la plataforma de hosting. Nunca en el repo (`.env` en `.gitignore`, `.env.example` sin valores, escaneo de secretos en CI).
- Si se cargan claves desde el panel, se guardan **cifradas** en la base de datos con una clave maestra que vive solo en el entorno del servidor.
- El cliente se vincula una vez con un código de emparejamiento y recibe un **token de dispositivo** propio, guardado en el Administrador de credenciales de Windows. Revocable desde el panel.
- Panel web protegido con login (un solo usuario al principio, pero el modelo de datos tiene `user_id` para no cerrar puertas).
- Todo el tráfico por HTTPS/WSS.

---

## 4. Tecnologías elegidas y por qué

| Área | Elección | Por qué |
|---|---|---|
| Servidor | **Python 3.12 + FastAPI** (asyncio, WebSockets nativos) | Python tiene el mejor ecosistema de IA (SDKs de todos los proveedores, LiteLLM, procesamiento de audio). FastAPI es rápido, tipado y simple. |
| Gestión Python | **uv** | Instalación y lockfiles rapidísimos, reproducibles. |
| Validación / esquemas | **Pydantic v2** | Define eventos y configuración una sola vez; exporta JSON Schema para el frontend. |
| Abstracción de LLM | **LiteLLM** detrás de interfaz propia | Un SDK para decenas de proveedores, con conteo de tokens y costos. |
| Base de datos | **PostgreSQL + pgvector** | Una sola base para datos y búsqueda semántica. Hay planes gratuitos (Supabase, Neon) y es fácil de autoalojar. |
| ORM / migraciones | **SQLAlchemy 2 + Alembic** | Estándar maduro, migraciones versionadas. |
| Cliente Windows | **Tauri 2** (Rust + WebView2) | Instalador de ~10 MB y poco consumo de RAM, frente a ~150 MB de Electron. Permite ventanas transparentes, bandeja del sistema, atajos globales, captura de pantalla y audio desde Rust, y almacenamiento seguro de credenciales. |
| Interfaz (cliente y panel) | **React + TypeScript + Vite** | Mismo código para la UI del cliente y el panel web. Enorme ecosistema. |
| Monorepo JS | **pnpm workspaces** | Comparte paquetes (protocolo, avatar, UI) entre apps sin duplicar. |
| Avatar 2D / 3D | **pixi.js + pixi-live2d-display** / **three.js + three-vrm** | Estándares de facto para VTubers en web; funcionan igual en la app y como fuente de OBS. |
| Wake word | **openWakeWord** (local) | Gratis, liviano y entrenable con "Mika". Alternativa: Porcupine (plan personal gratuito). |
| VAD | **Silero VAD** (local) | Modelo de ~2 MB, muy preciso; evita mandar silencio a la nube. |
| STT | **Groq Whisper** → Deepgram / OpenAI | Rápido y barato; respaldo configurable. |
| TTS | **Azure Speech (voces es-AR)** → ElevenLabs / Cartesia | Acento rioplatense nativo con cuota gratuita; opciones premium para voz propia. |
| Herramientas | **MCP** | Estándar abierto, reutilizable, desacopla integraciones del núcleo. |
| Twitch / YouTube | **EventSub + Helix** / **YouTube Live Streaming API** | APIs oficiales y soportadas. |
| Contenedores | **Docker + docker compose** | Mismo entorno en local y en la nube. |
| Hosting servidor | Un VPS chico o **Fly.io / Railway** | Ver sección de costos. Se elige en la Etapa 1 según precios vigentes. |
| CI | **GitHub Actions** | Lint, tests, tipos y escaneo de secretos en cada push. |
| Calidad | **ruff + mypy + pytest** / **ESLint + Prettier + Vitest** / **clippy** | Detecta errores antes de que lleguen a la asistente. |

**Alternativas descartadas (y por qué):**
- *Electron*: más fácil, pero pesado; contradice el objetivo de no ocupar recursos.
- *Cliente 100% en Python*: empaquetado grande y frágil en Windows, UI pobre para un avatar.
- *Node.js para el servidor*: viable, pero el ecosistema de audio/IA en Python es más completo.
- *Todo local (modelos en la PC)*: requiere GPU y gigas de disco; se deja como opción vía Ollama para quien quiera.
- *LangChain como núcleo*: agrega abstracción y cambios frecuentes; preferimos un orquestador propio, chico y entendible.

---

## 5. Estructura de carpetas

```
Rei-AI-Assistant/
├── PLAN.md                      # este documento
├── README.md
├── .env.example                 # variables necesarias, sin valores
├── .gitignore
├── docker-compose.yml           # servidor + Postgres para desarrollo local
├── pnpm-workspace.yaml
├── .github/workflows/           # CI: lint, tests, tipos, escaneo de secretos
│
├── apps/
│   ├── server/                  # Mika Server (Python / FastAPI)
│   │   ├── pyproject.toml
│   │   ├── alembic/             # migraciones de base de datos
│   │   ├── src/mika/
│   │   │   ├── main.py          # arranque de la app
│   │   │   ├── config/          # carga de configuración y secretos
│   │   │   ├── core/
│   │   │   │   ├── events.py    # tipos de eventos + bus
│   │   │   │   ├── orchestrator.py
│   │   │   │   ├── context.py   # constructor de contexto/prompt
│   │   │   │   └── persona.py   # personalidad
│   │   │   ├── providers/       # adaptadores intercambiables
│   │   │   │   ├── llm/  stt/  tts/  embeddings/  vision/
│   │   │   ├── memory/          # corto plazo, largo plazo, extracción
│   │   │   ├── modules/         # capacidades enchufables
│   │   │   │   ├── voice/  vision/  twitch/  youtube/  games/
│   │   │   ├── tools/           # registro de herramientas / cliente MCP
│   │   │   ├── api/             # rutas HTTP + WebSocket
│   │   │   ├── auth/            # login del panel, tokens de dispositivo
│   │   │   ├── usage/           # registro de uso, costos y topes
│   │   │   └── db/              # modelos y sesión de base de datos
│   │   └── tests/
│   │
│   ├── desktop/                 # Cliente Windows (Tauri 2)
│   │   ├── src-tauri/           # Rust: audio, captura, atajos, permisos, credenciales
│   │   └── src/                 # React: chat, ajustes rápidos, ventana del avatar
│   │
│   └── web/                     # Panel de configuración (React)
│
├── packages/
│   ├── protocol/                # tipos TS generados desde los esquemas del servidor
│   ├── avatar/                  # AvatarRenderer: PNGTuber, Live2D, VRM
│   └── ui/                      # componentes compartidos
│
├── config/
│   ├── personas/
│   │   └── mika.yaml            # personalidad por defecto
│   └── models.example.yaml      # perfiles de modelo y cadenas de respaldo
│
├── assets/                      # sprites, modelos Live2D/VRM (los pesados, fuera de git o con LFS)
└── docs/                        # decisiones de arquitectura (ADR), guías
```

### Ejemplo de personalidad (`config/personas/mika.yaml`)

```yaml
name: Mika
language: es-AR          # voseo, modismos rioplatenses moderados
traits:
  calidez: 0.9
  humor: 0.7
  energia: 0.4           # tranquila
  formalidad: 0.1
style:
  - Hablás de vos, con naturalidad, sin exagerar el lunfardo.
  - Respuestas cortas cuando estás hablando en voz alta.
  - Cariñosa pero no empalagosa; podés bromear y tomarle el pelo con cariño.
boundaries:
  - No inventás recuerdos: si no sabés algo, lo decís.
  - En stream no revelás datos privados del usuario.
voice:
  provider: azure
  voice_id: es-AR-ElenaNeural
emotions: [neutral, feliz, divertida, sorprendida, triste, enojada, pensativa, avergonzada]
```

---

## 6. Etapas de desarrollo

Cada etapa termina con: código + tests + documentación + una demo que puedas probar. **No se avanza a la siguiente sin tu aprobación.**

### Etapa 0 — Fundaciones
**Objetivo:** dejar el terreno listo para construir sin deuda.
- Estructura del monorepo, `pnpm` + `uv`, Docker Compose (servidor + Postgres).
- Configuración: carga de `.env`, `.env.example`, validación con Pydantic.
- CI en GitHub Actions: lint, tipos, tests, escaneo de secretos.
- Servidor FastAPI mínimo con `/health`.
- README con cómo levantar todo.

**Aceptación:** `docker compose up` levanta el servidor y la base; CI en verde.

### Etapa 1 — Núcleo conversacional (texto)
**Objetivo:** hablar con Mika por texto, con su personalidad.
- Bus de eventos + orquestador básico.
- Interfaz `LLMProvider` + adaptador LiteLLM, con al menos 2 proveedores configurados (p. ej. Gemini y Claude) y respaldo automático.
- Perfiles de modelo (`chat`, `fast`) en `models.yaml`.
- Personalidad desde `mika.yaml` → prompt de sistema con ejemplos en rioplatense.
- Memoria de corto plazo (ventana + resumen).
- WebSocket con respuestas en streaming + chat de prueba mínimo (web o CLI).
- Registro de uso y costo por llamada.

**Aceptación:** conversás con Mika, cambiás de modelo editando la configuración sin tocar código, y si un proveedor falla responde el de respaldo.

### Etapa 2 — Memoria de largo plazo
- Postgres + pgvector, migraciones con Alembic.
- Extracción automática de hechos y resúmenes episódicos con el perfil `memory`.
- Recuperación por relevancia + recencia + importancia al construir el contexto.
- API para listar/editar/borrar recuerdos; modo incógnito.

**Aceptación:** le contás algo, cerrás la sesión, y días después lo recuerda en contexto; podés borrarlo y lo olvida.

### Etapa 3 — Panel web de configuración
- Login (usuario único) y protección de rutas.
- Editores: personalidad, perfiles de modelo, proveedores, voz (preparado), permisos.
- Gestión de memoria (buscar, editar, borrar).
- Gestión de claves API cifradas.
- Tablero de uso y costos con topes.
- Despliegue del servidor en la nube (elección final de hosting).

**Aceptación:** podés cambiar personalidad y modelos desde el navegador y ver cuánto gastaste.

### Etapa 4 — Cliente de Windows básico
- App Tauri: bandeja del sistema, ventana de chat, inicio con Windows (opcional).
- Emparejamiento con el servidor y token en el Administrador de credenciales.
- **Guardián de permisos** local + indicadores + atajo de apagado de emergencia.
- Atajo global para abrir el chat.
- Reconexión automática.
- Instalador firmado/empaquetado (MSI/NSIS).

**Aceptación:** instalás un ejecutable chico, lo vinculás, y chateás con Mika desde la bandeja.

### Etapa 5 — Voz
- Captura de micrófono en el cliente + Silero VAD + envío Opus.
- Push-to-talk / toggle por tecla.
- Interfaces `STTProvider` / `TTSProvider` con 2 opciones cada una.
- TTS en streaming por oración; reproducción en el cliente.
- Interrupción (si hablás, Mika se calla y escucha).
- Wake word "Mika" con openWakeWord.

**Aceptación:** conversación por voz fluida, con latencia objetivo < 1,5–2 s hasta la primera palabra.

### Etapa 6 — Avatar
- Paquete `avatar` con `AvatarRenderer`.
- PNGTuber → Live2D (modelo de prueba gratuito, luego el tuyo).
- Emociones emitidas por el LLM → expresiones; lip sync por amplitud/visemas.
- Ventana transparente siempre visible + URL para Browser Source de OBS.

**Aceptación:** Mika aparece en pantalla y en OBS, mueve la boca al hablar y cambia de expresión.

### Etapa 7 — Visión (pantalla y webcam)
- Captura de pantalla a pedido desde el cliente (con selección de monitor/ventana y exclusiones).
- Modo "mirá conmigo" con frecuencia configurable y detección de cambios.
- Perfil `vision` con proveedor multimodal.
- Webcam opcional, misma política.

**Aceptación:** "Mika, ¿qué ves?" funciona; nada se captura sin que lo actives y siempre hay indicador.

### Etapa 8 — Integración con streams
- Módulo Twitch (EventSub + Helix) y módulo YouTube (con control de cuota).
- Cola con prioridades y selección de mensajes a responder.
- Moderación y protección contra manipulación desde el chat.
- Reacciones a eventos (raids, subs, follows).
- Memoria de espectadores separada.

**Aceptación:** en un stream de prueba, Mika lee y responde el chat de ambas plataformas sin saturar ni decir cosas que no debe.

### Etapa 9 — Herramientas, apps y videojuegos
- Sistema de herramientas sobre MCP (servidor y cliente).
- Herramientas locales con niveles de riesgo y confirmación.
- Primeras integraciones: música, navegador/búsqueda, abrir apps.
- Primer adaptador de juego (a elegir juntos según lo que jugás).

**Aceptación:** Mika puede hacer acciones concretas que le pedís, con tu confirmación cuando corresponde.

### Etapa 10 — Pulido y extras
- Comportamiento proactivo moderado (comentarios espontáneos, recordatorios), siempre configurable.
- Avatar 3D (VRM).
- Observabilidad (logs estructurados, métricas, alertas de cuota).
- Copias de seguridad de la memoria, exportación/importación.
- Optimización de latencia y costos.

> El orden de las etapas 5 a 9 se puede reordenar según tu prioridad (por ejemplo, adelantar el avatar o los streams). Las etapas 0–4 son la base que todo lo demás necesita.

---

## 7. Costos estimados y límites de planes gratuitos

> Los precios y cuotas cambian seguido: **se verifican al momento de implementar cada etapa**. Estas son órdenes de magnitud, no promesas.

| Servicio | Opción de bajo costo | Límite a tener en cuenta |
|---|---|---|
| Hosting del servidor | VPS chico (~4–6 USD/mes) o Fly.io/Railway por uso | Los planes "gratis" suelen dormir la app o desaparecer; para voz en tiempo real conviene algo siempre encendido. |
| Base de datos | Supabase / Neon (plan gratuito) o Postgres en el mismo VPS | Espacio limitado; Supabase pausa proyectos inactivos. |
| LLM | Gemini Flash / Groq / DeepSeek para tareas baratas; Claude u otro para el chat principal | Límites por minuto y por día en planes gratuitos; el costo crece con conversaciones largas (por eso la memoria resume). |
| STT | Groq Whisper | Límite de requests/min y horas de audio por día. |
| TTS | Azure Speech | Cuota mensual de caracteres gratis; luego cobra por millón de caracteres. |
| YouTube API | Cuota diaria gratuita | Leer chat consume cuota; streams largos pueden agotarla. |
| Twitch API | Gratis | Límites de frecuencia de mensajes. |

**Mitigaciones incluidas en el diseño:** topes de gasto configurables, cadenas de respaldo, VAD local (no se manda silencio), capturas comprimidas y deduplicadas, resumen de memoria para acortar el contexto, y modelos baratos para tareas auxiliares.

---

## 8. Riesgos y cómo se manejan

| Riesgo | Mitigación |
|---|---|
| Latencia de voz alta por depender de la nube | Streaming en todas las etapas (LLM → TTS por oración), servidor en una región cercana (São Paulo / EE. UU. Este), proveedores rápidos. |
| Cuotas gratuitas agotadas en pleno stream | Registro de uso, alertas, respaldo automático a otro proveedor. |
| Privacidad (pantalla, webcam, memoria) | Permisos locales apagados por defecto, indicadores, borrado de recuerdos, modo incógnito, nada se guarda sin permiso. |
| Manipulación desde el chat del stream | Moderación, separación de roles en el prompt, herramientas peligrosas deshabilitadas en modo stream. |
| Dependencia de un proveedor | Interfaces propias + adaptadores + configuración por perfiles. |
| Crecimiento desordenado del código | Monorepo con límites claros, protocolo tipado, tests y CI desde la Etapa 0, ADRs para decisiones. |
| Licencias de modelos Live2D / voces clonadas | Usar modelos con licencia clara; documentar en `assets/`. |

---

## 9. Forma de trabajo

1. Al empezar cada etapa, se detalla su alcance en un issue o sección de este documento.
2. Se desarrolla en una rama, con commits chicos y descriptivos.
3. Al terminar: demo, resumen de lo hecho y de lo que quedó pendiente.
4. **Revisás y aprobás** (o pedís cambios) antes de pasar a la siguiente.
5. Las decisiones importantes se registran en `docs/adr/`.

### Decisiones abiertas para charlar antes o durante la Etapa 0–1
- ¿Qué proveedores de LLM ya tenés (o querés probar) primero?
- ¿Preferís un VPS propio o una plataforma administrada para el servidor?
- ¿Tenés ya un modelo Live2D/VRM o arrancamos con uno de prueba?
- ¿Qué juegos y plataformas de stream usás más? (define el orden de las etapas 8–9)
