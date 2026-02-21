# Unidad 3

[Actividades](https://juanferfranco.github.io/interactivos1-2026-10/units/unit3/)

[Micro:bit](https://python.microbit.org/v/3)

[p5.js](https://editor.p5js.org/)

## Bitácora de proceso de aprendizaje

### Actividad 01

<details>
<summary><b>Semaforo</b></summary>
    
 ``` py
  
      from microbit import *
    import utime
    
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
    
    
    class Semaforo:
        def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
            self.event_queue = []
            self.timers = []
            self.x = _x
            self.y = _y
            self.timeInRed = _timeInRed
            self.timeInGreen = _timeInGreen
            self.timeInYellow = _timeInYellow
            self.myTimer = self.createTimer("Timeout",self.timeInRed)
    
            self.estado_actual = None
            self.transicion_a(self.estado_waitInRed)
    
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
                self.transicion_a(self.estado_waitInGreen)
    
        def estado_waitInGreen(self, ev):
            if ev == "ENTRY":
                self.clear()
                display.set_pixel(self.x,self.y+2,9)
                self.myTimer.start(self.timeInGreen)
    
            if ev == "Timeout":
                display.set_pixel(self.x,self.y+2,0)
                self.transicion_a(self.estado_waitInYellow)
    
            if ev == "A":
                self.transicion_a(self.estado_waitInYellow)
            if ev == "B":
                self.transicion_a(self.estado_nocturno)
    
        def estado_nocturno(self, ev):
            if ev == "ENTRY":
                self.clear()
                display.set_pixel(self.x,self.y+1,9)
                self.myTimer.start(self.timeInYellow)
                
            if ev == "Timeout":
                if display.get_pixel(self.x,self.y+1) == 0:
                    display.set_pixel(self.x,self.y+1,9)
                else:
                    display.set_pixel(self.x,self.y+1,0)
                
                self.myTimer.start(self.timeInYellow)
    
            if ev == "A":
                self.transicion_a(self.estado_waitInYellow)
            
    
        def estado_waitInYellow(self, ev):
            if ev == "ENTRY":
                self.clear()
                display.set_pixel(self.x,self.y+1,9)
                self.myTimer.start(self.timeInYellow)
            if ev == "Timeout":
                display.set_pixel(self.x,self.y+1,0)
                self.transicion_a(self.estado_waitInRed)
                if ev == "B":
                    self.transicion_a(self.estado_nocturno)
    
    semaforo1 = Semaforo(0,0,2000,1000,500)
    
    while True:
    
        if button_a.was_pressed():
            semaforo1.post_event("A")
        if button_b.was_pressed():
            semaforo1.post_event("B")
            
        
        semaforo1.update()
        utime.sleep_ms(20)
```
  
</details>

### Actividad 02

<details>
    <summary><b>Codigo de temporizador</b><br></summary>

**main.py**

``` py
from microbit import *
from fsm import FSMTask, ENTRY, EXIT
from utils import FILL
import utime
import music
    
class Temporizador(FSMTask):
    def __init__(self):
        super().__init__()
        self.secuencia = []
        self.mypassword = ["A","B","A"]
        self.counter = 20
        self.myTimer = self.add_timer("Timeout",1000)
        self.estado_actual = None
        self.transition_to(self.estado_config)
    
    
    def estado_config(self, ev):
        if ev == ENTRY:
            self.counter = 20
            display.show(FILL[self.counter])
            self.myTimer.start()
        if ev == "A":
            if self.counter > 15:
                self.counter -= 1
            display.show(FILL[self.counter])
        if ev == "B":
            if self.counter < 25:
                self.counter += 1
            display.show(FILL[self.counter])
        if ev == "S":
            self.transition_to(self.estado_armed)
    
    def estado_armed(self, ev):
        if ev == ENTRY:
            self.myTimer.start()
            
        if ev == "Timeout":
            if self.counter > 0:
                self.counter -= 1
                display.show(FILL[self.counter])
            if self.counter == 0:
                 self.transition_to(self.estado_timeout)
            else:
                self.myTimer.start()
    
        if ev == "S":
            if self.myTimer.active == False:
                self.myTimer.start()
            else:
                self.myTimer.stop()     

        if ev == "A" or ev == "B":
            self.secuencia.append(ev)
            if len(self.secuencia) == 3:
                if self.secuencia == self.mypassword:
                    self.transition_to(self.estado_config)
                    self.secuencia.clear()
                else:
                    self.secuencia.clear()
                
        
    def estado_timeout(self, ev):
        if ev == ENTRY:
             display.show(Image.SKULL)
             music.play(music.FUNERAL)
        if ev == "A":
            music.stop()
            self.transition_to(self.estado_config)
    
temporizador = Temporizador()
    
while True:
    
    if button_a.was_pressed():
        temporizador.post_event("A")
    if button_b.was_pressed():
        temporizador.post_event("B")
    if accelerometer.was_gesture("shake"):
        temporizador.post_event("S")
            
    
    temporizador.update()
    utime.sleep_ms(20)      
```
<br>

**fsm.py**

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
<br>

**utils.py**

``` py
from microbit import Image

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
```
    
</details>

## Bitácora de aplicación 

### Acticidad 04

<details>
    <summary><b>Micro:bit</b></summary><br>
   
**main.py:**
``` py
from microbit import *
from fsm import FSMTask, ENTRY, EXIT
from utils import FILL
import utime
import music
    
class Temporizador(FSMTask):
    def __init__(self):
        super().__init__()
        self.secuencia = []
        self.mypassword = ["A","B","A"]
        self.counter = 20
        self.myTimer = self.add_timer("Timeout",1000)
        self.estado_actual = None
        self.transition_to(self.estado_config)
    
    
    def estado_config(self, ev):
        if ev == ENTRY:
            self.counter = 20
            display.show(FILL[self.counter])
            self.myTimer.start()
        if ev == "A":
            if self.counter > 15:
                self.counter -= 1
            display.show(FILL[self.counter])
        if ev == "B":
            if self.counter < 25:
                self.counter += 1
            display.show(FILL[self.counter])
        if ev == "S":
            self.transition_to(self.estado_armed)
    
    def estado_armed(self, ev):
        if ev == ENTRY:
            self.myTimer.start()
            
        if ev == "Timeout":
            if self.counter > 0:
                self.counter -= 1
                display.show(FILL[self.counter])
            if self.counter == 0:
                 self.transition_to(self.estado_timeout)
            else:
                self.myTimer.start()
    
        if ev == "S":
            if self.myTimer.active == False:
                self.myTimer.start()
            else:
                self.myTimer.stop()     

        if ev == "A" or ev == "B":
            self.secuencia.append(ev)
            if len(self.secuencia) == 3:
                if self.secuencia == self.mypassword:
                    self.transition_to(self.estado_config)
                    self.secuencia.clear()
                else:
                    self.secuencia.clear()
                
        
    def estado_timeout(self, ev):
        if ev == ENTRY:
             display.show(Image.SKULL)
             music.play(music.FUNERAL)
        if ev == "A":
            music.stop()
            self.transition_to(self.estado_config)
    
temporizador = Temporizador()
    
while True:
    
    if button_a.was_pressed():
        temporizador.post_event("A")
        uart.write('A')
    if button_b.was_pressed():
        temporizador.post_event("B")
        uart.write('B')
    if accelerometer.was_gesture("shake"):
        temporizador.post_event("S")
        uart.write('S')
            
    
    temporizador.update()
    utime.sleep_ms(20)
```
<br>

**fsm.py:**
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
<br>

**utils.py:**
``` py
from microbit import Image

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
```

</details>

<details>
    <summary><b>p5.js</b></summary><br>

**sketch.js:**

``` js
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",
  INC: "B",
  START: "S",
  TICK: "Timeout",
};

const UI = {
  dialSize: 250,
  ringWeight: 20,
  bigText: 100,
  configText: 120,
  helpText: 18,
};

class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;
    this.secuencia = [];
    this.mypassword = ["A", "B", "A"];

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);
    this.transitionTo(this.estado_config);
  }

  get currentState() {
    return this.state;
  }

  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
    } else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };

  compararPassword(secuencia, mypassword) {
    for (let i = 0; i < secuencia.length; i++) {
      if (secuencia[i] !== mypassword[i]) return false;
    }
    return true;
  }

  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.start();
    } else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      }
    } else if (ev === EXIT) {
      this.myTimer.stop();
    }

    if (ev === "S") {
      if (this.myTimer.active == false) {
        // console.log("hola");
        this.myTimer.start();
      } else {
        // console.log("hola2");
        this.myTimer.stop();
      }
    }

    if (ev === "A" || ev === "B") {
      this.secuencia.push(ev);
      if (this.secuencia.length === 3) {
        // console.log("secuencia longitud")
        if (this.compararPassword(this.secuencia, this.mypassword)) {
          this.transitionTo(this.estado_config);
          this.secuencia = [];
        } else {
          this.secuencia = [];
        }
      }
    }
  };

  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  };
}

let temporizador;
const renderer = new Map();

function setup() {
  port = createSerial();
  connectBtn = createButton("Connect to micro:bit");
  connectBtn.position(windowWidth / 2, 600);
  connectBtn.mousePressed(connectBtnClick);
  createCanvas(windowWidth, windowHeight);
  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );
  textAlign(CENTER, CENTER);

  renderer.set(temporizador.estado_config, () =>
    drawConfig(temporizador.configValue)
  );
  renderer.set(temporizador.estado_armed, () =>
    drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds)
  );
  renderer.set(temporizador.estado_timeout, () => drawTimeout());
}

function draw() {
  temporizador.update();
  renderer.get(temporizador.currentState)?.();

  if (port.availableBytes() > 0) {
    let dataRx = port.read(1);
    if (dataRx == "A") {
      console.log("A");
      temporizador.postEvent("A");
    } else if (dataRx == "B") {
      console.log("B");
      temporizador.postEvent("B");
    } else if (dataRx == "S") {
      console.log("S");
      temporizador.postEvent("S");
    }
  }
  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  } else {
    connectBtn.html("Disconnect");
  }
}

function drawConfig(val) {
  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);
  textSize(18);
  fill(200);
  text("A(-) B(+) S(start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total) {
  background(20, 20, 20);
  let pulse = sin(frameCount * 0.1) * 10;

  noFill();
  strokeWeight(20);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, 250);

  stroke(255, 150, 0);
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);

  fill(255);
  noStroke();
  textSize(100 + pulse);
  text(val, width / 2, height / 2);
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}

function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent("A");
  if (key === "b" || key === "B") temporizador.postEvent("B");
  if (key === "s" || key === "S") temporizador.postEvent("S");
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}

function connectBtnClick() {
  if (!port.opened()) {
    port.open("MicroPython", 115200);
  } else {
    port.close();
  }
}
```
<br>

**index.html:**

``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>Sketch</title>

    <link rel="stylesheet" type="text/css" href="style.css">

    <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
    <script src="https://unpkg.com/@gohai/p5.webserial@^1/libraries/p5.webserial.js"></script>
  </head>

  <body>
    <script src="fsm.js"></script>
    <script src="sketch.js"></script>
  </body>
</html>
```
<br>

**fsm.js:**

``` js
const ENTRY = "ENTRY";
const EXIT = "EXIT";

class Timer {
  constructor(owner, eventToPost, duration) {
    this.owner = owner;
    this.event = eventToPost;
    this.duration = duration;
    this.startTime = 0;
    this.active = false;
  }

  start(newDuration = null) {
    if (newDuration !== null) this.duration = newDuration;
    this.startTime = millis();
    this.active = true;
  }

  stop() {
    this.active = false;
  }

  update() {
    if (this.active && millis() - this.startTime >= this.duration) {
      this.active = false;
      this.owner.postEvent(this.event);
    }
  }
}

class FSMTask {
  constructor() {
    this.queue = [];
    this.timers = [];
    this.state = null;
  }

  postEvent(ev) {
    this.queue.push(ev);
  }

  addTimer(event, duration) {
    let t = new Timer(this, event, duration);
    this.timers.push(t);
    return t;
  }

  transitionTo(newState) {
    if (this.state) this.state(EXIT);
    this.state = newState;
    this.state(ENTRY);
  }

  update() {
    for (let t of this.timers) {
      t.update();
    }
    while (this.queue.length > 0) {
      let ev = this.queue.shift();
      if (this.state) this.state(ev);
    }
  }
}
```
</details>

## Bitácora de reflexión






