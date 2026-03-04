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



## Bitácora de reflexión

### Actividad 03
