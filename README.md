<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Radar Fix</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        body { 
            margin: 0; background: #111; color: white; 
            font-family: sans-serif; display: flex; flex-direction: column; height: 100vh; 
        }
        /* Video verkleinert, damit Platz für UI bleibt */
        #video-container { position: relative; flex: 1; background: #000; display: flex; justify-content: center; align-items: center; }
        video { max-width: 100%; max-height: 70vh; border: 2px solid #444; }
        
        #controls { 
            padding: 20px; background: #222; text-align: center; 
            border-top: 2px solid #444; z-index: 100;
        }
        
        #startBtn { 
            background: #28a745; color: white; border: none; 
            padding: 25px 50px; font-size: 1.8rem; font-weight: bold;
            border-radius: 15px; box-shadow: 0 4px 15px rgba(0,0,0,0.5);
        }
        
        #status-box { font-size: 1.2rem; margin-bottom: 10px; color: #aaa; }
        #distance-info { font-size: 2.5rem; font-weight: bold; margin: 10px 0; }
        .stop { color: #ff4d4d; }
        .warn { color: #ffcc00; }
        .ok { color: #4dff4d; }
    </style>
</head>
<body>

    <div id="video-container">
        <video id="webcam" autoplay playsinline></video>
    </div>

    <div id="controls">
        <div id="status-box">System bereit</div>
        <div id="distance-info" class="ok">BEREIT</div>
        <button id="startBtn">RADAR STARTEN</button>
    </div>

    <script>
        const video = document.getElementById('webcam');
        const startBtn = document.getElementById('startBtn');
        const statusBox = document.getElementById('status-box');
        const distInfo = document.getElementById('distance-info');
        
        let model, audioCtx, nextBeepTime = 0;

        function playBeep(freq, dur) {
            if (!audioCtx) return;
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

        async function startSystem() {
            statusBox.innerText = "Suche Kamera & lade KI...";
            
            try {
                // Versuche die externe Kamera zu finden
                const devices = await navigator.mediaDevices.enumerateDevices();
                const videoDevices = devices.filter(d => d.kind === 'videoinput');
                
                // Wir nehmen die letzte Kamera (oft die externe bei USB)
                const selectedId = videoDevices.length > 0 ? videoDevices[videoDevices.length - 1].deviceId : null;
                
                const stream = await navigator.mediaDevices.getUserMedia({ 
                    video: selectedId ? { deviceId: { exact: selectedId } } : { facingMode: "environment" }
                });
                
                video.srcObject = stream;
                model = await cocoSsd.load();
                
                statusBox.innerText = "Radar aktiv!";
                startBtn.style.display = "none";
                detect();
            } catch (err) {
                statusBox.innerText = "Fehler: " + err.message;
                console.error(err);
            }
        }

        async function detect() {
            if (!model) return;
            const predictions = await model.detect(video);
            let maxArea = 0;

            predictions.forEach(p => {
                const area = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                if (area > maxArea) maxArea = area;
            });

            const now = audioCtx.currentTime;

            if (maxArea > 0.05) {
                if (maxArea > 0.55) {
                    distInfo.innerText = "!!! STOPP !!!";
                    distInfo.className = "stop";
                    if (now > nextBeepTime) {
                        playBeep(1000, 0.2);
                        nextBeepTime = now + 0.08; // Dauerton
                    }
                } else {
                    distInfo.innerText = "ABSTAND REDUZIERT";
                    distInfo.className = "warn";
                    let pause = 0.7 - (maxArea * 1.1);
                    pause = Math.max(0.1, pause);
                    if (now > nextBeepTime) {
                        playBeep(800, 0.1);
                        nextBeepTime = now + pause;
                    }
                }
            } else {
                distInfo.innerText = "WEG FREI";
                distInfo.className = "ok";
            }
            requestAnimationFrame(detect);
        }

        startBtn.onclick = () => {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            startSystem();
        };
    </script>
</body>
</html>
