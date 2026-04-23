<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Radar Wand-Modus</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        body { margin: 0; background: #000; color: #fff; font-family: sans-serif; display: flex; flex-direction: column; height: 100vh; overflow: hidden; }
        #video-container { flex: 1; position: relative; background: #000; display: flex; justify-content: center; overflow: hidden; }
        video, canvas { position: absolute; height: 100%; width: auto; }
        canvas { opacity: 0; } /* Unsichtbarer Canvas für die Analyse */
        #overlay { position: absolute; inset: 0; border: 20px solid transparent; pointer-events: none; z-index: 10; }
        #controls { padding: 15px; background: #111; display: grid; grid-template-columns: 1fr 1fr; gap: 10px; z-index: 20; }
        .full { grid-column: span 2; }
        button, select { padding: 15px; border-radius: 10px; border: none; background: #333; color: white; font-size: 1rem; }
        #distance-info { position: absolute; top: 20px; width: 100%; text-align: center; font-size: 4rem; font-weight: 800; z-index: 15; text-shadow: 0 0 15px #000; }
    </style>
</head>
<body>

    <div id="video-container">
        <video id="webcam" autoplay playsinline></video>
        <canvas id="analyzer"></canvas>
        <div id="overlay"></div>
        <div id="distance-info">STOP</div>
    </div>

    <div id="controls">
        <select id="cameraSelect" class="full"></select>
        <button id="toggleSound" style="background: #007aff;">TON: AN</button>
        <div class="full">
            <label style="font-size: 0.7rem; color: #777;">WAND-EMPFINDLICHKEIT (Niedriger = reagiert früher):</label>
            <input type="range" id="sensRange" min="5" max="60" value="15" style="width:100%">
        </div>
        <button id="startBtn" class="full" style="background: #28a745; font-weight: bold;">START</button>
    </div>

    <script>
        const video = document.getElementById('webcam');
        const canvas = document.getElementById('analyzer');
        const ctx = canvas.getContext('2d', {willReadFrequently: true});
        const startBtn = document.getElementById('startBtn');
        const camSelect = document.getElementById('cameraSelect');
        const distInfo = document.getElementById('distance-info');
        const overlay = document.getElementById('overlay');
        
        let model, audioCtx, lastBeep = 0, smoothedArea = 0;
        let prevFrameData = null;

        async function getCams() {
            const devices = await navigator.mediaDevices.enumerateDevices();
            const vids = devices.filter(d => d.kind === 'videoinput');
            camSelect.innerHTML = vids.map((d, i) => `<option value="${d.deviceId}">${d.label || 'Kamera '+(i+1)}</option>`).join('');
        }
        getCams();

        function beep(f, d) {
            if (!audioCtx || audioCtx.state !== 'running') return;
            const o = audioCtx.createOscillator(), g = audioCtx.createGain();
            o.type = 'sine'; o.frequency.value = f;
            g.gain.setValueAtTime(0.1, audioCtx.currentTime);
            g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + d);
            o.connect(g); g.connect(audioCtx.destination);
            o.start(); o.stop(audioCtx.currentTime + d);
        }

        function analyzeWall() {
            // Wir zeichnen ein kleines Abbild für die Analyse
            ctx.drawImage(video, 0, 0, 64, 64);
            const currFrame = ctx.getImageData(0, 0, 64, 64).data;
            let diff = 0;

            if (prevFrameData) {
                for (let i = 0; i < currFrame.length; i += 4) {
                    // Wir messen die Veränderung der Helligkeit
                    diff += Math.abs(currFrame[i] - prevFrameData[i]);
                }
            }
            prevFrameData = currFrame;
            // Normalisierter Wert für "Bewegung/Annäherung"
            return diff / (64 * 64 * 3); 
        }

        async function detect() {
            const predictions = await model.detect(video);
            const wallDiff = analyzeWall();
            let frameMax = 0;

            // 1. KI-Check (für Autos, Personen etc.)
            for (let p of predictions) {
                if (p.score > 0.4) {
                    const a = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                    if (a > frameMax) frameMax = a;
                }
            }

            // 2. Wand-Check: Wenn die KI nichts sieht, aber die Pixel sich stark ändern
            // (Simuliert die optische Vergrößerung einer Fläche)
            if (frameMax < 0.1 && wallDiff > 15) {
                 frameMax = wallDiff / 40; // Wand-Bewegung in "Fläche" umrechnen
            }

            // Schnelle Glättung
            smoothedArea = (smoothedArea * 0.4) + (frameMax * 0.6);

            const threshold = document.getElementById('sensRange').value / 100;
            const now = audioCtx.currentTime;

            if (smoothedArea > threshold) {
                if (smoothedArea > 0.6) {
                    distInfo.innerText = "!!! STOPP !!!";
                    distInfo.style.color = "red";
                    overlay.style.borderColor = "red";
                    if (now - lastBeep > 0.07) { beep(1200, 0.1); lastBeep = now; }
                } else {
                    distInfo.innerText = "WAND/OBJEKT";
                    distInfo.style.color = "yellow";
                    overlay.style.borderColor = "yellow";
                    let interval = Math.max(0.07, 0.6 - (smoothedArea * 0.9));
                    if (now - lastBeep > interval) { beep(800, 0.05); lastBeep = now; }
                }
            } else {
                distInfo.innerText = "FREI";
                distInfo.style.color = "#28a745";
                overlay.style.borderColor = "transparent";
            }
            requestAnimationFrame(detect);
        }

        startBtn.onclick = async () => {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            const s = await navigator.mediaDevices.getUserMedia({ video: { deviceId: camSelect.value } });
            video.srcObject = s;
            startBtn.innerText = "KALIBRIERE...";
            model = await cocoSsd.load();
            startBtn.style.display = "none";
            detect();
        };
    </script>
</body>
</html>
