# Unidad 1

[Actividades](https://juanferfranco.github.io/interactivos1-2026-10/units/unit1/)

## Bitácora de proceso de aprendizaje

### Actividad 01

**_¿Qué es un sistema físico interactivo?_**

Es una forma de generar nuevas experiencias combinando las nuevas tecnologias con el mundo fisico, permitiendo que lo digital se proyecte en la realidad o que el usuario pueda interactuar directemente con el programa

**_¿Cómo podrías aplicar lo que has visto en tu perfil profesional?_**

aaaaa

### Actividad 02

**_¿Qué es el diseño/arte generativo?_**

Es una forma de creación artistica en la que el artista crea un sistema que luego puede producir una o varias obras por si misma, el artista no crea directamente la obra sino el metodo que la genera

**_¿Cómo podrías aplicar lo que has visto en tu perfil profesional?_**

aaaaaa

### Actividad 03

desarrollo cruzado

### Actividad 04

**_¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()?_**

según el segundo paso: "Si usas is_pressed(), el programa podría enviar múltiples mensajes si el botón se mantiene presionado", se habia intentado usar _was_pressed()_ y como se menciona en el mismo paso, solamente genera un click cuando el boton es presionado, ocasionando que el cambio de color solamente sea momentaneo sin importar que se esté dejando presionado, en cambio, _is_pressed()_, puede generar varios clicks cuando se esta presionando el boton permitiendo que se mantenga el color deseado  

## Bitácora de aplicación 

### Actividad 05

**Micro:bit**🫦

```py
from microbit import *

uart.init(baudrate=115200)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(250)
    if button_b.is_pressed():
        uart.write('B')
        sleep(250)
    
```

**p5.js**

```js
let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(120,300);
    connectBtn.mousePressed(connectBtnClick);
    
    positionX = width / 2;
  
    fill('white');
    ellipse(positionX, height / 2, 100, 100)
    
}

function draw() {

    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){
            fill('red');
            positionX = positionX - 20;
         
        }
        else if(dataRx == 'B'){
            fill('yellow');
          positionX = positionX + 20;
        }
        
        background(220);
        ellipse(positionX, height / 2, 100, 100);
        fill('black');
        text("", positionX, height / 2);
    }


    if (!port.opened()) {
        connectBtn.html('Connect to micro:bit');
    }
    else {
        connectBtn.html('Disconnect');
    }
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}

```

En el editor del micro:bit, le transfiero el codigo al dispositivo el cual simplemente trata de mandar ciertas señales cuando se presionan cada botón

En p5.js reutilicé el codigo de la actividad 3 y descarté las partes no relevantes para el ejercicio. El codigo inicializa creando un canvas con fondo, un boton con el se conecte al micro:bit una variable con la que mas adelante se modificará la posicion en X y el propio circulo en medio del canvas.
la posicion del circulo esta escrito asi: (PositionX, height / 2), permitiendo la movilizacion del circulo

## Bitácora de reflexión







