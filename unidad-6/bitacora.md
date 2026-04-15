# Unidad 6

[ACTIVIDADES](https://juanferfranco.github.io/interactivos1-2026-10/units/unit6/)

[MICRO:BIT](https://python.microbit.org/v/3)


## Bitácora de proceso de aprendizaje

### Actividad 01



## Bitácora de aplicación 

### Actividad 02

Crear un adaptador para strudel en el repositorio de las ultimas unidades

<details>
<summary><b>StrudelAdapter.js</b></summary>

``` js
const BaseAdapter = require("./BaseAdapter");
const { WebSocketServer } = require("ws");

class StrudelAdapter extends BaseAdapter {
  constructor({ port = 8080, verbose = false } = {}) {
    super();
    this.port = port;
    this.verbose = verbose;
    this.wss = null;
  }

  async connect() {
    if (this.connected) return;

    this.wss = new WebSocketServer({ port: this.port });

    this.wss.on("connection", (ws) => {
      this.onConnected?.(`Strudel conectado en ws://127.0.0.1:${this.port}`);

      ws.on("message", (raw) => {
        const normalized = this._normalize(raw);
        if (!normalized) return;

        if (this.verbose) {
          console.log("[StrudelAdapter] normalized:", normalized);
        }

        this.onData?.(normalized);
      });

      ws.on("close", () => {
        this.onDisconnected?.("Strudel desconectado");
      });

      ws.on("error", (err) => {
        this.onError?.(`WS client error: ${err.message || err}`);
      });
    });

    this.connected = true;
    this.onConnected?.(`Esperando eventos de Strudel en ws://127.0.0.1:${this.port}`);
  }

  async disconnect() {
    if (!this.connected) return;

    if (this.wss) {
      this.wss.close();
      this.wss = null;
    }

    this.connected = false;
    this.onDisconnected?.("Strudel adapter detenido");
  }

  getConnectionDetail() {
    return `strudel ws port ${this.port}`;
  }

  _normalize(raw) {
    let msg;

    try {
      msg = JSON.parse(raw.toString("utf8"));
    } catch (e) {
      this.onError?.("Mensaje de Strudel no es JSON válido");
      return null;
    }

    if (!msg || !Array.isArray(msg.args) || typeof msg.timestamp !== "number") {
      this.onError?.("Mensaje de Strudel incompleto");
      return null;
    }

    const params = {};
    for (let i = 0; i < msg.args.length; i += 2) {
      const key = msg.args[i];
      const value = msg.args[i + 1];
      params[key] = value;
    }

    const sound = params.s || "unknown";
    const family = this._getFamily(sound);

    return {
      type: "strudel",
      timestamp: msg.timestamp,
      payload: {
        eventType: "noteEvent",
        sound: sound,
        family: family,
        delta: Number(params.delta ?? 0.25),
        cycle: Number(params.cycle ?? 0),
        cps: Number(params.cps ?? 0),
        bank: params.bank ?? null
      }
    };
  }

  _getFamily(sound) {
    if (sound.includes("bd")) return "bd";
    if (sound.includes("sd")) return "sd";
    if (sound.includes("cp")) return "cp";
    if (sound.includes("hh")) return "hh";
    if (sound.includes("oh")) return "hh";
    return "other";
  }
}

module.exports = StrudelAdapter;
```

</details>

---

Modificar `bridgeServer.js`para que se registre el nuevo adapter

pegar esto al inicio junto con los demas adapter

``` js
const StrudelAdapter = require("./adapters/StrudelAdapter");
```

y en la funcion `createAdapter()` agregar esto para que el se conecte el dispositivo y se use el adaptador correspondiente

``` js
  if (DEVICE === "strudel") {
    const port = parseInt(getArg("strudelPort", "8080"), 10);
    return new StrudelAdapter({ port, verbose: VERBOSE });
  }
```

añadir lo siguiente en `adapter.onData = (d) => {`, que está dentro de la funcion `main`, al principio

``` js
 if (d.type === "strudel") {
      broadcast(wss, d);
      return;
    }
```

lo que permite que, si el adapter es de Strudel, reenvía el mensaje ya normalizado; y si es de micro:bit siga con normalidad.

---

Modificar `bridgeClient.js` solo para permitir que pase `type: "strudel"`, buscar esta parte (deberia estar aprox en la linea 70)

``` js
      if (msg.type === "microbit") {
        // payload ya normalizado
        this._onData?.(msg);
        return;
      }
```
y cambiarla por esta:

``` js
      if (msg.type === "microbit" || msg.type === "strudel") {
        this._onData?.(msg);
        return;
      }
```
---
Modificar `sketch.js` 

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

    this.scheduledEvents = [];
    this.activeAnimations = [];
    this.latencyCorrection = 0;

    this.transitionTo(this.estado_esperando);
  }

  estado_esperando = (ev) => {
    if (ev.type === "ENTRY") {
      cursor();
      console.log("Waiting for Strudel connection...");
    } else if (ev.type === EVENTS.CONNECT) {
      this.transitionTo(this.estado_corriendo);
    }
  };

  estado_corriendo = (ev) => {
    if (ev.type === "ENTRY") {
      noCursor();
      background(0);
      console.log("Strudel ready");
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
    if (msg.type !== "strudel") return;

    this.scheduledEvents.push({
      timestamp: msg.timestamp,
      sound: msg.payload.sound,
      family: msg.payload.family,
      delta: msg.payload.delta
    });

    this.scheduledEvents.sort((a, b) => a.timestamp - b.timestamp);
  }

  processScheduledEvents() {
    const now = Date.now() + this.latencyCorrection;

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
        color: getColorForFamily(ev.family)
      });
    }
  }

  cleanupAnimations() {
    const now = Date.now() + this.latencyCorrection;

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

  switch (anim.family) {
    case "bd":
      drawKick(anim, p);
      break;

    case "sd":
    case "cp":
      drawSnare(anim, p);
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
  const d = lerp(100, 600, p);
  const alpha = lerp(255, 0, p);
  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  circle(width / 2, height / 2, d);
}

function drawSnare(anim, p) {
  const w = lerp(width, 0, p);
  const alpha = lerp(255, 0, p);
  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  rect(width / 2, height / 2, w, 50);
}

function drawHat(anim, p) {
  const sz = lerp(40, 0, p);
  const alpha = lerp(255, 0, p);
  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  rect(anim.x, anim.y, sz, sz);
}

function drawDefault(anim, p) {
  const size = lerp(100, 0, p);
  const angle = p * TWO_PI;
  const alpha = lerp(255, 0, p);

  translate(anim.x, anim.y);
  rotate(angle);

  stroke(anim.color[0], anim.color[1], anim.color[2], alpha);
  strokeWeight(2);
  noFill();
  rect(0, 0, size, size);
}

function getColorForFamily(family) {
  const colors = {
    bd: [255, 0, 80],
    sd: [0, 200, 255],
    cp: [100, 220, 255],
    hh: [255, 255, 0],
    other: [200, 200, 200]
  };

  return colors[family] || colors.other;
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```
</details>

---
para conectarse usar en el Git Bash:

``` gitbash
node bridgeServer.js --device strudel --wsPort 8081 --strudelPort 8080
```
`device strudel` → usa el nuevo adapter

`wsPort 8081` → frontend escucha ahí

`strudelPort 8080` → Strudel manda eventos ahí

en Strudel asegurarse que dentro de `$: stack ()`, esten el nombre del instrumento para que suene y tambien el nombre del instrumento + `.osc()` para emita los eventos para que el adapter del programa los reciba y genere las visuales
## Bitácora de reflexión

### Actividad 03
