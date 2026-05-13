# Unidad 2

[Actividades](https://juanferfranco.github.io/interactivos1-2026-10/units/unit2/)
## Bitácora de proceso de aprendizaje


### Actividad 01

- **_¿Cuáles son los estados en el programa?_**

para la primera version solamente hay 1 estado "estado_waitTimeout".
para la segunda version tiene 2 estados "estado_waitInON" y "estado_waitInOFF".
  
- **_¿Cuáles son los eventos en el programa?_**

"ENTRY", Se dispara cuando entras a un estado (en transicion_a()).
"EXIT", Se dispara cuando sales de un estado (en transicion_a()), aunque en estos estados no lo usan para hacer algo.
"Timeout", evento que genera el Timer cuando se cumple el tiempo(duration)
  
- **_¿Cuáles son las acciones en el programa?_**

    - Encender el pixel
    ``` py
      display.set_pixel(self.x, self.y, 9)
    ```

     - Apagar el pixel
    ```py
    display.set_pixel(self.x, self.y, 0)
    ```

     - Iniciar el temporizador
    ``` py
    self.myTimer.start()
    ```

     - (En la versión 2) Cambiar de estado
    ``` py
    self.transicion_a(self.estado_waitInOFF)
    self.transicion_a(self.estado_waitInON)
    ```

     - Actualizar timers (para que detecten si ya se cumplió el tiempo)
    ``` py
    t.update()
    ```

     - Poner eventos en cola
    ``` py
    post_event("Timeout")
    ```

     - Procesar eventos de la cola
    ``` py
    ev = self.event_queue.pop(0) y luego self.estado_actual(ev)
    ```

### Actividad 02

### Main.py

``` py
from microbit import *
import utime
from fsm import Timer,FSMTask, ENTRY

class Semaforo(FSMTask):
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        super().__init__()
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.add_timer("Timeout",self.timeInRed)
        self.transition_to(self.estado_waitInRed)
        

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)
    
    def estado_waitInRed(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y,0)
            self.transition_to(self.estado_waitInGreen)

    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)

        if ev == "Timeout":
            display.set_pixel(self.x,self.y+2,0)
            self.transition_to(self.estado_waitInYellow)

        if ev == "A":
            display.set_pixel(self.x,self.y+2,0)
            self.transition_to(self.estado_waitInYellow)

    def estado_waitInYellow(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y+1,0)
            self.transition_to(self.estado_waitInRed)
    
Semaforo1 = Semaforo(0,0,2000,1000,500)

while True:
    # Input processing
    if button_a.was_pressed(): Semaforo1.post_event("A")
    
    Semaforo1.update()
    utime.sleep_ms(20)
    
```

### sfm.py
``` py
import utime

ENTRY = "ENTRY"
EXIT  = "EXIT"

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active and utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
            self.active = False
            self.owner.post_event(self.event)


class FSMTask:
    def __init__(self):
        self._q = []
        self._timers = []
        self._state = None

    def post_event(self, ev):
        self._q.append(ev)

    def add_timer(self, event, duration):
        t = Timer(self, event, duration)
        self._timers.append(t)
        return t

    def transition_to(self, new_state):
        if self._state:
            self._state(EXIT)
        self._state = new_state
        self._state(ENTRY)

    def update(self):
        for t in self._timers:
            t.update()
        while self._q:
            ev = self._q.pop(0)
            if self._state:
                self._state(ev)
```

### Actividad 03




## Bitácora de aplicación 


### Actividad 04

<img width="280" height="578" alt="image" src="https://github.com/user-attachments/assets/4cee936e-0a53-4461-ab79-8ac1bd614b13" />



``` py
from microbit import *
import utime
import music

def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()
# Para mostrar usas display.show(FILL[n]) donde n será
# un valor de 0 a 25

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        #configuracion inical
        self.n = 20
        #inicia en 20 pixeles
        self.MIN_N = 15
        self.MAX_N = 25
        
        # Personalizas el nombre del evento y la duración, 1 segundo
        self.myTimer = self.createTimer("Timeout",1000)

        self.estado_actual = None
        self.transicion_a(self.estado_config)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers internos automáticamente
        for t in self.timers:
            t.update()

        # 2. Procesar la cola de eventos resultante
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")


    def estado_config(self, ev):
        if ev == "ENTRY":
            self.myTimer.stop() # por si veniamos contando
            display.show(FILL[self.n])
            
        elif ev == "A":
            if self.n < self.MAX_N:
                self.n += 1
                display.show(FILL[self.n])
                
        elif ev == "B":
            if self.n < self.MIN_N:
                self.n -= 1
                display.show(FILL[self.n])
                
        elif ev == "S": #Armado
            self.transicion_a(self.estado_counting)

    def estado_counting(self, ev):
        if ev == "ENTRY": #mostrar el valor actual y arrancar el tick de 1s
              display.show(FILL[self.n])
              self.myTimer.start(1000)
            
        elif ev == "Timeout": #apagar 1 pixel por segundo
             if self.n > 0:
                self.n -= 1
                display.show(FILL[self.n])            
            # Si ya llegó a 0, explota; si no, seguir contando 
             if self.n == 0:
                self.transicion_a(self.estado_exploded)
             else:
                self.myTimer.start(1000)

    def estado_exploded(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            try:
                music.play(music.WAWAWAWAA, wait=False)
            except:
                music.play(music.WAWAWAWAA)
           
        elif ev == "A":
                self.n = 20
                self.transicion_a(self.estado_config)
         
task = Task()

while True:
    # Aquí generas los eventos de los botones y el gesto
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)
```


## Bitácora de reflexión

### Actividad 05

**p5.js**
- **index.html**
```
<script src="https://unpkg.com/@gohai/p5.webserial@^1/libraries/p5.webserial.js"></script>
```
- **sketch.js**
``` js
let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
    let sendBtnA = createButton('A');
    let sendBtnB = createButton('B');
    let sendBtnS = createButton('S');
    sendBtnA.position(200, 300);
    sendBtnB.position(230, 300);
    sendBtnS.position(260, 300);
    sendBtnA.mousePressed(() => sendChar('A'));
    sendBtnB.mousePressed(() => sendChar('B'));
    sendBtnS.mousePressed(() => sendChar('S'));
    fill('white');
    ellipse(width / 2, height / 2, 100, 100);
}

function sendChar(ch) {
  if (!port.opened()) return;
  port.write(ch)
}

function draw() {
    
    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){
            fill('red');
        }
        else if(dataRx == 'B'){
            fill('yellow');
        }
        else if(dataRx == 'S'){
            fill('green');
        }
        background(220);
        ellipse(width / 2, height / 2, 100, 100);
        fill('black');
        text(dataRx, width / 2, height / 2);
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

function sendBtnClick() {
    port.write('h');
}
```
**micro:bit**
``` py
from microbit import *
import utime
import music

uart.init(baudrate=115200)

def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()
# Para mostrar usas display.show(FILL[n]) donde n será
# un valor de 0 a 25

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        #configuracion inical
        self.n = 20
        #inicia en 20 pixeles
        self.MIN_N = 15
        self.MAX_N = 25
        
        # Personalizas el nombre del evento y la duración, 1 segundo
        self.myTimer = self.createTimer("Timeout",1000)

        self.estado_actual = None
        self.transicion_a(self.estado_config)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers internos automáticamente
        for t in self.timers:
            t.update()

        # 2. Procesar la cola de eventos resultante
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")


    def estado_config(self, ev):
        if ev == "ENTRY":
            self.myTimer.stop() # por si veniamos contando
            display.show(FILL[self.n])
            
        elif ev == "A":
            if self.n < self.MAX_N:
                self.n += 1
                display.show(FILL[self.n])
                
        elif ev == "B":
            if self.n > self.MIN_N:
                self.n -= 1
                display.show(FILL[self.n])
                
        elif ev == "S": #Armado
            self.transicion_a(self.estado_counting)

    def estado_counting(self, ev):
        if ev == "ENTRY": #mostrar el valor actual y arrancar el tick de 1s
              display.show(FILL[self.n])
              self.myTimer.start(1000)
            
        elif ev == "Timeout": #apagar 1 pixel por segundo
             if self.n > 0:
                self.n -= 1
                display.show(FILL[self.n])            
            # Si ya llegó a 0, explota; si no, seguir contando 
             if self.n == 0:
                self.transicion_a(self.estado_exploded)
             else:
                self.myTimer.start(1000)

    def estado_exploded(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            try:
                music.play(music.RINGTONE, wait=False)
            except:
                music.play(music.RINGTONE)
           
        elif ev == "A":
                self.n = 20
                self.transicion_a(self.estado_config)
         
task = Task()

while True:
    # Aquí generas los eventos de los botones y el gesto
    if button_a.was_pressed():
        task.post_event("A")
        uart.write('A')
    if button_b.was_pressed():
        task.post_event("B")
        uart.write('B')
    if accelerometer.was_gesture("shake"):
        task.post_event("S")
        uart.write('S')

    if uart.any():
        ch = uart.read(1)
        if ch:
             ch = ch.decode('utf-8')
             if ch in ("A","B","S"):
                task.post_event(ch)
    
    task.update()
    utime.sleep_ms(20)
```




