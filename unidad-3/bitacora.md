# Unidad 3

[Actividades](https://juanferfranco.github.io/interactivos1-2026-10/units/unit3/)

[Micro:bit](https://python.microbit.org/v/3)

[p5.js](https://editor.p5js.org/)

## Bitácora de proceso de aprendizaje

### Actividad 01

<details>
<summary>Semaforo</summary>
    
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

## Bitácora de aplicación 



## Bitácora de reflexión



