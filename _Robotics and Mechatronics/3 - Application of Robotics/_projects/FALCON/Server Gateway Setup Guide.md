---
isCurrent: false
needsUpdating: true
---
# Project F.A.L.C.O.N. — Teacher Server & LoRa Gateway Setup Guide

## 1. System Architecture Overview

Project F.A.L.C.O.N. uses a streamlined **Host PC $\rightarrow$ USB Serial $\rightarrow$ 915 MHz LoRa** architecture. Instead of requiring a complex MQTT/Docker stack or Wi-Fi FeatherWings, the Node.js server communicates directly over USB Serial with a single **LoRa Gateway Feather RP2040 RFM95**, which broadcasts JSON price and product updates to all student eInk shelf tags on the Australian **915.0 MHz ISM band**.

```
+-------------------------------------------------------------------+
|                     TEACHER HOST SERVER / PC                      |
|                                                                   |
|  [Admin Portal (/admin)] ──┐                                      |
|                            ├──> [Node.js Express Server :3000]    |
|  [POS Terminal (/terminal)]┘             │                        |
|                                          │ USB Serial (115200)    |
|                                          ▼                        |
|                            [LoRa Gateway Feather RP2040]          |
+------------------------------------------│------------------------+
                                           │ 915.0 MHz LoRa RF
                    ┌──────────────────────┼──────────────────────┐
                    ▼                      ▼                      ▼
            [Student Node #1]      [Student Node #2]      [Student Node #N]
             2.9" Tri-Color         2.9" Tri-Color         2.9" Tri-Color
              eInk Display           eInk Display           eInk Display
```

---

## 2. Prerequisites & Hardware Checklist

Before starting, ensure you have:
1. **Host Workstation:** Windows 10/11, macOS, or Ubuntu Linux PC with **Node.js (v18 LTS or v20 LTS)** installed.
2. **Gateway Microcontroller:** 1x **Adafruit Feather RP2040 with RFM95 LoRa Radio (915 MHz)** + a **915 MHz uFL/SMA antenna** (or 8.2 cm quarter-wave wire soldered to the `ANT` pad).
3. **Data-Sync USB-C Cable:** Must support data transfer (not charge-only).

---

## 3. Step 1: Configure the LoRa Gateway Microcontroller

The Gateway Feather acts as a transparent USB-to-LoRa bridge. It reads newline-terminated JSON strings from USB Serial (`sys.stdin`) and transmits them over 915.0 MHz LoRa.

### 3.1 Flash CircuitPython 10 & Libraries
1. Download the **CircuitPython 10.x `.uf2`** firmware for the **Adafruit Feather RP2040 RFM9x** from [circuitpython.org](https://circuitpython.org/board/adafruit_feather_rp2040_rfm9x/).
2. Plug the Gateway Feather into your PC via USB-C. Hold **BOOTSEL**, tap **RESET**, and release **BOOTSEL** when the `RPI-RP2` drive appears.
3. Drag and drop the `.uf2` file onto `RPI-RP2`. It will reboot as `CIRCUITPY`.
4. Download the **Adafruit CircuitPython 10.x Library Bundle** from [circuitpython.org/libraries](https://circuitpython.org/libraries) and copy the following into `CIRCUITPY/lib/`:
   * `adafruit_rfm9x.mpy`
   * `adafruit_bus_device/` (entire folder)

### 3.2 Gateway Firmware (`CIRCUITPY/code.py`)
Save the following code as `code.py` on the Gateway `CIRCUITPY` drive:

```python
import sys
import time
import board
import digitalio
import supervisor
import adafruit_rfm9x

# ==============================================================================
# F.A.L.C.O.N. USB-TO-LORA GATEWAY FIRMWARE (915 MHz AU ISM BAND)
# ==============================================================================
RADIO_FREQ_MHZ = 915.0
TX_POWER_DBM = 20

# Initialize SPI & RFM95 Radio
spi = board.SPI()
cs = digitalio.DigitalInOut(board.RFM_CS)
reset = digitalio.DigitalInOut(board.RFM_RST)

rfm95 = adafruit_rfm9x.RFM9x(spi, cs, reset, RADIO_FREQ_MHZ)
rfm95.tx_power = TX_POWER_DBM

print(f"[GATEWAY READY] Listening on USB Serial -> Broadcasting @ {RADIO_FREQ_MHZ} MHz")

serial_buffer = ""

while True:
    # Non-blocking read from USB CDC Serial
    while supervisor.runtime.serial_bytes_available:
        char = sys.stdin.read(1)
        if char == "\n":
            payload = serial_buffer.strip()
            serial_buffer = ""
            if payload:
                try:
                    rfm95.send(bytes(payload, "utf-8"))
                    print(f"[LORA TX OK] {payload}")
                except Exception as err:
                    print(f"[LORA TX ERR] {err}")
        else:
            serial_buffer += char
    time.sleep(0.01)
```

### 3.3 Identify the Gateway Serial Port
Note the COM/TTY port assigned to your Gateway Feather:
* **Windows:** Open **Device Manager $\rightarrow$ Ports (COM & LPT)** and note the port (e.g., `COM4`).
* **macOS:** Run `ls /dev/tty.usbmodem*` in Terminal (e.g., `/dev/tty.usbmodem14101`).
* **Linux:** Run `ls /dev/ttyACM*` in Terminal (e.g., `/dev/ttyACM0`).

---

## 4. Step 2: Set Up the Node.js Backend Server

### 4.1 Directory Structure
Create a project folder named `falcon-server` with the following structure:

```text
falcon-server/
├── package.json
├── server.js
└── public/
    ├── index.html       (Landing Hub)
    ├── admin.html       (ESL Management & LoRa Console)
    └── terminal.html    (Touch POS Checkout Terminal)
```

### 4.2 Initialize Project & Install Dependencies
Open a terminal inside `falcon-server/` and run:

```bash
npm init -y
npm install express serialport
```

### 4.3 Create `server.js`
Save the following backend code in `falcon-server/server.js`. Update `SERIAL_PORT_PATH` if you want to lock a specific port, or set the `FALCON_PORT` environment variable.

```javascript
const express = require('express');
const path = require('path');
const { SerialPort } = require('serialport');

const app = express();
const HTTP_PORT = process.env.PORT || 3000;
const SERIAL_PORT_PATH = process.env.FALCON_PORT || 'COM4'; // Change to /dev/ttyACM0 or /dev/tty.usbmodem* on Mac/Linux

app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// Initial Classroom Product Catalogue (1:1 Mapped to Student Node IDs)
let products = [
  { id: '1001', name: 'Adafruit Feather RP2040 RFM95', price: 32.95, stock: 25, category: 'Hardware' },
  { id: '1002', name: '2.9 in Tri-Color eInk FeatherWing', price: 34.50, stock: 20, category: 'Displays' },
  { id: '1003', name: 'RFM95W LoRa Breakout 915MHz', price: 19.95, stock: 30, category: 'Networking' },
  { id: '1004', name: '3.7V 1200mAh LiPo Battery Pack', price: 14.50, stock: 40, category: 'Power' },
  { id: '1005', name: 'USB-C Data Sync Cable 1m', price: 8.95, stock: 50, category: 'Accessories' }
];

// Initialize USB Serial Connection to LoRa Gateway
let gatewayPort = null;
try {
  gatewayPort = new SerialPort({ path: SERIAL_PORT_PATH, baudRate: 115200, autoOpen: false });
  gatewayPort.open((err) => {
    if (err) {
      console.warn(`[SERIAL WARN] Gateway not connected on ${SERIAL_PORT_PATH}: ${err.message}`);
      console.warn('[SERIAL WARN] Running in web-only simulation mode.');
    } else {
      console.log(`[SERIAL OK] Connected to LoRa Gateway on ${SERIAL_PORT_PATH}`);
    }
  });
} catch (e) {
  console.warn('[SERIAL WARN] SerialPort initialization skipped:', e.message);
}

// Helper to transmit JSON packet to Gateway Feather
function broadcastToLoRa(product) {
  const packet = JSON.stringify({
    id: String(product.id),
    name: product.name,
    price: Number(product.price)
  }) + '\n';

  if (gatewayPort && gatewayPort.isOpen) {
    gatewayPort.write(packet, (err) => {
      if (err) console.error('[LORA WRITE ERR]', err.message);
      else console.log(`[LORA DISPATCH] ${packet.trim()}`);
    });
  } else {
    console.log(`[SIMULATED LORA TX] ${packet.trim()}`);
  }
}

// Routes
app.get('/admin', (req, res) => res.sendFile(path.join(__dirname, 'public', 'admin.html')));
app.get('/terminal', (req, res) => res.sendFile(path.join(__dirname, 'public', 'terminal.html')));

app.get('/api/products', (req, res) => {
  res.json({ success: true, data: products });
});

app.post('/api/products/update', (req, res) => {
  const { id, name, price } = req.body;
  const product = products.find(p => String(p.id) === String(id));

  if (!product) {
    // Allow dynamic registration of new student Node IDs
    const newProduct = {
      id: String(id),
      name: String(name).trim().slice(0, 46),
      price: parseFloat(price),
      stock: 20,
      category: 'General'
    };
    products.push(newProduct);
    broadcastToLoRa(newProduct);
    return res.json({ success: true, product: newProduct });
  }

  if (name !== undefined) product.name = String(name).trim().slice(0, 46);
  if (price !== undefined && !isNaN(parseFloat(price))) product.price = parseFloat(price);

  broadcastToLoRa(product);
  res.json({ success: true, product });
});

app.post('/api/checkout', (req, res) => {
  const { cartItems } = req.body;
  if (Array.isArray(cartItems)) {
    cartItems.forEach(item => {
      const prod = products.find(p => String(p.id) === String(item.id));
      if (prod) prod.stock = Math.max(0, prod.stock - (item.count || item.qty || 1));
    });
  }
  res.json({ success: true, data: products });
});

app.listen(HTTP_PORT, '0.0.0.0', () => {
  console.log(`\n==================================================`);
  console.log(` Project F.A.L.C.O.N. Server Running!`);
  console.log(` Admin ESL Portal : http://localhost:${HTTP_PORT}/admin`);
  console.log(` POS Terminal     : http://localhost:${HTTP_PORT}/terminal`);
  console.log(`==================================================\n`);
});
```

### 4.4 Place Web Assets in `public/`
Copy `admin.html` and `terminal.html` into `falcon-server/public/`, and create a simple `falcon-server/public/index.html` hub linking to `/admin` and `/terminal`.

---

## 5. Step 3: Launching & Classroom Verification

1. **Close Mu / VS Code Serial Monitor:** Ensure no serial terminal is locking the Gateway Feather's COM port before starting Node.js.
2. **Start the Server:**
   ```bash
   # Windows PowerShell example:
   $env:FALCON_PORT="COM4"; node server.js

   # macOS / Linux example:
   FALCON_PORT=/dev/ttyACM0 node server.js
   ```
3. **Verify LoRa Broadcast:**
   * Open `http://<YOUR_SERVER_IP>:3000/admin` in your browser.
   * Edit the price of **Node `#1001`** and click **Broadcast**.
   * Confirm `[LORA DISPATCH] {"id":"1001",...}` appears in your server console and student tag `#1001` refreshes its eInk screen.
4. **Firewall Tip:** If student laptops or POS tablets cannot reach port `3000`, allow inbound TCP port `3000` on your host firewall.