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

en Strudel asegurarse que dentro de `$: stack ()`, esten el nombre del instrumento para que suene y tambien el nombre del instrumento + `.osc()` para emita las notas, no como audio sino como eventos/datos, para que el adapter del programa los reciba y genere las visuales

## Bitácora de reflexión

### Actividad 03

1 

<img width="790" height="526" alt="image" src="https://github.com/user-attachments/assets/255d0069-1cc2-4385-a4e8-9bd5ef1d7575" />

- El sistema comienza en Strudel, que genera eventos musicales en el navegador. Estos eventos no se toman como audio solamente, sino como datos temporizados. 

- El StrudelAdapter recibe el mensaje crudo, valida su estructura y lo transforma en un contrato estable. 

- Luego, el bridgeServer.js lo reenvía sin mezclar lógica visual. 

- En el frontend, bridgeClient.js recibe el mensaje y lo entrega a la FSM.

- La capa de estado actualiza una cola de eventos programados y, cuando el tiempo local alcanza el timestamp del evento, este se activa.

- Finalmente, drawRunning dibuja la respuesta visual usando únicamente el estado ya calculado.

2

En la unidad 4 el reto era leer un formato textual proveniente del micro:bit. 

En la unidad 5 el problema se volvió más complejo porque el mensaje pasó a ser binario y fue necesario resolver sincronización de bytes, framing y checksum. 

En la unidad 6 el cambio principal ya no está tanto en el parseo, sino en la naturaleza temporal del evento: ahora no basta con recibir y traducir el dato, también hay que decidir cuándo debe ocurrir la respuesta visual.

3

Aunque en esta unidad la fuente de datos ya no sea hardware físico, la arquitectura sigue siendo la misma porque se conserva la misma lógica de separación de responsabilidades. Sigue existiendo una fuente externa de datos, sigue siendo necesario traducir el formato de entrada a un contrato comprensible para el sistema, sigue habiendo una capa de transporte y sigue existiendo una capa de frontend que transforma datos en estado y luego en imagen.

Lo que cambió no fue la arquitectura, sino la naturaleza de la fuente. Antes el sistema recibía sensores y botones desde un dispositivo físico. Ahora recibe eventos musicales desde otra aplicación web. Sin embargo, el patrón arquitectónico sigue siendo equivalente: fuente externa → adapter → bridge → cliente → FSM/estado → render.

Por eso esta unidad no rompe la continuidad del curso, sino que demuestra que la arquitectura construida en las unidades anteriores era lo suficientemente flexible para integrar otra clase de emisor sin rehacer todo el sistema.

4

Para traducir los eventos musicales en visualidad, decidí mapear principalmente la familia del sonido y la duración del evento.

Mapeo usado
bd → círculo grande central
sd / cp → barra o flash horizontal
hh → figuras pequeñas y rápidas
oh → se agrupa visualmente con hi-hat o como variante ligera
delta → duración de la animación
timestamp → momento exacto en que debe activarse la respuesta visual

**Justificación del mapeo**

Este mapeo tiene sentido porque busca traducir propiedades rítmicas y perceptuales del sonido a propiedades visuales equivalentes.

El bombo suele sentirse como un golpe grave, fuerte y centrado, así que un círculo grande que expande desde el centro representa bien esa sensación de impacto. 
La caja y la palmada son sonidos más secos y cortantes, por eso una barra o flash ancho funciona mejor, ya que se perciben como un corte o golpe transversal. 
Los hi-hats son sonidos cortos, brillantes y rápidos, así que visualmente se representan mejor con elementos pequeños, repetitivos y de corta duración.

El uso de delta para controlar la duración visual también tiene sentido porque mantiene una relación directa entre el tiempo musical del evento y el tiempo de permanencia de la animación. De esta manera, la visualidad no es arbitraria, sino que responde a rasgos reales del evento musical.

5

<img width="1089" height="347" alt="image" src="https://github.com/user-attachments/assets/f074fe8d-a19f-48a5-87e5-7911d894dd78" />
