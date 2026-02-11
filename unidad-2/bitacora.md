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


### Actividad 04



## Bitácora de aplicación 



## Bitácora de reflexión


