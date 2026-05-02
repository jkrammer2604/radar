
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Calliope Park-Assistent (Bluetooth)</title>
    <style>
        * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
        body {
            margin: 0; background-color: #050505; color: #ffffff;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            display: flex; flex-direction: column; height: 100vh; overflow: hidden;
        }

        /* Video Container */
        #viewport {
            flex: 1; position: relative; display: flex;
            justify-content: center; align-items: center; background: #000;
        }
        video {
            width: 100%; height: 100%; object-fit: cover;
            border-bottom: 2px solid #333;
        }

        /* Overlay für Distanz */
        #overlay-dist {
            position: absolute; top: 15%; width: 100%;
            text-align: center; pointer-events: none; z-index: 10;
        }
        #dist-text {
            font-size: 7rem; font-weight: 900; margin: 0;
            text-shadow: 0 0 30px rgba(0,0,0,0.8);
            transition: color 0.3s ease; color: #4cd964;
        }
        #unit { font-size: 2rem; margin-left: 5px; }

        /* Status Bar */
        #status-bar {
            background: rgba(0,0,0,0.6); padding: 5px 15px;
            font-size: 0.8rem; color: #aaa; text-align: center;
        }

        /* Steuerung unten */
        #controls {
            background: #1c1c1e; padding: 20px;
            display: grid; grid-template-columns: 1fr 1fr; gap: 12px;
            padding-bottom: 40px;
        }
        button {
            padding: 18px; border-radius: 14px; border: none;
            font-size: 1.1rem; font-weight: 600; cursor: pointer;
            transition: opacity 0.2s;
        }
        button:active { opacity: 0.7; }
        
        #btn-connect {
            grid-column: span 2; background-color: #007aff; color: white;
        }
        #btn-cam { background-color: #3a3a3c; color: white; }
        #btn-reset { background-color: #3a3a3c; color: white; }

        .connected { background-color: #28a745 !important; }
    </style>
</head>
<body>

    <div id="viewport">
        <div id="overlay-dist">
            <p id="dist-text">---<span id="unit">cm</span></p>
        </div>
        <video id="webcam" autoplay playsinline muted></video>
    </div>

    <div id="status-bar">Warte auf Verbindung...</div>

    <div id="controls">
        <button id="btn-connect">CALLIOPE BLUETOOTH VERBINDEN</button>
        <button id="btn-cam">KAMERA AN</button>
        <button id="btn-reset" onclick="location.reload()">RELOAD</button>
    </div>

    <script>
        const distText = document.getElementById('dist-text');
        const statusBar = document.getElementById('status-bar');
        const btnConnect = document.getElementById('btn-connect');
        const video = document.getElementById('webcam');

        let bluetoothDevice, uartCharacteristic;
        let receiveBuffer = "";

        // Kamera starten
        document.getElementById('btn-cam').onclick = async () => {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ 
                    video: { facingMode: "environment" } 
                });
                video.srcObject = stream;
            } catch (err) {
                alert("Kamera-Fehler: " + err.message);
            }
        };

        // Bluetooth Logik
        btnConnect.onclick = async () => {
            try {
                statusBar.innerText = "Suche nach Calliope...";
                
                bluetoothDevice = await navigator.bluetooth.requestDevice({
                    filters: [
                        { namePrefix: 'BBC micro:bit' },
                        { namePrefix: 'Calliope' }
                    ],
                    optionalServices: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e']
                });

                statusBar.innerText = "Verbinde...";
                const server = await bluetoothDevice.gatt.connect();
                
                const service = await server.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e');
                
                // Charakteristik 0002 ist für den Empfang vom Calliope zum iPad
                uartCharacteristic = await service.getCharacteristic('6e400002-b5a3-f393-e0a9-e50e24dcca9e');

                await uartCharacteristic.startNotifications();
                
                btnConnect.innerText = "VERBUNDEN";
                btnConnect.classList.add('connected');
                statusBar.innerText = "Daten werden empfangen...";

                uartCharacteristic.addEventListener('characteristicvaluechanged', (event) => {
                    let value = event.target.value;
                    let str = new TextDecoder().decode(value);
                    receiveBuffer += str;

                    if (receiveBuffer.includes("!")) {
                        let parts = receiveBuffer.split("!");
                        let lastValid = parts[parts.length - 2]; 
                        
                        let match = lastValid.match(/D:(\d+)/);
                        if (match) {
                            let d = parseInt(match[1]);
                            updateDisplay(d);
                        }
                        receiveBuffer = parts[parts.length - 1];
                    }
                });

                bluetoothDevice.addEventListener('gattserverdisconnected', () => {
                    btnConnect.innerText = "GETRENNT - NEU VERBINDEN";
                    btnConnect.classList.remove('connected');
                    statusBar.innerText = "Verbindung verloren.";
                });

            } catch (err) {
                statusBar.innerText = "Fehler: " + err.message;
            }
        };

        function updateDisplay(d) {
            distText.innerText = d;
            
            if (d < 15) {
                distText.style.color = "#ff3b30";
            } else if (d < 40) {
                distText.style.color = "#ffcc00";
            } else {
                distText.style.color = "#4cd964";
            }
            
            const span = document.createElement('span');
            span.id = 'unit';
            span.innerText = 'cm';
            distText.appendChild(span);
        }
    </script>
</body>
</html>
