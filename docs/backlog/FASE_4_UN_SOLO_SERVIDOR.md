# FASE 4: UN SOLO SERVIDOR PARA TODAS LAS SESIONES, Y EL RELOJ CORRE UNA VEZ

**Destraba**: que abrir 5 sesiones de Claude no levante 5 copias de hylian-mcp en el Mac Mini, ni corra 5 veces la
sincronización diaria y el respaldo semanal.
**Estado**: sin rama
**Tipo**: infra
**Dependencias**: ninguna
**Fuente**: Freddy, 2026-11-14: «¿no es solo un MCP como singleton que responde a cualquier agente?… puedo tener miles de
clientes conectados a un solo MCP… tenemos que pensar en cómo mantenerlo sostenible y eficiente en recursos»

## El hueco, medido

Medido el 2026-11-13 y el 2026-11-14:

- **Cada sesión arranca su propia copia.** El cliente se configura por stdio:
  `ssh mac-mini docker run -i --rm … hylian-mcp:local`, en `~/.claude.json` §mcpServers.hylian-mcp. Con stdio cada
  cliente es dueño de su proceso: con 3 sesiones abiertas había 3 contenedores.
- **Cada copia prende el reloj.** `startJobs` corre en todas (`src/main.ts:53`), y en producción los jobs se activan
  (`src/jobs/index.ts:22`). `daily-sync` y `weekly-backup` corrían 3 veces.
- **El modo HTTP ya existe, pero atiende a un solo cliente.** `src/main.ts:14-46` crea **un**
  `StreamableHTTPServerTransport` con `sessionIdGenerator`. Ese transporte con estado lleva una sola sesión, así que el
  segundo cliente choca. Además solo acepta `POST /mcp` (sin `GET` ni `DELETE`) y no pide token.
- El patrón bueno ya corre en el Mac Mini: ladder-lab sirve su MCP por HTTP en `:8446/mcp`, con token, como un solo
  servicio.

## Las decisiones

### D1 · Un contenedor siempre encendido, por HTTP; los clientes se conectan por URL — 2026-11-14 `[código]`

- Un servicio en compose con `restart: unless-stopped`, en `127.0.0.1`, publicado con `tailscale serve` en su propio
  puerto HTTPS (`:8447`, el siguiente libre).
- El cliente pasa de `ssh … docker run` a `{"type":"http","url":"https://mac-mini-server…:8447/mcp"}` con su token,
  igual que ladder-lab.

### D2 · Un transporte por sesión; el reloj, una vez por proceso — 2026-11-14 `[código]`

- Un mapa `sessionId → transporte`. Cada `initialize` crea el suyo, y `GET`/`DELETE /mcp` lo usan o lo cierran.
- Es el patrón del SDK para varios clientes.
- `startJobs` queda donde está: con un solo proceso, corre una sola vez.

### D3 · Token como ladder-lab — 2026-11-14 `[código]`

Va en `Authorization: Bearer`, el valor en el `.env` del Mac Mini (600) y en `settings.local`/`~/.claude.json` del
cliente. Sin token la respuesta es 401, y eso se prueba mandando basura (§12).

## Abierto

Ninguno.

## Tareas

- [ ] **Tarea 1** · `src/main.ts`: transporte por sesión (D2) y `GET`/`DELETE /mcp`. Test con dos clientes simultáneos
      que listan tools; primero en rojo, con el código de hoy.
- [ ] **Tarea 2** · token (D3), con test de 401 y de 200.
- [ ] **Tarea 3** · servicio en el compose del Mac Mini, `tailscale serve --https=8447`, y el deploy de su workflow.
- [ ] **Tarea 4** · cliente: `~/.claude.json` del ThinkPad pasa a HTTP. El nodo «hylian-mcp» del grafo (Mis máquinas →
      Mac Mini) dice la URL.
- [ ] **Tarea 5** · README y `docs/decisions/DECISION_DEPLOY_MAC_MINI.md`: stdio queda solo para desarrollo local.

## Criterios de Aceptación

- [ ] 1 · Con 3 sesiones de Claude abiertas, `docker ps` en el Mac Mini muestra **un** contenedor de hylian-mcp.
- [ ] 2 · En un día, el log muestra `daily-sync` **una** vez.
- [ ] 3 · Sin token: 401; con token basura: 401.

## QA

- [ ] Listo para PR
- [ ] Mergeado
- [ ] Adoptado
