<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Radar Ultra-Stabil</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        body { margin: 0; background: #000; color: #fff; font-family: -apple-system, sans-serif; display: flex; flex-direction: column; height: 100vh; }
        #video-container { flex: 1; position: relative; background: #000; overflow: hidden; display: flex; justify-content: center; }
        video { height: 100%; width: auto; }
        #overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: 10px solid transparent; pointer-events: none; transition: border 0.2s; }
        
        #controls { padding: 20px; background: #1a1a1a; display: grid; grid-template-columns: 1fr 1fr; gap: 15px; }
        .full { grid-column: span 2; }
        
        button, select { padding: 18px; border-radius: 12px; border: none; background: #333; color: white; font-size: 1rem; -webkit-appearance: none; }
        .active { background: #28a745; }
        .off { background: #444; color: #888; }
        
        input[type=range] { width: 100%; margin: 15px 0; }
        #distance-info { position: absolute; top: 30px; width: 100%; text-align: center; font-size: 5rem; font-weight: 900; pointer-events: none; }
    </style>
</head>
<body>

    <div id="video-container">
        <video id="webcam" autoplay playsinline></video>
        <div id="overlay"></div>
        <div id="distance-info"></div>
    </div>

    <div id="controls">
        <select id="cameraSelect" class="full"></select>
        
        <button id="toggleSound" class="active">TON AN</button>
        <button id="toggleMsg" class="active">TEXT AN</button>

        <div class="full">
            <label style="font-size: 0.8rem; color: #888;">REAKTIONS-SCHWELLE (Höher = weniger Fehlalarme):</label>
            <input type="range" id="sensRange" min="10" max="90" value="40">
        </div>

        <button id="startBtn" class="full" style="background: #007aff; font-weight: bold;">SYSTEM AKTIVIEREN</button>
    </div>

    <script>
        const video = document.getElementById('webcam');
        const startBtn = document.getElementById('startBtn');
        const camSelect = document.getElementById('cameraSelect');
        const distInfo = document.getElementById('distance-info');
        const soundBtn = document.getElementById('toggleSound');
        const msgBtn = document.getElementById('toggleMsg');
        const sensRange = document.getElementById('sensRange');
        const overlay = document.getElementById('overlay');

        let model, audioCtx;
        let soundOn = true, msgOn = true;
        let lastBeepTime = 0;
        let smoothedArea = 0; // Der "Stoßdämpfer"-Wert

        // Kameras finden
        async function getCams() {
            const devices = await navigator.mediaDevices.enumerateDevices();
            const videoDevices = devices.filter(d => d.kind === 'videoinput');
            camSelect.innerHTML = videoDevices.map((d, i) => `<option value="${d.deviceId}">${d.label || 'Kamera ' + (i+1)}</option>`).join('');
        }
        getCams();

        function beep(freq, dur) {
            if (!audioCtx || !soundOn) return;
            const osc = audioCtx.createOscillator();
            const g = audioCtx.createGain();
            osc.type = 'sine';
            osc.frequency.value = freq;
            g.gain.setValueAtTime(0.1, audioCtx.currentTime);
            g.gain.linearRampToValueAtTime(0.0001, audioCtx.currentTime + dur);
            osc.connect(g);
            g.connect(audioCtx.destination);
            osc.start();
            osc.stop(audioCtx.currentTime + dur);
        }

        async function loop() {
            const predictions = await model.detect(video);
            let frameMaxArea = 0;

            // FILTER: Wir suchen nur nach relevanten Objekten mit hoher Sicherheit
            for (let p of predictions) {
                if (p.score > 0.65) { 
                    const area = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                    if (area > frameMaxArea) frameMaxArea = area;
                }
            }

            // TIEFPASS-FILTER (Smoothing)
            // Wir nehmen nur 20% des neuen Wertes und 80% des alten. 
            // Das verhindert extremes Springen der Werte.
            smoothedArea = (smoothedArea * 0.8) + (frameMaxArea * 0.2);

            const threshold = sensRange.value / 100;
            const now = audioCtx.currentTime;

            if (smoothedArea > threshold) {
                // WARN-LOGIK
                if (smoothedArea > 0.75) {
                    // STOPP (Dauerton-Effekt)
                    if (msgOn) { distInfo.innerText = "STOPP"; distInfo.style.color = "red"; }
                    overlay.style.borderColor = "red";
                    if (now - lastBeepTime > 0.08) { beep(1000, 0.1); lastBeepTime = now; }
                } else {
                    // ANNÄHERUNG
                    if (msgOn) { distInfo.innerText = "⚠️"; distInfo.style.color = "yellow"; }
                    overlay.style.borderColor = "yellow";
                    let interval = Math.max(0.08, 0.7 - (smoothedArea * 0.9));
                    if (now - lastBeepTime > interval) { beep(800, 0.06); lastBeepTime = now; }
                }
            } else {
                if (msgOn) distInfo.innerText = "OK";
                overlay.style.borderColor = "transparent";
                distInfo.style.color = "#28a745";
            }

            requestAnimationFrame(loop);
        }

        startBtn.onclick = async () => {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            startBtn.innerText = "WARTE AUF KAMERA...";
            const stream = await navigator.mediaDevices.getUserMedia({ video: { deviceId: camSelect.value } });
            video.srcObject = stream;
            startBtn.innerText = "LADE KI...";
            model = await cocoSsd.load();
            startBtn.style.display = "none";
            loop();
        };

        soundBtn.onclick = () => { soundOn = !soundOn; soundBtn.className = soundOn ? "active" : "off"; };
        msgBtn.onclick = () => { msgOn = !msgOn; msgBtn.className = msgOn ? "active" : "off"; if(!msgOn) distInfo.innerText=""; };
    </script>
</body>
</html>
