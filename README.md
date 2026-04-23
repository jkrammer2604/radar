<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Radar Ultra-Smooth</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        body { margin: 0; background: #000; color: white; font-family: sans-serif; display: flex; flex-direction: column; height: 100vh; }
        #video-container { position: relative; flex: 1; background: #000; display: flex; justify-content: center; align-items: center; overflow: hidden; }
        video { max-width: 100%; max-height: 100%; }
        #controls { padding: 15px; background: #1a1a1a; display: grid; grid-template-columns: 1fr 1fr; gap: 12px; border-top: 1px solid #333; }
        .full { grid-column: span 2; }
        button, select, input { padding: 15px; border-radius: 10px; border: none; font-size: 1rem; background: #333; color: white; }
        button.active { background: #2ecc71; }
        button.off { background: #e74c3c; }
        #distance-info { position: absolute; top: 10%; width: 100%; text-align: center; font-size: 4rem; font-weight: bold; text-shadow: 0 0 20px rgba(0,0,0,0.8); pointer-events: none; }
        .label { font-size: 0.85rem; color: #888; margin-bottom: 5px; display: block; }
    </style>
</head>
<body>

    <div id="video-container">
        <video id="webcam" autoplay playsinline></video>
        <div id="distance-info">---</div>
    </div>

    <div id="controls">
        <div class="full">
            <span class="label">Kamera:</span>
            <select id="cameraSelect"></select>
        </div>

        <button id="toggleSound" class="active">🔊 Ton</button>
        <button id="toggleMsg" class="active">💬 Text</button>

        <div class="full">
            <span class="label">Filter (Reaktion erst ab % Objektgröße im Bild):</span>
            <input type="range" id="sensRange" min="5" max="80" value="20">
            <div id="sensVal" style="text-align:right; font-size: 0.8rem;">20%</div>
        </div>

        <button id="startBtn" class="full" style="background: #2ecc71; font-weight: bold;">SYSTEM STARTEN</button>
    </div>

    <script>
        const video = document.getElementById('webcam');
        const startBtn = document.getElementById('startBtn');
        const camSelect = document.getElementById('cameraSelect');
        const distInfo = document.getElementById('distance-info');
        const soundBtn = document.getElementById('toggleSound');
        const msgBtn = document.getElementById('toggleMsg');
        const sensRange = document.getElementById('sensRange');
        const sensVal = document.getElementById('sensVal');
        
        let model, audioCtx, nextBeepTime = 0;
        let soundEnabled = true;
        let msgEnabled = true;
        let areaHistory = [0,0,0,0,0]; // Buffer für Smoothing

        async function getDevices() {
            const devices = await navigator.mediaDevices.enumerateDevices();
            const videoDevices = devices.filter(d => d.kind === 'videoinput');
            camSelect.innerHTML = videoDevices.map((d, i) => 
                `<option value="${d.deviceId}">${d.label || 'Cam ' + (i+1)}</option>`
            ).join('');
        }
        getDevices();

        function playBeep(freq, dur) {
            if (!audioCtx || !soundEnabled || audioCtx.state !== 'running') return;
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.frequency.value = freq;
            osc.type = 'sine';
            gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + dur);
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            osc.start();
            osc.stop(audioCtx.currentTime + dur);
        }

        async function detect() {
            if (!model) return;
            const predictions = await model.detect(video);
            
            // Nur Objekte finden, die eine gewisse Grund-Wahrscheinlichkeit haben (>60%)
            let currentFrameMax = 0;
            predictions.forEach(p => {
                if(p.score > 0.6) {
                    const area = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                    if (area > currentFrameMax) currentFrameMax = area;
                }
            });

            // Smoothing: Durchschnitt der letzten 5 Messungen berechnen
            areaHistory.shift();
            areaHistory.push(currentFrameMax);
            const smoothedArea = areaHistory.reduce((a, b) => a + b) / areaHistory.length;

            const threshold = sensRange.value / 100;
            const now = audioCtx.currentTime;

            if (smoothedArea > threshold) {
                // Skalierung für Warnstufen
                if (smoothedArea > 0.7) {
                    if (msgEnabled) { distInfo.innerText = "STOPP"; distInfo.style.color = "#ff4d4d"; }
                    if (now > nextBeepTime) { playBeep(1200, 0.15); nextBeepTime = now + 0.05; }
                } else {
                    if (msgEnabled) { distInfo.innerText = "⚠️"; distInfo.style.color = "#ffcc00"; }
                    // Schnelligkeit des Piepens berechnen
                    let pause = 0.8 - (smoothedArea * 1.1);
                    if (now > nextBeepTime) { playBeep(800, 0.08); nextBeepTime = now + Math.max(0.08, pause); }
                }
            } else {
                distInfo.innerText = msgEnabled ? "OK" : "";
                distInfo.style.color = "#2ecc71";
            }
            requestAnimationFrame(detect);
        }

        sensRange.oninput = () => sensVal.innerText = sensRange.value + "%";

        soundBtn.onclick = () => {
            soundEnabled = !soundEnabled;
            soundBtn.className = soundEnabled ? "active" : "off";
        };

        msgBtn.onclick = () => {
            msgEnabled = !msgEnabled;
            msgBtn.className = msgEnabled ? "active" : "off";
            if (!msgEnabled) distInfo.innerText = "";
        };

        startBtn.onclick = async () => {
            if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            if (audioCtx.state === 'suspended') await audioCtx.resume();
            
            startBtn.innerText = "LADE KI...";
            const stream = await navigator.mediaDevices.getUserMedia({ 
                video: { deviceId: camSelect.value ? { exact: camSelect.value } : undefined } 
            });
            video.srcObject = stream;
            model = await cocoSsd.load();
            startBtn.style.display = "none";
            detect();
        };
    </script>
</body>
</html>
