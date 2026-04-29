# Unidad 7

[ACTIVIDADES](https://juanferfranco.github.io/interactivos1-2026-10/units/unit7/)

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Actividad 02

<details>
<summary><b>OpenStageControlAdapter.js</b></summary>

``` js
const BaseAdapter = require("./BaseAdapter");
const osc = require("osc");

class OpenStageControlAdapter extends BaseAdapter {
  constructor({ port = 9000, verbose = false } = {}) {
    super();
    this.port = port;
    this.verbose = verbose;
    this.udpPort = null;
  }

  async connect() {
    if (this.connected) return;

    this.udpPort = new osc.UDPPort({
      localAddress: "0.0.0.0",
      localPort: this.port,
      metadata: false
    });

    this.udpPort.on("message", (oscMsg) => {
      const normalized = this._normalize(oscMsg);
      if (!normalized) return;

      if (this.verbose) {
        console.log("[OpenStageControlAdapter]", normalized);
      }

      this.onData?.(normalized);
    });

    this.udpPort.on("error", (err) => {
      this.onError?.(`OSC error: ${err.message || err}`);
    });

    this.udpPort.open();

    this.connected = true;
    this.onConnected?.(`Open Stage Control escuchando OSC UDP en puerto ${this.port}`);
  }

  async disconnect() {
    if (!this.connected) return;

    if (this.udpPort) {
      this.udpPort.close();
      this.udpPort = null;
    }

    this.connected = false;
    this.onDisconnected?.("Open Stage Control adapter detenido");
  }

  getConnectionDetail() {
    return `Open Stage Control OSC UDP port ${this.port}`;
  }

  _normalize(oscMsg) {
    if (!oscMsg || !oscMsg.address) return null;

    return {
      type: "osc",
      payload: {
        address: oscMsg.address,
        args: oscMsg.args ?? []
      }
    };
  }
}

module.exports = OpenStageControlAdapter;
```
</details>

---

<details>
<summary><b>BridgeServer.js</b></summary><br>


<details>
<summary>Codigo</summary>

``` js

//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}


const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitAdapter2 = require("./adapters/MicrobitAdapter2");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");
const OpenStageControlAdapter = require("./adapters/OpenStageControlAdapter");

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};


function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try {
    return JSON.parse(s);

  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return [new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE })];
  }

  if (DEVICE === "microbitv2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return [new MicrobitAdapter2({ path, baud: BAUD, verbose: VERBOSE })];
  }

  if (DEVICE === "microbit-bin") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    return [new MicrobitBinaryAdapter({ path, baud: BAUD })];
  }

  if (DEVICE === "strudel") {
    const port = parseInt(getArg("strudelPort", "8080"), 10);
    return [new StrudelAdapter({ port, verbose: VERBOSE })];
  }

  if (DEVICE === "strudel-osc") {
    const strudelPort = parseInt(getArg("strudelPort", "8080"), 10);
    const oscPort = parseInt(getArg("oscPort", "9000"), 10);

    return [
      new StrudelAdapter({ port: strudelPort, verbose: VERBOSE }),
      new OpenStageControlAdapter({ port: oscPort, verbose: VERBOSE })
    ];
  }

  return [new SimAdapter({ hz: SIM_HZ })];
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapters = await createAdapter();

  for (const adapter of adapters) {
    adapter.onConnected = (detail) => {
      log.info(`[ADAPTER] Connected: ${detail}`);
      status(wss, "connected", detail);
    };

    adapter.onDisconnected = (detail) => {
      log.warn(`[ADAPTER] Disconnected: ${detail}`);
      status(wss, "disconnected", detail);
    };

    adapter.onError = (detail) => {
      log.error(`[ADAPTER] Error: ${detail}`);
      status(wss, "error", detail);
    };

    adapter.onData = (d) => {
      if (d.type === "strudel" || d.type === "osc") {
        broadcast(wss, d);
        return;
      }

      broadcast(wss, {
        type: "microbit",
        x: d.x,
        y: d.y,
        btnA: !!d.btnA,
        btnB: !!d.btnB,
        t: nowMs(),
      });
    };
  }

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const anyConnected = adapters.some(a => a.connected);
    const state = anyConnected ? "connected" : "ready";

    const detail = anyConnected
      ? adapters.map(a => a.getConnectionDetail()).join(" + ")
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapters connect`);

        try {
          for (const adapter of adapters) {
            if (!adapter.connected) {
              await adapter.connect();
            }
          }
        } catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapters disconnect`);

        try {
          for (const adapter of adapters) {
            if (adapter.connected) {
              await adapter.disconnect();
            }
          }
          status(wss, "disconnected", "all adapters disconnected");
        } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "setSimHz") {
        const simAdapter = adapters.find(a => a instanceof SimAdapter);

        if (simAdapter) {
          log.info(`Setting Sim Hz to ${msg.hz}`);
          await simAdapter.handleCommand(msg);
          status(wss, "connected", `sim hz=${simAdapter.hz}`);
        }
        return;
      }

      if (msg.cmd === "setLed") {
        try {
          for (const adapter of adapters) {
            await adapter.handleCommand?.(msg);
          }
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");
        for (const adapter of adapters) {
          adapter.disconnect();
        }
      }
    });
  });

  if (DEVICE === "sim") {
    for (const adapter of adapters) {
      await adapter.connect();
    }
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```
</details>

---

<details>
<summary>Cambios realizados</summary><br>
  
en `bridServer.js`:

``` js
const OpenStageControlAdapter = require("./adapters/OpenStageControlAdapter");
```

en `createAdapter()` añadir:

``` js
if (DEVICE === "strudel-osc") {
    const strudelPort = parseInt(getArg("strudelPort", "8080"), 10);
    const oscPort = parseInt(getArg("oscPort", "9000"), 10);

    return [
      new StrudelAdapter({ port: strudelPort, verbose: VERBOSE }),
      new OpenStageControlAdapter({ port: oscPort, verbose: VERBOSE })
    ];
  }
```

Al principios de `main()` cambiar:

``` js
const adapter = await createAdapter();
```

por 

``` js
const adapters = await createAdapter();
```
y el siguiente bloque de

``` js
adapter.onConnected = ...
adapter.onDisconnected = ...
adapter.onError = ...
adapter.onData = ...
```

reemplazarlo por

``` js
for (const adapter of adapters) {
  adapter.onConnected = (detail) => {
    log.info(`[ADAPTER] Connected: ${detail}`);
    status(wss, "connected", detail);
  };

  adapter.onDisconnected = (detail) => {
    log.warn(`[ADAPTER] Disconnected: ${detail}`);
    status(wss, "disconnected", detail);
  };

  adapter.onError = (detail) => {
    log.error(`[ADAPTER] Error: ${detail}`);
    status(wss, "error", detail);
  };

  adapter.onData = (d) => {
    if (d.type === "strudel" || d.type === "osc") {
      broadcast(wss, d);
      return;
    }

    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs(),
    });
  };
}
```

Dentro de `wss.on("connection")`, cambia referencias a `adapter`

cambiar

``` js
const state = adapter.connected ? "connected" : "ready";

const detail = adapter.connected
  ? adapter.getConnectionDetail()
  : `bridge (${DEVICE})`;
```

por

```js
const anyConnected = adapters.some(a => a.connected);
const state = anyConnected ? "connected" : "ready";

const detail = anyConnected
  ? adapters.map(a => a.getConnectionDetail()).join(" + ")
  : `bridge (${DEVICE})`;
```

Cambiar el comando `connect` (aprox linea 185), reemplazar el contenido de `if (msg.cmd === "connect") {` por

``` js
if (msg.cmd === "connect") {
  log.info(`[NETWORK] Client requested adapters connect`);

  try {
    for (const adapter of adapters) {
      if (!adapter.connected) {
        await adapter.connect();
      }
    }
  } catch (e) {
    const detail = `connect failed: ${e.message || e}`;
    log.error(`[ADAPTER] ` + detail);
    status(wss, "error", detail);
  }
  return;
}
```
hacer lo mismo con `disconnect`, que esta inmediatamente despues de este

``` js
if (msg.cmd === "disconnect") {
  log.info(`[NETWORK] Client requested adapters disconnect`);

  try {
    for (const adapter of adapters) {
      if (adapter.connected) {
        await adapter.disconnect();
      }
    }
    status(wss, "disconnected", "all adapters disconnected");
  } catch (e) {
    const detail = `disconnect failed: ${e.message || e}`;
    log.error(`[ADAPTER] ` + detail);
    status(wss, "error", detail);
  }
  return;
}
```

en `if (msg.cmd === "setSimHz" && adapters instanceof SimAdapter)`, quedaria

``` js
if (msg.cmd === "setSimHz") {
  const simAdapter = adapters.find(a => a instanceof SimAdapter);

  if (simAdapter) {
    log.info(`Setting Sim Hz to ${msg.hz}`);
    await simAdapter.handleCommand(msg);
    status(wss, "connected", `sim hz=${simAdapter.hz}`);
  }
  return;
}
```

y en `if (msg.cmd === "setLed")` seria

``` js
if (msg.cmd === "setLed") {
  try {
    for (const adapter of adapters) {
      await adapter.handleCommand?.(msg);
    }
  } catch (e) {
    const detail = `command failed: ${e.message || e}`;
    log.error(`[ADAPTER] ` + detail);
    status(wss, "error", detail);
  }
  return;
}
```

y en `if (DEVICE === "sim")` usar

``` js
if (DEVICE === "sim") {
  for (const adapter of adapters) {
    await adapter.connect();
  }
}
```
</details>

</details>

---

<details>
<summary><b>bridgeClient.js</b></summary><br>

<details>
<summary>Codigo</summary><br>

``` js
class BridgeClient {
  constructor(url = "ws://127.0.0.1:8081") {
    this._url = url;
    this._ws = null;
    this._isOpen = false;

    this._onData = null;
    this._onConnect = null;
    this._onDisconnect = null;
    this._onStatus = null;
  }

  get isOpen() {
    return this._isOpen;
  }

  onData(callback) { this._onData = callback; }
  onConnect(callback) { this._onConnect = callback; }
  onDisconnect(callback) { this._onDisconnect = callback; }
  onStatus(callback) { this._onStatus = callback; }

  open() {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      if (!this._isOpen) this.send({ cmd: "connect" });
      return;
    }

    if (this._ws) {
      this.close();
    }

    this._ws = new WebSocket(this._url);

    this._ws.onopen = () => {
      this.send({ cmd: "connect" });
    };

    this._ws.onmessage = (event) => {
      // Esperamos JSON normalizado desde el bridge
      let msg;
      try {
        msg = JSON.parse(event.data);
      } catch (e) {
        console.warn("WS message is not JSON:", event.data);
        return;
      }

      // Convención mínima:
      // - {type:"status", state:"...", detail:"..."}
      // - {type:"microbit", x:..., y:..., btnA:..., btnB:...}
      if (msg.type === "status") {
        this._onStatus?.(msg);

        if (msg.state === "connected") {
          this._isOpen = true;
          this._onConnect?.();
        }

        if (msg.state === "disconnected" || msg.state === "error" || msg.state === "ready") {
          this._isOpen = false;
          this._onDisconnect?.();
          if (msg.state === "error") {
            this._ws?.close();
            this._ws = null;
          }
        }
        return;
      }

      if (msg.type === "microbit" || msg.type === "strudel" || msg.type === "osc") {
        this._onData?.(msg);
        return;
      }
    };

    this._ws.onerror = (err) => {
      console.warn("WS error:", err);
    };

    this._ws.onclose = () => {
      this._handleDisconnect();
    };
  }

  close() {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;

    try {
      this.send({ cmd: "disconnect" });
      this._isOpen = false;
    } catch (e) {
      console.warn("Failed to send disconnect command:", e);
    }
  }

  send(obj) {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;
    this._ws.send(JSON.stringify(obj));
  }

  _handleDisconnect() {
    this._isOpen = false;
    this._ws = null;
    this._onDisconnect?.();
  }
}

```
</details>

---

En bridgeClient.js, en la linea 70 aprox cambiar

``` js
if (msg.type === "microbit" || msg.type === "strudel") {
  this._onData?.(msg);
  return;
}
```
por

``` js
if (msg.type === "microbit" || msg.type === "strudel" || msg.type === "osc") {
  this._onData?.(msg);
  return;
}
```

</details>

---

<details>
<summary><b>sketch.js</b></summary>

``` js
const EVENTS = {
  CONNECT: "CONNECT",
  DISCONNECT: "DISCONNECT",
  DATA: "DATA",
};

class PainterTask extends FSMTask {
  constructor() {
    super();

    // Cola de eventos musicales de Strudel.
    // Aquí se guardan hasta que llegue su timestamp.
    this.scheduledEvents = [];

    // Animaciones que ya fueron activadas y están siendo dibujadas.
    this.activeAnimations = [];

    // Corrección opcional de latencia.
    this.latencyCorrection = 0;

    // Controles persistentes enviados desde Open Stage Control.
    // Estos valores no disparan animaciones por sí solos;
    // modifican cómo se dibujan las animaciones de Strudel.
    this.controls = {
      colors: {
        bd: [255, 0, 80],       // bombo
        sd: [0, 200, 255],      // caja
        cp: [100, 220, 255],    // clap
        hh: [255, 255, 0],      // hi-hat
        other: [200, 200, 200]  // sonidos no clasificados
      },

      scale: 1,              // tamaño general de las visuales
      showHatLayer: true     // activa/desactiva visuales de hi-hat
    };

    this.transitionTo(this.estado_esperando);
  }

  estado_esperando = (ev) => {
    if (ev.type === "ENTRY") {
      cursor();
      console.log("Waiting for bridge connection...");
    } else if (ev.type === EVENTS.CONNECT) {
      this.transitionTo(this.estado_corriendo);
    }
  };

  estado_corriendo = (ev) => {
    if (ev.type === "ENTRY") {
      noCursor();
      background(0);
      console.log("Bridge ready");

      // Se limpia la cola al entrar al estado activo.
      this.scheduledEvents = [];
      this.activeAnimations = [];
    }

    else if (ev.type === EVENTS.DISCONNECT) {
      this.transitionTo(this.estado_esperando);
    }

    else if (ev.type === EVENTS.DATA) {
      this.updateLogic(ev.payload);
    }

    else if (ev.type === "EXIT") {
      cursor();
    }
  };

  updateLogic(msg) {
    // STRUDEL:
    // Eventos musicales temporizados.
    // No se dibujan apenas llegan: se guardan en cola.
    if (msg.type === "strudel") {
      this.scheduledEvents.push({
        timestamp: msg.timestamp,
        sound: msg.payload.sound,
        family: msg.payload.family,
        delta: msg.payload.delta
      });

      // Mantiene la cola ordenada por tiempo.
      this.scheduledEvents.sort((a, b) => a.timestamp - b.timestamp);
      return;
    }

    // OSC:
    // Controles persistentes de Open Stage Control.
    // No entran a la cola temporal.
    // Actualizan variables del estado visual.
    if (msg.type === "osc") {
      this.updateControls(msg.payload);
      return;
    }
  }

  updateControls(payload) {
    const address = payload.address;
    const args = payload.args;

    // Cada RGB controla una familia sonora distinta.
    // Open Stage Control debe enviar tres valores: [R, G, B].

    if (address === "/rgb_bd") {
      this.controls.colors.bd = [
        Number(args[0] ?? 255),
        Number(args[1] ?? 0),
        Number(args[2] ?? 80)
      ];
    }

    if (address === "/rgb_sd") {
      this.controls.colors.sd = [
        Number(args[0] ?? 0),
        Number(args[1] ?? 200),
        Number(args[2] ?? 255)
      ];
    }

    if (address === "/rgb_cp") {
      this.controls.colors.cp = [
        Number(args[0] ?? 100),
        Number(args[1] ?? 220),
        Number(args[2] ?? 255)
      ];
    }

    if (address === "/rgb_hh") {
      this.controls.colors.hh = [
        Number(args[0] ?? 255),
        Number(args[1] ?? 255),
        Number(args[2] ?? 0)
      ];
    }

    if (address === "/rgb_other") {
      this.controls.colors.other = [
        Number(args[0] ?? 200),
        Number(args[1] ?? 200),
        Number(args[2] ?? 200)
      ];
    }

    // Slider para controlar escala general.
    // Recomendado en Open Stage Control: rango 0.5 a 2.0
    if (address === "/scale_1") {
      this.controls.scale = Number(args[0] ?? 1);
    }

    // Toggle para mostrar u ocultar hi-hats.
    // 1 = visible, 0 = oculto
    if (address === "/toggle_hats") {
      this.controls.showHatLayer = Boolean(args[0]);
    }
  }

  processScheduledEvents() {
    const now = Date.now() + this.latencyCorrection;

    // Activa eventos solo cuando el reloj local alcanza su timestamp.
    while (
      this.scheduledEvents.length > 0 &&
      now >= this.scheduledEvents[0].timestamp
    ) {
      const ev = this.scheduledEvents.shift();

      this.activeAnimations.push({
        startTime: ev.timestamp,
        duration: ev.delta * 1000,
        sound: ev.sound,
        family: ev.family,
        x: random(width * 0.2, width * 0.8),
        y: random(height * 0.2, height * 0.8),

        // El color se toma del estado persistente de OSC.
        color: getColorForFamily(ev.family, this.controls)
      });
    }
  }

  cleanupAnimations() {
    const now = Date.now() + this.latencyCorrection;

    // Elimina animaciones que ya terminaron.
    for (let i = this.activeAnimations.length - 1; i >= 0; i--) {
      const anim = this.activeAnimations[i];
      const elapsed = now - anim.startTime;
      const progress = elapsed / anim.duration;

      if (progress > 1) {
        this.activeAnimations.splice(i, 1);
      }
    }
  }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  rectMode(CENTER);
  noStroke();
  background(0);

  painter = new PainterTask();
  bridge = new BridgeClient();

  bridge.onConnect(() => {
    connectBtn.html("Disconnect");
    painter.postEvent({ type: EVENTS.CONNECT });
  });

  bridge.onDisconnect(() => {
    connectBtn.html("Connect");
    painter.postEvent({ type: EVENTS.DISCONNECT });
  });

  bridge.onStatus((s) => {
    console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
  });

  bridge.onData((data) => {
    painter.postEvent({
      type: EVENTS.DATA,
      payload: data
    });
  });

  connectBtn = createButton("Connect");
  connectBtn.position(10, 10);

  connectBtn.mousePressed(() => {
    if (bridge.isOpen) bridge.close();
    else bridge.open();
  });

  renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
  // Fondo con transparencia para dejar estela visual.
  background(0, 30);

  painter.update();

  if (painter.state === painter.estado_corriendo) {
    painter.processScheduledEvents();
    painter.cleanupAnimations();
  }

  renderer.get(painter.state)?.();
}

function drawRunning() {
  const now = Date.now() + painter.latencyCorrection;

  // drawRunning no interpreta mensajes de red.
  // Solo dibuja animaciones ya activadas.
  for (const anim of painter.activeAnimations) {
    const elapsed = now - anim.startTime;
    const progress = elapsed / anim.duration;

    if (progress >= 0 && progress <= 1) {
      drawAnimation(anim, progress);
    }
  }
}

function drawAnimation(anim, p) {
  push();

  // Toggle persistente controlado desde Open Stage Control.
  if (anim.family === "hh" && !painter.controls.showHatLayer) {
    pop();
    return;
  }

  // Selección visual según familia sonora ya normalizada.
  switch (anim.family) {
    case "bd":
      drawKick(anim, p);
      break;

    case "sd":
      drawSnare(anim, p);
      break;

    case "cp":
      drawClap(anim, p);
      break;

    case "hh":
      drawHat(anim, p);
      break;

    default:
      drawDefault(anim, p);
      break;
  }

  pop();
}

function drawKick(anim, p) {
  const s = painter.controls.scale;
  const d = lerp(100, 600, p) * s;
  const alpha = lerp(255, 0, p);

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  circle(width / 2, height / 2, d);
}

function drawSnare(anim, p) {
  const s = painter.controls.scale;
  const w = lerp(width, 0, p) * s;
  const alpha = lerp(255, 0, p);

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  rect(width / 2, height / 2, w, 50 * s);
}

function drawClap(anim, p) {
  const s = painter.controls.scale;
  const alpha = lerp(255, 0, p);
  const offset = lerp(120, 0, p) * s;

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);

  // Dos rectángulos laterales para diferenciar clap de snare.
  rect(width / 2 - offset, height / 2, 80 * s, 160 * s);
  rect(width / 2 + offset, height / 2, 80 * s, 160 * s);
}

function drawHat(anim, p) {
  const s = painter.controls.scale;
  const sz = lerp(40, 0, p) * s;
  const alpha = lerp(255, 0, p);

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  rect(anim.x, anim.y, sz, sz);
}

function drawDefault(anim, p) {
  const s = painter.controls.scale;
  const size = lerp(100, 0, p) * s;
  const angle = p * TWO_PI;
  const alpha = lerp(255, 0, p);

  translate(anim.x, anim.y);
  rotate(angle);

  stroke(anim.color[0], anim.color[1], anim.color[2], alpha);
  strokeWeight(2);
  noFill();
  rect(0, 0, size, size);
}

function getColorForFamily(family, controls) {
  return controls.colors[family] || controls.colors.other;
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```
</details>

---

Open Stage Control

Send 127.0.0.1:9000
Port 10010

[unidad7_controls.json](https://github.com/user-attachments/files/27210006/unidad7_controls.json)



---

para abrir el servidor

``` bash
node bridgeServer.js --device strudel-osc --wsPort 8081 --strudelPort 8080 --oscPort 9000 --verbose
```
`--device strudel-osc`
le dice al bridge que va a escuchar dos adapters al mismo tiempo:
- Strudel
- Open Stage Control
`--wsPort 8081`
puerto WebSocket hacia el frontend (bridgeClient.js)
`--strudelPort 8080`
puerto donde Strudel manda .osc()
`--oscPort 9000`
puerto UDP donde Open Stage Control manda OSC
`--verbose`
para que te imprima logs y puedas ver si llegan mensajes

ASEGURARSE DE TENER

``` bash
npm install osc
```

---



## Bitácora de reflexión
