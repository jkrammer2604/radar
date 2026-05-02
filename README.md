<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Radar Bluetooth</title>
    <style>
        body { margin: 0; background: #000; color: white; font-family: sans-serif; display: flex; flex-direction: column; height: 100vh; overflow: hidden; }
        #video-container { flex: 1; display: flex; justify-content: center; align-items: center; position: relative; background: #111; }
        video { max-width: 90%; max-height: 80%; border-radius: 15px; border: 2px solid #444; }
        #dist-info { position: absolute; top: 10%; width: 100%; text-align: center; font-size: 5rem; font-weight: 900; text-shadow: 0 0 20px #000; }
        #controls { background: #1a1a1a; padding: 20px; display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
        button { padding: 15px; border-radius: 12px; border: none; font-size: 1rem; color: white; background: #333; }
        .btn-bt { background: #007aff; grid-column: span 2; font-weight: bold; }
    </style>
</head>
<body>

    <div id="video-container">
        <div id="dist-info">--- cm</div>
        <video id="webcam" autoplay playsinline muted></video>
    </div>

    <div id="controls">
        <button id="camBtn">KAMERA AN</button>
        <button id="btBtn" class="btn-bt">CALLIOPE BLUETOOTH VERBINDEN</button>
    </div>

    <script>
        let bluetoothDevice, uartCharacteristic;

        // Kamera starten
        document.getElementById('camBtn').onclick = async () => {
            const stream = await navigator.mediaDevices.getUserMedia({ video: true });
            document.getElementById('webcam').srcObject = stream;
        };

        // Bluetooth Verbindung
        document.getElementById('btBtn').onclick = async () => {
            try {
                // Suche nach Calliope
                bluetoothDevice = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC micro:bit' }, { namePrefix: 'Calliope' }],
                    optionalServices: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e'] // UART Service
                });

                const server = await bluetoothDevice.gatt.connect();
                const service = await server.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e');
                uartCharacteristic = await service.getCharacteristic('6e400001-b5a3-f393-e0a9-e50e24dcca9e');

                await uartCharacteristic.startNotifications();
                document.getElementById('btBtn').innerText = "VERBUNDEN";
                document.getElementById('btBtn').style.background = "#28a745";

                let buffer = "";
                uartCharacteristic.addEventListener('characteristicvaluechanged', (event) => {
                    let decoder = new TextDecoder();
                    let str = decoder.decode(event.target.value);
                    buffer += str;

                    // Wir suchen nach dem "!" Trennzeichen
                    if (buffer.includes("!")) {
                        const match = buffer.match(/D:(\d+)/);
                        if (match) {
                            const d = match[1];
                            document.getElementById('dist-info').innerText = d + " cm";
                            document.getElementById('dist-info').style.color = d < 20 ? "red" : "green";
                        }
                        buffer = ""; // Buffer leeren
                    }
                });
            } catch (err) { alert("BT Fehler: " + err); }
        };
    </script>
</body>
</html>
