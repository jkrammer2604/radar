<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Radar Wand-Modus Fix</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        body { margin: 0; background: #000; color: #fff; font-family: sans-serif; display: flex; flex-direction: column; height: 100vh; overflow: hidden; }
        
        #video-container { 
            flex: 1; position: relative; background: #000; 
            display: flex; justify-content: center; align-items: center; overflow: hidden; 
        }
        
        /* Das eigentliche Kamerabild */
        video { 
            position: absolute; width: 100%; height: 100%; object-fit: cover; z-index: 1; 
        }
        
        /* Unsichtbarer Analyse-Canvas */
        #analyzer { display: none; } 

        /* Roter/Gelber Rahmen bei Gefahr */
        #overlay { 
            position: absolute; inset: 0; border: 20px solid transparent; 
            pointer-events: none; z-index: 5; transition: border 0.1s;
        }
        
        #distance-info { 
            position: absolute; top: 20%; width: 100%; text-align: center; 
            font-size: 5rem; font-weight: 900; z-index: 10; 
            text-shadow: 0 0 20px #000; pointer-events: none;
        }

        #controls { 
            padding: 20px; background: #111; display: grid; 
            grid-template-columns: 1fr 1fr; gap: 10px; z-index: 20; 
            border-top: 2px solid #333;
        }
        
        .full { grid-column: span 2; }
        button, select { padding: 18px; border-radius: 12px; border: none; background: #333; color: white; font-size: 1.1rem; }
        input[type=range] { width: 100%; margin: 10px 0; }
        label { font-size: 0.8rem; color: #888; }
    </style>
</head>
<body>

    <div id="video-container">
        <video id="webcam" autoplay playsinline muted></video>
        <canvas id="analyzer" width="64" height="64"></canvas>
        <div id="overlay"></div>
        <div id="distance-info">BEREIT</div>
    </div>

    <div id="controls">
        <select id="cameraSelect" class="full"></select>
        <button id="toggleSound" style="background: #007aff;">TON: AN</button>
        <div class="full">
            <label>SENSITIVITÄT (Wand & Objekte):</label>
            <input type="range" id="sensRange" min="5" max="60" value="20">
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
        let soundOn = true;

        async function getCams() {
            const devices = await navigator.mediaDevices.enumerateDevices();
            const vids = devices.filter(d => d.kind === 'videoinput');
            camSelect.innerHTML = vids.map((d, i) => `<option value="${d.deviceId}">${d.label || 'Kamera '+(i+1)}</option>`).join('');
        }
        getCams();

        function beep(f, d) {
            if (!audioCtx || !soundOn || audioCtx.state !== 'running') return;
            const o = audioCtx.createOscillator(), g = audioCtx.createGain();
            o.type = 'sine'; o.frequency.value = f;
            g.gain.setValueAtTime(0.1, audioCtx.currentTime);
            g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + d);
            o.connect(g); g.connect(audioCtx.destination);
            o.start(); o.stop(audioCtx.currentTime + d);
        }

        function analyzeWall() {
            ctx.drawImage(video, 0, 0, 64, 64);
            const currFrame = ctx.getImageData(0, 0, 64, 64).data;
            let diff = 0;
            if (prevFrameData) {
                for (let i = 0; i < currFrame.length; i += 4) {
                    diff += Math.abs(currFrame[i] - prevFrameData[i]);
                }
            }
            prevFrameData = currFrame;
            return diff / (64 * 64 * 3); 
        }

        async function detect() {
            const predictions = await model.detect(video);
            const wallDiff = analyzeWall();
            let frameMax = 0;

            for (let p of predictions) {
                if (p.score > 0.45) {
                    const a = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                    if (a > frameMax) frameMax = a;
                }
            }

            // Wand-Logik: Falls KI nichts sieht, aber Pixel-Änderung hoch ist
            if (frameMax < 0.1 && wallDiff > 12) {
                 frameMax = wallDiff / 35; 
            }

            // Schnelle Reaktion
            smoothedArea = (smoothedArea * 0.5) + (frameMax * 0.5);

            const threshold = document.getElementById('sensRange').value / 100;
            const now = audioCtx.currentTime;

            if (smoothedArea > threshold) {
                if (smoothedArea > 0.6) {
                    distInfo.innerText = "STOPP";
                    distInfo.style.color = "red";
                    overlay.style.borderColor = "red";
                    if (now - lastBeep > 0.08) { beep(1200, 0.1); lastBeep = now; }
                } else {
                    distInfo.innerText = "ACHTUNG";
                    distInfo.style.color = "yellow";
                    overlay.style.borderColor = "yellow";
                    let interval = Math.max(0.08, 0.6 - (smoothedArea * 0.9));
                    if (now - lastBeep > interval) { beep(800, 0.05); lastBeep = now; }
                }
            } else {
                distInfo.innerText = "OK";
                distInfo.style.color = "#28a745";
                overlay.style.borderColor = "transparent";
            }
            requestAnimationFrame(detect);
        }

        startBtn.onclick = async () => {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            const s = await navigator.mediaDevices.getUserMedia({ 
                video: { deviceId: camSelect.value ? { exact: camSelect.value } : undefined } 
            });
            video.srcObject = s;
            startBtn.innerText = "LADE KI...";
            model = await cocoSsd.load();
            startBtn.style.display = "none";
            detect();
        };

        document.getElementById('toggleSound').onclick = function() {
            soundOn = !soundOn;
            this.innerText = soundOn ? "TON: AN" : "TON: AUS";
            this.style.background = soundOn ? "#007aff" : "#444";
        };
    </script>
</body>
</html>
