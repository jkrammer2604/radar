<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Radar BT Fix</title>
    <style>
        body { margin: 0; background: #000; color: white; font-family: sans-serif; text-align: center; }
        #vid { width: 100%; max-height: 70vh; object-fit: contain; background: #111; }
        #dist { font-size: 5rem; font-weight: bold; margin: 20px 0; color: #4cd964; }
        #btn { padding: 20px; width: 80%; font-size: 1.2rem; border-radius: 15px; border: none; background: #007aff; color: white; font-weight: bold; }
        .status { font-size: 0.9rem; color: #888; margin-top: 10px; }
    </style>
</head>
<body>

    <video id="vid" autoplay playsinline muted></video>
    <div id="dist">--- cm</div>
    
    <button id="btn">VERBINDEN</button>
    <div id="status" class="status">Kamera wird gestartet...</div>

    <script>
        const distEl = document.getElementById('dist');
        const statusEl = document.getElementById('status');
        const video = document.getElementById('vid');

        // Kamera automatisch starten
        navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } })
            .then(s => video.srcObject = s)
            .catch(e => statusEl.innerText = "Kamera-Fehler");

        let uartChar;
        let receiveBuffer = "";

        document.getElementById('btn').onclick = async () => {
            try {
                statusEl.innerText = "Suche Calliope...";
                const device = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC micro:bit' }, { namePrefix: 'Calliope' }],
                    optionalServices: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e']
                });

                statusEl.innerText = "Verbinde...";
                const server = await device.gatt.connect();
                const service = await server.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e');
                uartChar = await service.getCharacteristic('6e400002-b5a3-f393-e0a9-e50e24dcca9e'); // TX Characteristic

                await uartChar.startNotifications();
                statusEl.innerText = "Verbunden!";
                document.getElementById('btn').style.background = "#28a745";

                uartChar.addEventListener('characteristicvaluechanged', (event) => {
                    let data = new TextDecoder().decode(event.target.value);
                    receiveBuffer += data;

                    if (receiveBuffer.includes("!")) {
                        let parts = receiveBuffer.split("!");
                        let lastMsg = parts[parts.length - 2]; 
                        let match = lastMsg.match(/D:(\d+)/);
                        if (match) {
                            let d = parseInt(match[1]);
                            distEl.innerText = d + " cm";
                            distEl.style.color = d < 15 ? "red" : (d < 35 ? "yellow" : "#4cd964");
                        }
                        receiveBuffer = parts[parts.length - 1];
                    }
                });
            } catch (err) {
                statusEl.innerText = "Fehler: " + err.message;
                console.error(err);
            }
        };
    </script>
</body>
</html>
