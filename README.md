<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Radar Zentriert</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        html, body { 
            margin: 0; padding: 0; width: 100%; height: 100%; 
            background: #000; color: #fff; font-family: sans-serif; 
            overflow: hidden; display: flex; flex-direction: column; 
        }

        /* Der Container hält alles zusammen */
        #video-container { 
            position: relative; 
            flex: 1; 
            width: 100vw; 
            overflow: hidden; /* Schneidet Überstehendes ab */
            display: flex; 
            justify-content: center; 
            align-items: center;
        }

        /* Video perfekt zentriert */
        video { 
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%); /* Das schiebt es exakt in die Mitte */
            min-width: 100%; 
            min-height: 100%;
            width: auto;
            height: auto;
            object-fit: contain; 
            z-index: 1;
        }

        #overlay { 
            position: absolute; inset: 0; 
            border: 20px solid transparent; 
            pointer-events: none; z-index: 10; 
            transition: border 0.1s;
        }

        #distance-info { 
            position: absolute; top: 15%; width: 100%; text-align: center; 
            font-size: 4rem; font-weight: 900; z-index: 20; 
            text-shadow: 0 0 20px #000; pointer-events: none;
        }

        #controls { 
            background: #111; padding: 20px; 
            display: grid; grid-template-columns: 1fr 1fr; gap: 10px; 
            z-index: 30; border-top: 1px solid #333;
        }

        .full { grid-column: span 2; }
        button, select { padding: 18px; border-radius: 12px; border: none; background: #333; color: white; font-size: 1.1rem; }
        input[type=range] { width: 100%; margin: 10px 0; }
    </style>
</head>
<body>

    <div id="video-container">
        <video id="webcam" autoplay playsinline muted></video>
        <div id="overlay"></div>
        <div id="distance-info">BEREIT</div>
    </div>

    <div id="controls">
        <select id="cameraSelect" class="full"></select>
        <button id="toggleSound" onclick="toggleSound()" style="background:#007aff">🔊 TON AN</button>
        <div class="full">
            <input type="range" id="sensRange" min="5" max="70" value="25">
        </div>
        <button id="startBtn" onclick="startSystem()" class="full" style="background:#28a745; font-weight:bold">SYSTEM STARTEN</button>
    </div>

    <canvas id="analyzer" width="64" height="64" style="display:none;"></canvas>

    <script>
        const video = document.getElementById('webcam');
        const startBtn = document.getElementById('startBtn');
        const camSelect = document.getElementById('cameraSelect');
        const distInfo = document.getElementById('distance-info');
        const overlay = document.getElementById('overlay');
        const canvas = document.getElementById('analyzer');
        const ctx = canvas.getContext('2d', {willReadFrequently: true});

        let model, audioCtx, lastBeep = 0, smoothedArea = 0, prevFrameData = null, soundOn = true;

        async function getCams() {
            try {
                await navigator.mediaDevices.getUserMedia({ video: true });
                const devices = await navigator.mediaDevices.enumerateDevices();
                const vids = devices.filter(d => d.kind === 'videoinput');
                camSelect.innerHTML = vids.map((d, i) => 
                    `<option value="${d.deviceId}">${d.label || 'Kamera '+(i+1)}</option>`
                ).join('');
            } catch (e) { distInfo.innerText = "Kamera-Fehler"; }
        }
        getCams();

        function toggleSound() {
            soundOn = !soundOn;
            const btn = document.getElementById('toggleSound');
            btn.innerText = soundOn ? "🔊 TON AN" : "🔇 TON AUS";
            btn.style.background = soundOn ? "#007aff" : "#444";
        }

        async function startSystem() {
            if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            if (audioCtx.state === 'suspended') await audioCtx.resume();
            
            startBtn.innerText = "LÄDT...";
            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: { deviceId: camSelect.value ? { exact: camSelect.value } : undefined }
                });
                video.srcObject = stream;
                model = await cocoSsd.load();
                startBtn.style.display = "none";
                detect();
            } catch (err) { alert("Start-Fehler: " + err.message); }
        }

        function beep(f, d) {
            if (!audioCtx || !soundOn) return;
            const o = audioCtx.createOscillator(), g = audioCtx.createGain();
            o.type = 'sine'; o.frequency.value = f;
            g.gain.setValueAtTime(0.1, audioCtx.currentTime);
            g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + d);
            o.connect(g); g.connect(audioCtx.destination);
            o.start(); o.stop(audioCtx.currentTime + d);
        }

        async function detect() {
            if (!model) return;
            const predictions = await model.detect(video);
            
            ctx.drawImage(video, 0, 0, 64, 64);
            const currFrame = ctx.getImageData(0, 0, 64, 64).data;
            let diff = 0;
            if (prevFrameData) {
                for (let i = 0; i < currFrame.length; i += 4) {
                    diff += Math.abs(currFrame[i] - prevFrameData[i]);
                }
            }
            prevFrameData = currFrame;
            const wallVal = diff / (64 * 64 * 3);

            let frameMax = 0;
            for (let p of predictions) {
                if (p.score > 0.4) {
                    const a = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                    if (a > frameMax) frameMax = a;
                }
            }

            if (frameMax < 0.1 && wallVal > 12) frameMax = wallVal / 35;

            smoothedArea = (smoothedArea * 0.5) + (frameMax * 0.5);
            const threshold = document.getElementById('sensRange').value / 100;
            const now = audioCtx.currentTime;

            if (smoothedArea > threshold) {
                if (smoothedArea > 0.6) {
                    distInfo.innerText = "STOPP"; distInfo.style.color = "red";
                    overlay.style.borderColor = "red";
                    if (now - lastBeep > 0.08) { beep(1200, 0.1); lastBeep = now; }
                } else {
                    distInfo.innerText = "⚠️"; distInfo.style.color = "yellow";
                    overlay.style.borderColor = "yellow";
                    let interval = Math.max(0.08, 0.6 - (smoothedArea * 0.9));
                    if (now - lastBeep > interval) { beep(800, 0.05); lastBeep = now; }
                }
            } else {
                distInfo.innerText = "OK"; distInfo.style.color = "#28a745";
                overlay.style.borderColor = "transparent";
            }
            requestAnimationFrame(detect);
        }
    </script>
</body>
</html>
