# Unidad 7

[ACTIVIDADES](https://juanferfranco.github.io/interactivos1-2026-10/units/unit7/)

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Actividad 02

<details>
<summary><b>OSCAdapter.js</b></summary>

``` js
const osc = require("osc");

class OSCAdapter {
  constructor(bridge) {
    this.bridge = bridge;

    this.udpPort = new osc.UDPPort({
      localAddress: "0.0.0.0",
      localPort: 9000 // mismo puerto de OSC
    });

    this.udpPort.on("message", (msg) => {
      this.handleMessage(msg);
    });

    this.udpPort.open();
  }

  handleMessage(msg) {
    const normalized = {
      type: "osc",
      payload: {
        address: msg.address,
        args: msg.args.map(a => a.value ?? a)
      }
    };

    this.bridge.broadcast(normalized);
  }
}

module.exports = OSCAdapter;
```
</details>

en `bridServer.js`:

``` js
const OSCAdapter = require("./adapters/OSCAdapter");
```

justo despues de `const adapter = await createAdapter();` dentro de `main()` agregar:

``` js
const OscAdapter = new OSCAdapter();
```

despues de `adapter.onData = (d) => {`:

``` js
oscAdapter.onData = (msg) => {
  broadcast(wss, {
    type: "osc",
    payload: {
      address: msg.address,
      args: msg.args
    }
  });
};
```

En bridgeClient.js, en la linea 70 aprox cambiar

``` js
if (msg.type === "microbit" || msg.type === "strudel") {
  this._onData?.(msg);
  return;
}
```
por

``` js
if (msg.type === "microbit" || msg.type === "strudel" || msg.type === "osc") {
  this._onData?.(msg);
  return;
}
```


## Bitácora de reflexión
