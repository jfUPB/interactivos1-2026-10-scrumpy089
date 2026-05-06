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

  this.microbit.influenceX = map(msg.x, -1024, 1024, -150, 150);
  this.microbit.influenceY = map(msg.y, -1024, 1024, -150, 150);

  if (this.microbit.btnA) {
    this.controls.scale = 1.8;
  }

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

`micro:bit`

``` py
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    data = "{},{},{},{}\n".format(xValue, yValue, aState, bState)
    uart.write(data)
    sleep(100) # Envia datos a 10 Hz
```
## Bitácora de reflexión
