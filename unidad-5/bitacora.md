# Unidad 5

[ACTIVIDADES](https://juanferfranco.github.io/interactivos1-2026-10/units/unit5/)

[MICRO:BIT](https://python.microbit.org/v/3)

## Bitácora de proceso de aprendizaje

### Actividad 01

- ¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?

  la ventaja seria que se transmitirian la misma información utilizando menos recursos y una mayor velocidad de lectura y de escritura.
  De desventaja es que no es legible para el humano, dificil de editar manualmente y de encontrar errores y depende de la arquitectura 


- Si xValue=500, yValue=524, aState=True, bState=False, ¿cómo se vería el paquete en hexadecimal? (Pista: convierte cada valor según su tipo y anota los bytes en orden.) Respuesta esperada: 01 F4 02 0C 01 00

  escribiendo en la calculadora en modo programador, en el apartado de decimal, si colocamos 500, en formato hexadecimal es 1F4, en datos de 2 bytes seria `01 F4`, lo mismo con yValue=524, su valor en hexadecimal es `02 0C`, el valor de true es 01 y false es 00

- ¿Por qué el protocolo ASCII de la unidad anterior no tenía este problema de sincronización? (Pista: piensa en qué rol cumplía el carácter \n.)

  porque se usaba \n como un delimitador claro para los mensajes, gracias a esto el programa podia saber desde donde a donde es la linea de informacion con la que tenia que manejar los datos
  
- ¿Por qué en binario no podemos usar \n como delimitador?

  porque en ASCII es un caracter especial mientras que en binario es otro numero mas, cualquier byte puede tener el mismo valor
  

## Bitácora de aplicación 

### Actividad 02

En `BridgeServer.js` eliminiar la mencion en `const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");` y dentro de la funcion `createAdapter()` sobre `if (DEVICE === "microbit-bin")`

se reutiliza el codigo de `MicrobitAdapter2.js` modificando la funcion `_onChunk(chunk)`, se puede eliminar la funcion de parseo porque ya no se trana de una linea de texto, y se cambia `this.buf = "";` por `this.buf = Buffer.alloc(0);` en todo el codigo

<DETAILS>
  <summary><b>MicrobitBinaryAdapter.js</b></summary>

``` js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class MicrobitBinaryAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = Buffer.alloc(0);
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
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
/*    
1. Acumula bytes entrantes
2. Busca el header (0xAA)
3. Verifica si hay un paquete completo
4. Valida checksum
5. Extrae datos y los envía al sistema
*/

//Acumular datos seriales, garantiza que el paquete esté completo antes de procesar, y maneja casos de corrupción de datos o pérdida de sincronización.
    this.buf = Buffer.concat([this.buf, chunk]);

    //Mientras haya datos en el buffer, intenta procesar varios paquetes.
    while (this.buf.length > 0){
        //Buscar el header del paquete (0xAA)
        const headerIndex = this.buf.indexOf(0xAA);

        //Si no se encuentra el header, descartar datos anteriores (posible corrupción) y esperar nuevos datos.
        if (headerIndex === -1){
            this.buf = Buffer.alloc(0);
            return;
        }

        //Si el header no está al inicio del buffer, descartar datos anteriores para realinear con el inicio del paquete.
        if (headerIndex > 0){
            this.buf = this.buf.subarray(headerIndex);
        }

        //Verificar si hay suficientes bytes para un paquete completo (8 bytes)
        if (this.buf.length < 8){
            return;
        }

        //Extraer el paquete completo
        const packet = this.buf.subarray(0, 8);

        //Validar checksum (suma de bytes 1 a 6 módulo 256 debe igualar byte 7(checksum))
        const receiveChk = packet[7];
        const calculatedChk = (packet[1] + packet[2] + packet[3] + packet[4] + packet[5] + packet[6]) % 256;

        //Si el checksum no coincide, descartar el byte de header actual y continuar buscando el siguiente paquete para evitar desincronización.
        if (calculatedChk !== receiveChk){
            console.warn("Trama corrupta");
            this.buf = this.buf.subarray(1);
            continue;
        }

        //Extraer datos de los paquetes (X, Y, btnA, btnB)
        const x = packet.readInt16BE(1);
        const y = packet.readInt16BE(3);
        const btnA = packet[5] === 1;
        const btnB = packet[6] === 1;

        this.onData?.({x, y, btnA, btnB});
        this.buf = this.buf.subarray(8);

    }
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = Buffer.alloc(0);
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

module.exports = MicrobitBinaryAdapter;


```

</DETAILS>

<details>
  <summary><b>Microbit</b></summary>

``` py

from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

HEADER = 0xAA

def int16_to_bytes(value):
    # convertir enteros con signos a 2 bytes big endian

    if value < 0:
        value = 65536 + value
        
    high = (value >> 8) & 0xFF
    low = value & 0xFF

    return high, low

while True:
    
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()

    if xValue < -2048:
        xValue = -2048
    if xValue > 2047:
        xValue = 2047

    if yValue < -2048:
        yValue = -2048
    if yValue > 2047:
        yValue = 2047
    
    aState = 1 if button_a.is_pressed() else 0
    bState = 1 if button_b.is_pressed() else 0

    #convertir X y Y a 2 bytes cada uno
    xHigh, xLow = int16_to_bytes(xValue)
    yHigh, yLow = int16_to_bytes(yValue)
    
    checksum = (xHigh + xLow + yHigh + yLow + aState + bState) % 256
    
    packet = bytes([HEADER,xHigh,xLow,yHigh,yLow,aState,bState,checksum])
    
    uart.write(packet)
    
    sleep(100) # Envia datos a 10 Hz
```
</details>

## Bitácora de reflexión

### Actividad 03
