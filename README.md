<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Pro-Park-Radar</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        body { margin: 0; background: #111; color: white; font-family: sans-serif; display: flex; flex-direction: column; height: 100vh; }
        
        #video-container { position: relative; flex: 1; background: #000; display: flex; justify-content: center; align-items: center; overflow: hidden; }
        video { max-width: 100%; max-height: 100%; border-bottom: 2px solid #333; }
        
        #controls { padding: 15px; background: #222; display: grid; grid-template-columns: 1fr 1fr; gap: 10px; border-top: 2px solid #444; }
        
        .full-width { grid-column: span 2; }
        
        button, select, input { 
            padding: 12px; border-radius: 8px; border: none; font-size: 1rem; 
            background: #444; color: white; width: 100%;
        }
        
        button.active { background: #28a745; }
        button.off { background: #c0392b; }
        
        #distance-info { 
            position: absolute; top: 20px; width: 100%; text-align: center; 
            font-size: 3rem; font-weight: bold; text-shadow: 2px 2px 10px black;
            pointer-events: none;
        }

        .label-text { font-size: 0.8rem; color: #aaa; margin-bottom: 4px; display: block; }
    </style>
</head>
<body>

    <div id="video-container">
        <video id="webcam" autoplay playsinline></video>
        <div id="distance-info">---</div>
    </div>

    <div id="controls">
        <div class="full-width">
            <span class="label-text">Kamera wählen:</span>
            <select id="cameraSelect"><option>Suche Kameras...</option></select>
        </div>

        <button id="toggleSound" class="active">🔊 Sound AN</button>
        <button id="toggleMsg" class="active">💬 Text AN</button>

        <div class="full-width">
            <span class="label-text">Empfindlichkeit (Reichweite):</span>
            <input type="range" id="sensRange" min="1" max="50" value="10">
        </div>

        <button id="startBtn" class="full-width" style="background: #28a745; font-weight: bold; font-size: 1.2rem;">SYSTEM STARTEN</button>
    </div>

    <script>
        const video = document.getElementById('webcam');
        const startBtn = document.getElementById('startBtn');
        const camSelect = document.getElementById('cameraSelect');
        const distInfo = document.getElementById('distance-info');
        const soundBtn = document.getElementById('toggleSound');
        const msgBtn = document.getElementById('toggleMsg');
        const sensRange = document.getElementById('sensRange');
        
        let model, audioCtx, nextBeepTime = 0;
        let soundEnabled = true;
        let msgEnabled = true;

        // Kameras auflisten
        async function getDevices() {
            const devices = await navigator.mediaDevices.enumerateDevices();
            const videoDevices = devices.filter(d => d.kind === 'videoinput');
            camSelect.innerHTML = videoDevices.map((d, i) => 
                `<option value="${d.deviceId}">${d.label || 'Kamera ' + (i+1)}</option>`
            ).join('');
        }
        getDevices();

        // Sound Funktion
        function playBeep(freq, dur) {
            if (!audioCtx || !soundEnabled) return;
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.frequency.value = freq;
            osc.type = 'square';
            gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + dur);
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            osc.start();
            osc.stop(audioCtx.currentTime + dur);
        }

        async function startCamera() {
            const constraints = { video: { deviceId: camSelect.value ? { exact: camSelect.value } : undefined } };
            const stream = await navigator.mediaDevices.getUserMedia(constraints);
            video.srcObject = stream;
        }

        async function detect() {
            if (!model) return;
            const predictions = await model.detect(video);
            let maxArea = 0;

            predictions.forEach(p => {
                const area = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                if (area > maxArea) maxArea = area;
            });

            const threshold = sensRange.value / 100; // Empfindlichkeit nutzen
            const now = audioCtx.currentTime;

            if (maxArea > threshold) {
                if (maxArea > 0.6) {
                    if (msgEnabled) { distInfo.innerText = "STOPP"; distInfo.style.color = "red"; }
                    if (now > nextBeepTime) { playBeep(1000, 0.2); nextBeepTime = now + 0.08; }
                } else {
                    if (msgEnabled) { distInfo.innerText = "WARNUNG"; distInfo.style.color = "yellow"; }
                    let pause = 0.7 - (maxArea * 1.1);
                    if (now > nextBeepTime) { playBeep(800, 0.1); nextBeepTime = now + Math.max(0.1, pause); }
                }
            } else {
                distInfo.innerText = msgEnabled ? "OK" : "";
                distInfo.style.color = "green";
            }
            requestAnimationFrame(detect);
        }

        // Event Listener
        soundBtn.onclick = () => {
            soundEnabled = !soundEnabled;
            soundBtn.innerText = soundEnabled ? "🔊 Sound AN" : "🔇 Sound AUS";
            soundBtn.className = soundEnabled ? "active" : "off";
        };

        msgBtn.onclick = () => {
            msgEnabled = !msgEnabled;
            msgBtn.innerText = msgEnabled ? "💬 Text AN" : "🚫 Text AUS";
            msgBtn.className = msgEnabled ? "active" : "off";
            if (!msgEnabled) distInfo.innerText = "";
        };

        startBtn.onclick = async () => {
            if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            startBtn.innerText = "LADE MODELL...";
            await startCamera();
            model = await cocoSsd.load();
            startBtn.style.display = "none";
            detect();
        };
    </script>
</body>
</html>
