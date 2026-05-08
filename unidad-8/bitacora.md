# Unidad 8

[ACTIVIDADES](https://juanferfranco.github.io/interactivos1-2026-10/units/unit8/)

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Actividad 01

``` bash
node bridgeServer.js --device micro-strudel-osc --wsPort 8081 --strudelPort 8080 --oscPort 9000 --baud 115200 --verbose
```

En `bridgeServer.js` en `createAdapter()`

``` js
 if (DEVICE === "micro-strudel-osc") {
    const path = SERIAL_PATH ?? await findMicrobitPort();

    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }

    log.info(`micro:bit found at ${path}`);

    const strudelPort = parseInt(getArg("strudelPort", "8080"), 10);
    const oscPort = parseInt(getArg("oscPort", "9000"), 10);

    return [
      new MicrobitAdapter2({ path, baud: BAUD, verbose: VERBOSE }),
      new StrudelAdapter({ port: strudelPort, verbose: VERBOSE }),
      new OpenStageControlAdapter({ port: oscPort, verbose: VERBOSE })
    ];
  }
```

---

En `sketch.js`, agregar al constructor

``` js
this.microbit = {
  x: 0,
  y: 0,
  btnA: false,
  btnB: false,
  influenceX: 0,
  influenceY: 0,
  shake: 0
};
```

en `updateLogic(msg)` agregar este bloque

``` js
if (msg.type === "microbit") {

      this.microbit.x = msg.x;
      this.microbit.y = msg.y;

      this.microbit.btnA = !!msg.btnA;
      this.microbit.btnB = !!msg.btnB;

      // Inclinación convertida a desplazamiento visual
      this.microbit.influenceX = map(msg.x, -2048, 2048, -500, 500);
      this.microbit.influenceY = map(msg.y, -2048, 2048, -500, 500);

      // Escala SOLO mientras A esté presionado
      this.controls.scale =
        this.microbit.btnA ? 2.5 : 1;

      // Shake temporal con botón B
      if (this.microbit.btnB) {
        this.microbit.shake = 30;
      }

      return;
    }
```

En `drawRunning()`, antes de dibujar las animaciones, agregar

``` js
if (painter.microbit.shake > 0) {
  translate(
    random(-painter.microbit.shake, painter.microbit.shake),
    random(-painter.microbit.shake, painter.microbit.shake)
  );

  painter.microbit.shake *= 0.85;
}
```

Y en `processScheduledEvents()`, cambia las posiciones

``` js
x: random(width * 0.2, width * 0.8) + this.microbit.influenceX,
y: random(height * 0.2, height * 0.8) + this.microbit.influenceY,
```

---

``` py
function drawClap(anim, p) {
  const s = painter.controls.scale;
  const alpha = lerp(255, 0, p);
  const offset = lerp(120, 0, p) * s;

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);

  rect(anim.x - offset, anim.y, 80 * s, 160 * s);
  rect(anim.x + offset, anim.y, 80 * s, 160 * s);
}
```

``` py
function drawKick(anim, p) {
  const s = painter.controls.scale;
  const d = lerp(100, 600, p) * s;
  const alpha = lerp(255, 0, p);

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  circle(anim.x, anim.y, d);
}
```
---

<details>
 <summary>Codigo Sketch completo</summary>

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

    this.microbit = {
      x: 0,
      y: 0,
      btnA: false,
      btnB: false,
      influenceX: 0,
      influenceY: 0,
      shake: 0,
      scaleBoost: 1
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

    if (msg.type === "microbit") {

      this.microbit.x = msg.x;
      this.microbit.y = msg.y;

      this.microbit.btnA = !!msg.btnA;
      this.microbit.btnB = !!msg.btnB;

      // Inclinación convertida a desplazamiento visual
      this.microbit.influenceX = map(msg.x, -2048, 2048, -500, 500);
      this.microbit.influenceY = map(msg.y, -2048, 2048, -500, 500);

      // Escala SOLO mientras A esté presionado
      this.microbit.scaleBoost =
  this.microbit.btnA ? 2.2 : 1;

      // Shake temporal con botón B
      if (this.microbit.btnB) {
        this.microbit.shake = 30;
      }

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

        x: random(width * 0.2, width * 0.8) + this.microbit.influenceX,
        y: random(height * 0.2, height * 0.8) + this.microbit.influenceY,

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

  if (painter.microbit.shake > 0) {
    translate(
      random(-painter.microbit.shake, painter.microbit.shake),
      random(-painter.microbit.shake, painter.microbit.shake)
    );

    painter.microbit.shake *= 0.85;
  }

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
  const s = painter.controls.scale * painter.microbit.scaleBoost;
  const d = lerp(100, 600, p) * s;
  const alpha = lerp(255, 0, p);

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  circle(anim.x, anim.y, d);
}

function drawSnare(anim, p) {
  const s = painter.controls.scale * painter.microbit.scaleBoost;
  const w = lerp(width, 0, p) * s;
  const alpha = lerp(255, 0, p);

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  rect(width / 2, height / 2, w, 50 * s);
}

function drawClap(anim, p) {
  const s = painter.controls.scale * painter.microbit.scaleBoost;
  const alpha = lerp(255, 0, p);
  const offset = lerp(120, 0, p) * s;

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);

  rect(anim.x - offset, anim.y, 80 * s, 160 * s);
  rect(anim.x + offset, anim.y, 80 * s, 160 * s);
}

function drawHat(anim, p) {
  const s = painter.controls.scale * painter.microbit.scaleBoost;
  const sz = lerp(40, 0, p) * s;
  const alpha = lerp(255, 0, p);

  fill(anim.color[0], anim.color[1], anim.color[2], alpha);
  rect(anim.x, anim.y, sz, sz);
}

function drawDefault(anim, p) {
  const s = painter.controls.scale * painter.microbit.scaleBoost;
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

`micro:bit`

``` py
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

startTime = running_time()

while True:
    t = running_time()
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = 1 if button_a.is_pressed() else 0
    bState = 1 if button_b.is_pressed() else 0
    checksum = abs(xValue) + abs(yValue) + aState + bState
    
    data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(t, xValue, yValue, aState, bState, checksum)
    
    uart.write(data)
    
    sleep(100) # Envia datos a 10 Hz
```
## Bitácora de reflexión
