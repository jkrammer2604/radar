<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Radar Compact</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        * { box-sizing: border-box; }
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; background: #000; overflow: hidden; font-family: sans-serif; }
        
        /* Video im Hintergrund */
        #video-container { position: absolute; inset: 0; z-index: 1; }
        video { width: 100%; height: 100%; object-fit: cover; }

        /* Warnanzeige: Kleiner und dezenter oben */
        #distance-info { 
            position: absolute; top: 50px; width: 100%; text-align: center; 
            font-size: 3rem; font-weight: bold; z-index: 20; 
            text-shadow: 0 0 10px #000; pointer-events: none; 
        }

        #overlay { position: absolute; inset: 0; border: 15px solid transparent; pointer-events: none; z-index: 10; }

        /* Steuerung: Kompakter am unteren Rand */
        #controls { 
            position: absolute; bottom: 0; width: 100%; 
            background: rgba(0, 0, 0, 0.8); padding: 10px; 
            display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 8px; 
            z-index: 30; backdrop-filter: blur(10px);
        }

        .full { grid-column: span 3; }
        button, select { 
            padding: 12px; border-radius: 8px; border: none; 
            background: #333; color: white; font-size: 0.9rem; 
        }
        
        /* Kleinerer Slider */
        input[type=range] { width: 100%; height: 20px; margin: 5px 0; }
        label { font-size: 0.7rem; color: #bbb; display: block; }
    </style>
</head>
<body>

    <div id="video-container">
        <video id="webcam" autoplay playsinline muted></video>
        <div id="overlay"></div>
        <div id="distance-info">BEREIT</div>
    </div>

    <div id="controls">
        <select id="cameraSelect">
            <option>Lade Kameras...</option>
        </select>
        <button id="toggleSound" onclick="toggleSound()" style="background:#007aff">🔊 TON</button>
        <button id="startBtn" onclick="startSystem()" style="background:#28a745; font-weight:bold">START</button>
        
        <div class="full">
            <label>SENSITIVITÄT (Wand & Objekte):</label>
            <input type="range" id="sensRange" min="5" max="80" value="25">
        </div>
    </div>

    <canvas id="analyzer" width="160" height="120" style="display:none;"></canvas>

    <script>
        const video = document.getElementById('webcam');
        const camSelect = document.getElementById('cameraSelect');
        const distInfo = document.getElementById('distance-info');
        const overlay = document.getElementById('overlay');
        const canvas = document.getElementById('analyzer');
        const ctx = canvas.getContext('2d', {willReadFrequently: true});

        let model, audioCtx, lastBeep = 0, smoothedArea = 0, prevFrameData = null, soundOn = true;

        async function getCams() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ video: true });
                stream.getTracks().forEach(t => t.stop());
                const devices = await navigator.mediaDevices.enumerateDevices();
                const vids = devices.filter(d => d.kind === 'videoinput');
                camSelect.innerHTML = vids.map((d, i) => 
                    `<option value="${d.deviceId}">${d.label || 'Cam '+(i+1)}</option>`
                ).join('');
            } catch (e) { distInfo.innerText = "Cam Error"; }
        }
        getCams();

        function toggleSound() {
            soundOn = !soundOn;
            document.getElementById('toggleSound').innerText = soundOn ? "🔊 TON" : "🔇 AUS";
        }

        async function startSystem() {
            if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            document.getElementById('startBtn').innerText = "...";
            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: { deviceId: camSelect.value ? { exact: camSelect.value } : undefined }
                });
                video.srcObject = stream;
                model = await cocoSsd.load({ base: 'mobilenet_v2' });
                document.getElementById('startBtn').style.display = "none";
                detect();
            } catch (err) { alert(err.message); }
        }

        function beep(f, d) {
            if (!audioCtx || !soundOn) return;
            const o = audioCtx.createOscillator(), g = audioCtx.createGain();
            o.type = 'sine'; o.frequency.value = f;
            g.gain.setValueAtTime(0.05, audioCtx.currentTime);
            g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + d);
            o.connect(g); g.connect(audioCtx.destination);
            o.start(); o.stop(audioCtx.currentTime + d);
        }

        async function detect() {
            const predictions = await model.detect(video);
            ctx.drawImage(video, 0, 0, 160, 120);
            const currFrame = ctx.getImageData(0, 0, 160, 120).data;
            let diff = 0;
            if (prevFrameData) {
                for (let i = 0; i < currFrame.length; i += 4) {
                    diff += Math.abs(currFrame[i] - prevFrameData[i]);
                }
            }
            prevFrameData = currFrame;
            const wallVal = diff / (160 * 120 * 3);

            let frameMax = 0;
            for (let p of predictions) {
                if (p.score > 0.35) {
                    const a = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                    if (a > frameMax) frameMax = a;
                }
            }

            if (frameMax < 0.1 && wallVal > 10) frameMax = wallVal / 30;
            smoothedArea = (smoothedArea * 0.4) + (frameMax * 0.6);
            const threshold = document.getElementById('sensRange').value / 100;
            const now = audioCtx.currentTime;

            if (smoothedArea > threshold) {
                if (smoothedArea > 0.65) {
                    distInfo.innerText = "STOPP"; distInfo.style.color = "#ff3b30";
                    overlay.style.borderColor = "#ff3b30";
                    if (now - lastBeep > 0.08) { beep(1200, 0.1); lastBeep = now; }
                } else {
                    distInfo.innerText = "⚠️"; distInfo.style.color = "#ffcc00";
                    overlay.style.borderColor = "#ffcc00";
                    let interval = Math.max(0.08, 0.6 - (smoothedArea * 0.9));
                    if (now - lastBeep > interval) { beep(800, 0.05); lastBeep = now; }
                }
            } else {
                distInfo.innerText = "OK"; distInfo.style.color = "#4cd964";
                overlay.style.borderColor = "transparent";
            }
            requestAnimationFrame(detect);
        }
    </script>
</body>
</html>
