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


## Bitácora de reflexión


```py
from microbit import *
import utime
import music

# ---------------------------
# IMÁGENES DE LLENADO (OBLIGATORIO)
# ---------------------------
def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):  # 0..25
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

# ---------------------------
# TIMER (OBLIGATORIO)
# ---------------------------
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

# ---------------------------
# FSM: TEMPORIZADOR ESCAPE ROOM
# ---------------------------
class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []

        # Configuración inicial
        self.n = 20                 # inicia en 20 pixeles
        self.MIN_N = 15
        self.MAX_N = 25

        # Timer de 1 segundo para el conteo
        self.myTimer = self.createTimer("Timeout", 1000)

        self.estado_actual = None
        self.transicion_a(self.estado_config)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1) actualizar timers
        for t in self.timers:
            t.update()

        # 2) procesar eventos
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    # ---------------------------
    # ESTADO: CONFIG (desarmado)
    # ---------------------------
    def estado_config(self, ev):
        if ev == "ENTRY":
            self.myTimer.stop()  # por si veníamos contando
            display.show(FILL[self.n])

        elif ev == "A":
            if self.n < self.MAX_N:
                self.n += 1
                display.show(FILL[self.n])

        elif ev == "B":
            if self.n > self.MIN_N:
                self.n -= 1
                display.show(FILL[self.n])

        elif ev == "S":  # ARMED
            self.transicion_a(self.estado_counting)

    # ---------------------------
    # ESTADO: COUNTING (armado contando)
    # ---------------------------
    def estado_counting(self, ev):
        if ev == "ENTRY":
            # mostrar el valor actual y arrancar el tick de 1s
            display.show(FILL[self.n])
            self.myTimer.start(1000)

        elif ev == "Timeout":
            # apagar 1 pixel por segundo (bajando n)
            if self.n > 0:
                self.n -= 1
                display.show(FILL[self.n])

            # Si ya llegó a 0, explota; si no, seguir contando
            if self.n == 0:
                self.transicion_a(self.estado_exploded)
            else:
                self.myTimer.start(1000)

        # En Counting ignoramos A/B/S (según el enunciado, no dice que alteren)
        # Si te pidieran cancelar, aquí se manejaría.

    # ---------------------------
    # ESTADO: EXPLODED (fin)
    # ---------------------------
    def estado_exploded(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            # Sonido (no bloqueante si el runtime lo soporta)
            try:
                music.play(music.WAWAWAWAA, wait=False)
            except:
                # fallback si wait=False no está disponible
                music.play(music.WAWAWAWAA)

        elif ev == "A":
            # volver a modo configuración reiniciando a 20
            self.n = 20
            self.transicion_a(self.estado_config)

# ---------------------------
# LOOP PRINCIPAL (EVENTOS OBLIGATORIOS A/B/S)
# ---------------------------
task = Task()

while True:
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)
```
