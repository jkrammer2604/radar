<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Park-Radar Calliope</title>
    <style>
        * { box-sizing: border-box; }
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; background: #000; color: #fff; font-family: sans-serif; display: flex; flex-direction: column; overflow: hidden; }
        
        #video-container { flex: 1; display: flex; justify-content: center; align-items: center; position: relative; background: #111; padding: 10px; }
        video { max-width: 100%; max-height: 100%; border-radius: 12px; border: 2px solid #444; }

        #distance-info { position: absolute; top: 40px; width: 100%; text-align: center; font-size: 5rem; font-weight: 900; text-shadow: 0 0 20px #000; z-index: 10; transition: color 0.2s; }

        #controls { background: #1a1a1a; padding: 15px; display: grid; grid-template-columns: 1fr 1fr; gap: 10px; border-top: 1px solid #333; z-index: 20; }
        button, select { padding: 15px; border-radius: 12px; border: none; background: #333; color: white; font-size: 1rem; cursor: pointer; }
        .active { background: #28a745 !important; }
    </style>
</head>
<body>

    <div id="video-container">
        <div id="distance-info">--- cm</div>
        <video id="webcam" autoplay playsinline muted></video>
    </div>

    <div id="controls">
        <select id="cameraSelect"></select>
        <button id="startCam">KAMERA AN</button>
        <button id="connectSerial" style="grid-column: span 2; background: #007aff;">CALLIOPE VERBINDEN</button>
    </div>

    <script>
        const video = document.getElementById('webcam');
        const distDisplay = document.getElementById('distance-info');
        const camSelect = document.getElementById('cameraSelect');

        // Kamera-Setup
        async function init() {
            const devices = await navigator.mediaDevices.enumerateDevices();
            const vids = devices.filter(d => d.kind === 'videoinput');
            camSelect.innerHTML = vids.map((d, i) => `<option value="${d.deviceId}">${d.label || 'Cam '+(i+1)}</option>`).join('');
        }
        init();

        document.getElementById('startCam').onclick = async () => {
            const stream = await navigator.mediaDevices.getUserMedia({ video: { deviceId: camSelect.value } });
            video.srcObject = stream;
            document.getElementById('startCam').classList.add('active');
        };

        // Calliope Serial-Verbindung
        document.getElementById('connectSerial').onclick = async () => {
            try {
                const port = await navigator.serial.requestPort();
                await port.open({ baudRate: 115200 });
                document.getElementById('connectSerial').innerText = "VERBUNDEN";
                document.getElementById('connectSerial').classList.add('active');

                const decoder = new TextDecoderStream();
                port.readable.pipeTo(decoder.writable);
                const reader = decoder.readable.getReader();

                while (true) {
                    const { value, done } = await reader.read();
                    if (done) break;
                    if (value) {
                        const match = value.match(/D:(\d+)/);
                        if (match) {
                            const d = parseInt(match[1]);
                            distDisplay.innerText = d + " cm";
                            
                            // Farbe ändern
                            if (d < 15) distDisplay.style.color = "#ff3b30";
                            else if (d < 40) distDisplay.style.color = "#ffcc00";
                            else distDisplay.style.color = "#4cd964";
                        }
                    }
                }
            } catch (e) { alert("Serial Fehler: " + e.message); }
        };
    </script>
</body>
</html>
