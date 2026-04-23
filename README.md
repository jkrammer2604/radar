<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Radar Final Fix</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        * { box-sizing: border-box; }
        body { margin: 0; background: #000; color: #fff; font-family: sans-serif; height: 100vh; display: flex; flex-direction: column; }
        
        /* WICHTIG: Das Video muss sichtbar sein */
        #video-container { 
            position: relative; 
            flex: 1; 
            width: 100vw; 
            background: #222; 
            display: flex; 
            justify-content: center; 
            align-items: center;
        }

        video { 
            width: 100%; 
            height: 100%; 
            object-fit: contain; /* Zeigt das ganze Kamerabild ohne Abschneiden */
            background: #000;
        }

        #overlay { 
            position: absolute; inset: 0; border: 15px solid transparent; 
            pointer-events: none; z-index: 10; 
        }

        #controls { 
            background: #111; padding: 15px; 
            display: grid; grid-template-columns: 1fr 1fr; gap: 10px; 
            border-top: 1px solid #333;
        }

        .full { grid-column: span 2; }
        button, select { padding: 15px; border-radius: 10px; border: none; background: #333; color: white; font-size: 1rem; }
        
        #distance-info { 
            position: absolute; top: 10%; width: 100%; text-align: center; 
            font-size: 3rem; font-weight: bold; z-index: 20; text-shadow: 2px 2px 10px #000; 
        }
    </style>
</head>
<body>

    <div id="video-container">
        <video id="webcam" autoplay playsinline muted></video>
        <div id="overlay"></div>
        <div id="distance-info">BEREIT</div>
    </div>

    <div id="controls">
        <select id="cameraSelect" class="full"><option>Kamera wird gesucht...</option></select>
        <button id="toggleSound" onclick="toggleSound()" style="background:#007aff">TON: AN</button>
        <button id="startBtn" onclick="startSystem()" style="background:#28a745; font-weight:bold">SYSTEM STARTEN</button>
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

        // 1. Kameras finden
        async function getCams() {
            try {
                // Erster Dummy-Aufruf um Berechtigung zu triggern
                await navigator.mediaDevices.getUserMedia({ video: true });
                const devices = await navigator.mediaDevices.enumerateDevices();
                const vids = devices.filter(d => d.kind === 'videoinput');
                camSelect.innerHTML = vids.map((d, i) => 
                    `<option value="${d.deviceId}">${d.label || 'Kamera '+(i+1)}</option>`
                ).join('');
            } catch (e) {
                distInfo.innerText = "Kamera-Zugriff verweigert";
            }
        }
        getCams();

        function toggleSound() {
            soundOn = !soundOn;
            document.getElementById('toggleSound').innerText = soundOn ? "TON: AN" : "TON: AUS";
            document.getElementById('toggleSound').style.background = soundOn ? "#007aff" : "#444";
        }

        async function startSystem() {
            // Audio initialisieren
            if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            if (audioCtx.state === 'suspended') await audioCtx.resume();

            startBtn.innerText = "LADE...";

            try {
                const constraints = {
                    video: {
                        deviceId: camSelect.value ? { exact: camSelect.value } : undefined,
                        width: { ideal: 1280 },
                        height: { ideal: 720 }
                    }
                };

                const stream = await navigator.mediaDevices.getUserMedia(constraints);
                video.srcObject = stream;
                
                // WICHTIG: Sicherstellen, dass das Video wirklich abspielt
                video.onloadedmetadata = () => {
                    video.play();
                };

                model = await cocoSsd.load();
                startBtn.style.display = "none";
                detect();
            } catch (err) {
                alert("Fehler beim Kamerastart: " + err.message);
            }
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
            const predictions = await model.detect(video);
            
            // Wand-Analyse (Pixel-Diff)
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

            // Mix aus KI und Wand
            if (frameMax < 0.1 && wallVal > 12) frameMax = wallVal / 35;

            smoothedArea = (smoothedArea * 0.5) + (frameMax * 0.5);
            const now = audioCtx.currentTime;

            if (smoothedArea > 0.15) {
                if (smoothedArea > 0.6) {
                    distInfo.innerText = "STOPP"; distInfo.style.color = "red";
                    overlay.style.borderColor = "red";
                    if (now - lastBeep > 0.08) { beep(1200, 0.1); lastBeep = now; }
                } else {
                    distInfo.innerText = "ACHTUNG"; distInfo.style.color = "yellow";
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
