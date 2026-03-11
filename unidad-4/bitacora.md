# Unidad 4

[ACTIVIDADES](https://juanferfranco.github.io/interactivos1-2026-10/units/unit4/)

[MICRO:BIT](https://python.microbit.org/v/3)


## Bitácora de proceso de aprendizaje

### Actividad 1

Codigo de p5

``` js
let c;
let lineModuleSize = 0;
let angle = 0;
let angleSpeed = 1;
let lineModule = [];
let lineModuleIndex = 0;

let clickPosX = 0;
let clickPosY = 0;

function preload() {
  lineModule[1] = loadImage("data/02.svg");
  lineModule[2] = loadImage("data/03.svg");
  lineModule[3] = loadImage("data/04.svg");
  lineModule[4] = loadImage("data/05.svg");
}

function setup() {
  createCanvas(windowWidth, windowHeight);
  background(255);
  //cursor(CROSS);
  noCursor();
  strokeWeight(0.75);

  c = color(181, 157, 0);
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}

function draw() {
  if (mouseIsPressed && mouseButton == LEFT) {
    let x = mouseX;
    let y = mouseY;

    push();
    translate(x, y);
    rotate(radians(angle));
    stroke(c);
    line(0, 0, lineModuleSize, lineModuleSize);
    angle += angleSpeed;
    pop();
  }
}

function mousePressed() {
  // create a new random color and line length
  lineModuleSize = random(50, 160);

  // remember click position
}

function keyReleased() {
  // change color
  if (key == " ")
    c = color(random(255), random(255), random(255), random(80, 100));
}

```

codigo de micro

``` py
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    data = "{},{},{},{}\n".format(xValue, yValue, aState,bState)
    uart.write(data)
    sleep(100) # Envia datos a 10 Hz
```

- crear carpeta con mkdir
- copiar el repositorio en el sistema
- entrar a la carpeta en la terminal de gitbash con " cd + nombre de carpeta
- instalar modulos con node
- node bridgeServer.js para abrir el servidor

- en VS code instalar la extension de p5.vscode
- zona inferior derecha, Go Live
- conectarse a pagina al serer

- para conectarse con el microbit:
- node bridgeServer.js --device microbit
- conectarse al server en la pagina




mkdir = crear carpeta
clone = copiar
cd = change directory + donde se quiera entrar según la posición actual
ls = MOSTRAR CONTENIDO de la posición actual
clear = limpiar consola
npm = node package manager
Ctrl + c = apagar servidor 



## Bitácora de aplicación 

### Actividad 02

**MicrobitAdapter2.js**

``` js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error { }

// ...existing code...
function parseCsvLine(line) {
  const raw = String(line || "").trim();
  if (!raw.startsWith("$")) throw new ParseError("Invalid start token");

  const chkMarker = "|CHK:";
  const chkPos = raw.lastIndexOf(chkMarker);
  if (chkPos < 0) throw new ParseError("Missing CHK field");

  // payload is everything after '$' up to (but not including) the '|CHK:' part
  const payload = raw.slice(1, chkPos);
  const fields = payload.split("|").map(f => f.split(":"));

  const map = {};
  for (const pair of fields) {
    if (pair.length < 2) continue;
    const key = pair[0].trim();
    const value = pair.slice(1).join(":").trim(); // allow ':' in values if any
    map[key] = value;
  }

  if (!("T" in map && "X" in map && "Y" in map && "A" in map && "B" in map)) {
    throw new ParseError("Missing required fields");
  }

  const providedChk = raw.slice(chkPos + chkMarker.length).trim().toUpperCase();
  // compute checksum: sum of char codes of payload modulo 256, hex uppercase 2 digits
  let sum = 0;
  for (let i = 0; i < payload.length; i++) sum = (sum + payload.charCodeAt(i)) & 0xFF;
  const expectedChk = sum.toString(16).toUpperCase().padStart(2, "0");
  if (providedChk !== expectedChk) throw new ParseError("Checksum mismatch");

  const x = Number(map.X);
  const y = Number(map.Y);
  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");

  const parseBool = (v) => {
    const s = String(v).trim().toLowerCase();
    if (s === "true" || s === "1") return true;
    if (s === "false" || s === "0") return false;
    throw new ParseError("Invalid button data");
  };

  const btnA = parseBool(map.A);
  const btnB = parseBool(map.B);

  // optional timestamp if valid integer
  const t = Number(map.T);
  const time = Number.isFinite(t) ? (t | 0) : undefined;

  return { x: x | 0, y: y | 0, btnA, btnB, t: time };
}
// ...existing code...


class MicrobitAsciiAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = "";
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    this.buf += chunk.toString("utf8");

    let idx;
    while ((idx = this.buf.indexOf("\n")) >= 0) {
      const line = this.buf.slice(0, idx).trim();
      this.buf = this.buf.slice(idx + 1);

      if (!line) continue;

      try {
        const parsed = parseCsvLine(line);
        this.onData?.(parsed);
      } catch (e) {
        if (e instanceof ParseError) {
          if (this.verbose) console.log("Bad data:", e.message, "raw:", line);
        } else {
          this._fail(e);
        }
      }
    }

    if (this.buf.length > 4096) this.buf = "";
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitAsciiAdapter2;

```
<br>

en `bridgeServer.js` agregar esto
``` js
const MicrobitAdapter2 = require("./adapters/MicrobitAdapter2");
```
<br>

y añadir esto en createAdapter()
``` js
return new MicrobitAdapter2({ path, baud: BAUD, verbose: VERBOSE });
```

<details>
<summary><b>Sketch.js</b></summary>

``` js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            ready: false
        };

        this.circleResolution = 2;
        this.radius = 10;
        this.shouldFill = false;
        this.shouldDraw = false;

        this.transitionTo(this.estado_esperando);
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            background(255);
            strokeWeight(2);
            stroke(0, 25);
            noFill();

            console.log("Microbit ready to draw");

            this.rxData = {
                x: 0,
                y: 0,
                btnA: false,
                btnB: false,
                ready: false
            };

            this.circleResolution = 2;
            this.radius = 10;
            this.shouldFill = false;
            this.shouldDraw = false;
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys?.(ev.keyCode, ev.key);
        }

        else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease?.(ev.keyCode, ev.key);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        this.rxData.ready = true;
        this.rxData.x = data.x;
        this.rxData.y = data.y;
        this.rxData.btnA = data.btnA;
        this.rxData.btnB = data.btnB;

        // Y del acelerómetro -> resolución del polígono
        this.circleResolution = int(map(data.y, -2048, 2047, 2, 10));
        this.circleResolution = constrain(this.circleResolution, 2, 10);

        // X del acelerómetro -> radio
        this.radius = map(data.x, -2048, 2047, -360, 360);

        // Botón B -> fill
        this.shouldFill = data.btnB;

        // Botón A -> dibujar
        this.shouldDraw = data.btnA;
    }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(720, 720);
    noFill();
    background(255);
    strokeWeight(2);
    stroke(0, 25);

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
            payload: {
                x: data.x,
                y: data.y,
                btnA: data.btnA,
                btnB: data.btnB
            }
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
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {
    let task = painter;

    if (!task.rxData.ready) return;
    if (!task.shouldDraw) return;

    push();
    translate(width / 2, height / 2);

    let angle = TAU / task.circleResolution;

    if (task.shouldFill) {
        fill(34, 45, 122, 50);
    } else {
        noFill();
    }

    beginShape();
    for (let i = 0; i <= task.circleResolution; i++) {
        let x = cos(angle * i) * task.radius;
        let y = sin(angle * i) * task.radius;
        vertex(x, y);
    }
    endShape();

    pop();
}

function windowResized() {
    // si quieres mantener exactamente 720x720, puedes dejar esto vacío
}
```
</details>

``` c++
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error {}

function parseFrame(line) {
  const trimmed = line.trim();

  if (!trimmed.startsWith("$")) {
    throw new ParseError("Frame does not start with $");
  }

  const body = trimmed.slice(1);
  const parts = body.split("|");

  if (parts.length !== 6) {
    throw new ParseError(`Expected 6 fields, got ${parts.length}`);
  }

  const data = {};

  for (const part of parts) {
    const [key, value] = part.split(":");
    if (key === undefined || value === undefined) {
      throw new ParseError(`Malformed field: ${part}`);
    }
    data[key] = value;
  }

  if (
    data.T === undefined ||
    data.X === undefined ||
    data.Y === undefined ||
    data.A === undefined ||
    data.B === undefined ||
    data.CHK === undefined
  ) {
    throw new ParseError("Missing required fields");
  }

  const t = Number(data.T);
  const x = Number(data.X);
  const y = Number(data.Y);
  const a = Number(data.A);
  const b = Number(data.B);
  const chk = Number(data.CHK);

  if (![t, x, y, a, b, chk].every(Number.isFinite)) {
    throw new ParseError("Invalid numeric data");
  }

  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) {
    throw new ParseError("Out of expected range");
  }

  if (![0, 1].includes(a) || ![0, 1].includes(b)) {
    throw new ParseError("Invalid button data");
  }

  const calculatedChk = Math.abs(x) + Math.abs(y) + a + b;

  if (calculatedChk !== chk) {
    throw new ParseError(
      `Corrupt frame: CHK=${chk}, expected=${calculatedChk}`
    );
  }

  return {
    x: x | 0,
    y: y | 0,
    btnA: a === 1,
    btnB: b === 1,
  };
}

class MicrobitAdapter2 extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = "";
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) {
      throw new Error("serialPort is required for microbit device mode");
    }

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }

    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    this.buf += chunk.toString("utf8");

    let idx;
    while ((idx = this.buf.indexOf("\n")) >= 0) {
      const line = this.buf.slice(0, idx).trim();
      this.buf = this.buf.slice(idx + 1);

      if (!line) continue;

      try {
        const parsed = parseFrame(line);
        this.onData?.(parsed);
      } catch (e) {
        if (e instanceof ParseError) {
          if (String(e.message).startsWith("Corrupt frame")) {
            console.warn("[MicrobitAdapter2] Warning:", e.message, "raw:", line);
          } else if (this.verbose) {
            console.log("[MicrobitAdapter2] Bad data:", e.message, "raw:", line);
          }
        } else {
          this._fail(e);
        }
      }
    }

    if (this.buf.length > 4096) this.buf = "";
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitAdapter2;
```

## Bitácora de reflexión

### Actividad 03

