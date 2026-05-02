<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>MotionKit v2 Steuerung</title>
    <style>
        body { margin: 0; background: #1a1a1a; color: white; font-family: sans-serif; text-align: center; user-select: none; -webkit-user-select: none; }
        #cam { width: 100%; height: 40vh; background: #000; object-fit: cover; }
        #info { padding: 15px; font-size: 1.5rem; background: #222; display: flex; justify-content: space-around; }
        .grid { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 15px; width: 300px; margin: 20px auto; }
        button { 
            padding: 25px; border-radius: 20px; border: none; font-size: 2rem; 
            background: #444; color: white; touch-action: none;
        }
        button:active { background: #007aff; }
        .conn { background: #007aff; grid-column: span 3; font-size: 1.2rem; padding: 15px; margin-bottom: 10px; }
        .dist { color: #4cd964; }
    </style>
</head>
<body>

    <video id="cam" autoplay playsinline muted></video>

    <div id="info">
        <div id="status">Bereit</div>
        <div class="dist"><span id="d">---</span> cm</div>
    </div>

    <div class="grid">
        <button id="connect" class="conn">CALLIOPE VERBINDEN</button>
        
        <div></div>
        <button onpointerdown="send('W\n')" onpointerup="send('Q\n')">▲</button>
        <div></div>
        
        <button onpointerdown="send('A\n')" onpointerup="send('Q\n')">◀</button>
        <button onpointerdown="send('S\n')" onpointerup="send('Q\n')">▼</button>
        <button onpointerdown="send('D\n')" onpointerup="send('Q\n')">▶</button>
    </div>

    <script>
        let rx; // Zum Senden an Calliope
        const dEl = document.getElementById('d');
        const sEl = document.getElementById('status');

        // Kamera aktivieren
        navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } })
            .then(s => document.getElementById('cam').srcObject = s);

        // Senden-Funktion mit "neuer Zeile" (\n)
        async function send(msg) {
            if (rx) {
                try {
                    await rx.writeValue(new TextEncoder().encode(msg));
                } catch (e) { console.log("Senden fehlgeschlagen"); }
            }
        }

        document.getElementById('connect').onclick = async () => {
            try {
                const device = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC' }, { namePrefix: 'Calliope' }],
                    optionalServices: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e']
                });
                const server = await device.gatt.connect();
                const service = await server.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e');
                
                // Charakteristiken für UART
                rx = await service.getCharacteristic('6e400003-b5a3-f393-e0a9-e50e24dcca9e');
                const tx = await service.getCharacteristic('6e400002-b5a3-f393-e0a9-e50e24dcca9e');

                await tx.startNotifications();
                tx.addEventListener('characteristicvaluechanged', (e) => {
                    let text = new TextDecoder().decode(e.target.value);
                    if (text.includes("D:")) {
                        let match = text.match(/D:(\d+)/);
                        if (match) dEl.innerText = match[1];
                    }
                });

                sEl.innerText = "Verbunden!";
                document.getElementById('connect').style.background = "#28a745";
                document.getElementById('connect').innerText = "VERBUNDEN";
            } catch (e) { alert("Verbindungsfehler: " + e); }
        };
    </script>
</body>
</html>
