# UnoArena — Guía de Entrega DDD

> Proyecto académico · Arquitectura de Microservicios · ITBA 2025

---

## 1. Contexto del Proyecto

UnoArena es una plataforma backend de Uno competitivo en tiempo real que soporta salas ad-hoc (2-10 jugadores) y torneos masivos de eliminación (hasta 1M de jugadores). Las acciones se envían por REST y el estado se difunde por SSE.

**Constraints clave:**
- Proyecto académico de aprendizaje, no producción.
- Sin dependencias externas de identidad (no Keycloak, Auth0, etc.).
- Foco en definición de requerimientos y diseño, no en implementación.

---

## 2. Event Storming (Obligatorio)

El documento debe incluir la salida de un ejercicio de Event Storming como punto de partida del análisis de dominio. Esto implica:

### 2.1 Qué incluir

- **Eventos de dominio (naranjas):** Todo lo que sucede en el sistema en tiempo pasado. Ejemplos: `GameCreated`, `PlayerJoined`, `CardPlayed`, `UnoCallMade`, `UnoCallChallenged`, `TurnSkipped`, `DirectionReversed`, `DrawPenaltyApplied`, `GameCompleted`, `TournamentRoundStarted`, `TournamentRoundCompleted`, `PlayerEliminated`, `EloUpdated`, `BracketAdvanced`.
- **Comandos (azules):** La intención del usuario o sistema que dispara el evento. Ejemplos: `CreateRoom`, `JoinRoom`, `PlayCard`, `CallUno`, `ChallengeUnoCall`, `DrawCard`, `StartTournament`, `AdvanceBracket`.
- **Actores (amarillos):** Quién ejecuta el comando: `Player`, `Spectator`, `TournamentOrchestrator`, `System (RNG)`.
- **Aggregates (amarillos):** La entidad que recibe el comando y garantiza consistencia: `Room`, `GameState`, `Tournament`, `Bracket`, `PlayerRanking`.
- **Políticas (púrpura):** Reglas de negocio reactivas. Ejemplos: "Cuando un jugador juega su penúltima carta y no llama Uno, cualquier otro jugador puede desafiarlo dentro de una ventana de tiempo" · "Cuando se completa un game, el orquestador avanza la bracket si todos los games de la ronda terminaron" · "Cuando se detecta una acción contra un estado stale, se rechaza con conflicto".
- **Hotspots (rojos):** Puntos de conflicto o ambigüedad que requieren decisión. Ejemplo: ¿Cuánto dura la ventana para desafiar un Uno call? ¿Qué pasa si dos jugadores intentan jugar simultáneamente?

### 2.2 Formato esperado

Una representación visual (diagrama, foto de post-its, o esquema digital) que muestre el flujo temporal de izquierda a derecha con los elementos del Event Storming agrupados. No necesita ser perfecto; necesita ser completo para los flujos principales.

---

## 3. Strategic Design

### 3.1 Identificación de Subdominios

Clasificar cada subdomain en Core, Supporting o Generic, y justificar por qué.

| Subdomain | Tipo | Justificación |
|---|---|---|
| Game Engine (lógica de Uno) | **Core** | Diferenciador principal: reglas, concurrencia, integridad del estado |
| Tournament Management | **Core** | Orquestación de brackets, escalamiento masivo, progresión |
| Ranking & Elo | **Supporting** | Necesario pero la lógica de cálculo no es diferenciadora |
| Room Lifecycle | **Supporting** | Manejo de estados de sala (waiting → in_progress → completed) |
| Player/User Management | **Generic** | CRUD de usuarios, perfiles — estándar |
| Notifications / SSE Broadcasting | **Generic** | Distribución de eventos a clientes — infraestructura |

### 3.2 Bounded Contexts

Definir los Bounded Contexts con sus responsabilidades claras. Cada BC posee sus datos; ningún otro accede directamente a su storage.

Bounded Contexts sugeridos para UnoArena:

- **Game Context:** Estado de la partida, validación de jugadas, deck, turnos, reglas de Uno.
- **Tournament Context:** Brackets, rondas, orquestación de fases, avance de ganadores.
- **Ranking Context:** Cálculo de Elo, estadísticas por jugador, leaderboards.
- **Room Context:** Ciclo de vida de salas, matchmaking, conexión de jugadores.
- **Player Context:** Identidad, perfil, autenticación.
- **Broadcast Context:** Fan-out de SSE, distribución de delta patches a clientes.

### 3.3 Context Map

Incluir un mapa que muestre las relaciones entre BCs. Usar las relaciones estándar de DDD:

- Upstream / Downstream (U/S)
- Published Language (eventos de dominio como contrato)
- Anti-Corruption Layer donde sea necesario
- Conformist, Shared Kernel, etc. si aplican

Ejemplo de relación: Game Context (upstream) publica `GameCompleted` → Tournament Context (downstream) consume el evento para avanzar la bracket.

### 3.4 Ubiquitous Language

Definir un glosario con los términos clave del dominio. Todos los miembros del equipo deben usar estos términos consistentemente en documentación y código.

Ejemplos mínimos:

| Término | Definición |
|---|---|
| Room | Instancia de una partida con 2-10 jugadores |
| GameState | Estado actual inmutable de una partida en un punto del tiempo |
| Turn | Turno activo de un jugador dentro de una partida |
| Deck | Mazo de cartas administrado server-side por el servicio de RNG |
| UnoCall | Declaración obligatoria al quedar con una carta |
| Challenge | Acción de un jugador denunciando que otro no llamó Uno |
| Tournament | Competencia de eliminación con múltiples rondas |
| Bracket | Estructura de enfrentamientos dentro de una ronda del torneo |
| Round | Fase de un torneo donde N partidas se juegan en paralelo |
| Elo | Rating numérico que refleja el nivel competitivo de un jugador |
| DeltaPatch | Cambio incremental del estado de juego enviado por SSE |
| SequenceNumber | Número de orden que serializa las acciones concurrentes |

---

## 4. Tactical Design

### 4.1 Por cada Bounded Context, definir:

**Entities:** Objetos con identidad única que persiste a través de cambios de estado. Ejemplo: `Player` (tiene un ID que no cambia aunque cambie su Elo o su nombre).

**Value Objects:** Objetos inmutables definidos por sus atributos, no por identidad. Ejemplo: `Card` (definida por color + valor; dos cartas iguales son intercambiables), `EloRating`, `RoomCode`.

**Aggregates:** Cluster de entidades y VOs tratados como unidad transaccional. Definir claramente el Aggregate Root. Ejemplo: `GameState` como aggregate root que contiene el deck, las manos de los jugadores, el turno actual y la dirección. Toda modificación al estado del juego pasa por el aggregate root.

**Reglas de diseño de Aggregates:**
- Mantenerlos lo más pequeños posible.
- Referenciar otros aggregates solo por ID, nunca por referencia directa.
- Una transacción = un aggregate. Si necesitás modificar dos aggregates, usá eventos de dominio.

**Repositories:** Interfaz definida en el dominio (no la implementación). Debe reflejar intención de negocio, no operaciones de base de datos. Ejemplo: `findActiveGamesByTournamentRound(roundId)` en vez de `SELECT * FROM games WHERE...`.

**Domain Services:** Lógica que no pertenece naturalmente a una Entity ni a un VO. Ejemplo: servicio de validación de jugadas que requiere conocer el estado completo del juego.

### 4.2 Domain Events

Listar los eventos de dominio principales con su payload conceptual. No especificar tecnología de transporte (no mencionar Kafka, Redis, RabbitMQ, etc.), sino describir qué dato viaja y quién lo consume.

Ejemplo:
- **Evento:** `CardPlayed` → **Payload:** roomId, playerId, card, resultingGameState, sequenceNumber → **Consumidores:** Broadcast Context (para notificar a clientes), Game Context (para evaluar reglas reactivas como penalidades).

---

## 5. Decisiones de Arquitectura (Flujo, no Tecnología)

El documento debe explicar las decisiones de diseño en términos de flujo y justificación de negocio, sin atarse a tecnologías específicas. Describir el "qué" y el "por qué", no el "con qué".

### Ejemplos de cómo redactar:

**Bien (enfoque en flujo):**
> "Las acciones de los jugadores se envían de forma síncrona y el estado se propaga de forma asíncrona a todos los clientes conectados. Elegimos esta separación porque la lógica de juego necesita serializar acciones concurrentes, mientras que la distribución del estado es un problema de fan-out que escala independientemente."

**Mal (enfoque en tecnología):**
> ~~"Usamos Redis Streams para publicar delta patches que el tier de SSE broadcasters consume con un consumer group de Redis."~~

### Decisiones que deben estar justificadas:

- Por qué separar la lógica de juego de la distribución de estado.
- Cómo se resuelve la concurrencia de acciones simultáneas (sequence numbers, rechazo optimista).
- Por qué los torneos se orquestan como sagas con eventos en vez de llamadas síncronas.
- Cómo se mantiene la consistencia eventual entre Game Context y Tournament Context.
- Por qué el cálculo de Elo se hace de forma asíncrona como read model (patrón CQRS).
- Cómo se audita la integridad del juego (log inmutable de acciones).

---

## 6. Checklist de Entrega Mínima

- [ ] Event Storming: diagrama completo con eventos, comandos, actores, aggregates, políticas y hotspots para los flujos principales (partida y torneo).
- [ ] Subdominios identificados y clasificados (Core / Supporting / Generic) con justificación.
- [ ] Bounded Contexts definidos con responsabilidades claras.
- [ ] Context Map mostrando relaciones entre BCs (U/S, ACL, Published Language, etc.).
- [ ] Ubiquitous Language: glosario de al menos 15 términos del dominio.
- [ ] Tactical Design por cada BC core: entities, value objects, aggregates (con aggregate root), repositories (interfaz), domain events.
- [ ] Domain Events: lista con payload conceptual y consumidores.
- [ ] Decisiones de arquitectura explicadas como flujos de negocio, sin especificaciones tecnológicas.
- [ ] Rich Domain Model: demostrar que la lógica de negocio vive en las entidades/aggregates, no en services anémicos externos.

---

## 7. Errores Comunes a Evitar

- **Modelo anémico:** Entidades que son solo getters/setters con toda la lógica en services externos. La lógica de validar una jugada debe estar en el aggregate `GameState`, no en un `GameService`.
- **Compartir base de datos entre BCs:** Cada Bounded Context es dueño de sus datos. Si Tournament necesita saber el resultado de un game, lo obtiene por evento, no por query directa.
- **Llamadas síncronas entre BCs:** Genera acoplamiento temporal. Preferir eventos de dominio para comunicación entre contextos.
- **Bounded Contexts demasiado granulares:** No crear un BC por cada entidad. Un BC agrupa conceptos que cambian juntos y comparten lenguaje.
- **Olvidar los hotspots:** El Event Storming debe surfacear las ambigüedades (ej: ¿qué pasa si dos jugadores juegan al mismo tiempo? ¿cuánto dura la ventana de challenge?). Documentarlas aunque no tengan respuesta definitiva.
- **Mezclar flujo con tecnología:** Este documento es de diseño de dominio. Las decisiones de infra (qué broker, qué DB) van en otro documento.
