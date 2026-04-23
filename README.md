<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>iPad Park-Radar</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        body { margin: 0; background: #000; color: white; font-family: sans-serif; text-align: center; overflow: hidden; }
        video { width: 100%; height: auto; border-bottom: 2px solid #333; transform: scaleX(-1); } /* Spiegelbild für einfacheres Rangieren */
        #ui { padding: 20px; }
        .distance-display { font-size: 3rem; font-weight: bold; margin: 10px; }
        #startBtn { 
            padding: 20px 40px; font-size: 1.5rem; background: #28a745; 
            color: white; border: none; border-radius: 10px; cursor: pointer; 
        }
        .warning-red { color: #ff4d4d; }
        .warning-yellow { color: #ffcc00; }
        .warning-green { color: #4dff4d; }
    </style>
</head>
<body>

    <video id="webcam" autoplay playsinline></video>
    
    <div id="ui">
        <div id="info">System bereit</div>
        <div id="distText" class="distance-display">---</div>
        <button id="startBtn">AKTIVIEREN & SOUND AN</button>
    </div>

    <script>
        const video = document.getElementById('webcam');
        const distText = document.getElementById('distText');
        const startBtn = document.getElementById('startBtn');
        const info = document.getElementById('info');
        
        let model;
        let audioCtx;
        let nextBeepTime = 0;

        // Erzeugt den Piepton (800Hz = klassischer Parkwarner-Ton)
        function playBeep(duration) {
            if (!audioCtx) return;
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            
            osc.type = 'sine'; 
            osc.frequency.setValueAtTime(800, audioCtx.currentTime);
            
            gain.gain.setValueAtTime(0.2, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + duration);
            
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            
            osc.start();
            osc.stop(audioCtx.currentTime + duration);
        }

        async function init() {
            // Kamera-Zugriff (Versucht die externe oder Rück-Kamera)
            const stream = await navigator.mediaDevices.getUserMedia({ 
                video: { facingMode: "environment" } 
            });
            video.srcObject = stream;

            // KI Modell laden
            info.innerText = "Lade KI-Modell...";
            model = await cocoSsd.load();
            info.innerText = "Radar läuft!";
            
            detectFrame();
        }

        async function detectFrame() {
            const predictions = await model.detect(video);
            let largestObjectArea = 0;

            predictions.forEach(p => {
                // Berechnet Größe des Objekts im Bild (0.0 bis 1.0)
                const area = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                if (area > largestObjectArea) largestObjectArea = area;
            });

            const now = audioCtx.currentTime;

            if (largestObjectArea > 0.05) { // Objekt erkannt
                if (largestObjectArea > 0.6) {
                    // DAUERTON (Ganz nah)
                    distText.innerText = "STOPP";
                    distText.className = "distance-display warning-red";
                    if (now > nextBeepTime) {
                        playBeep(0.4); 
                        nextBeepTime = now + 0.05; // Sehr kurze Pause = Dauerton-Gefühl
                    }
                } else {
                    // INTERVALL-PIEPEN
                    distText.innerText = "ACHTUNG";
                    distText.className = "distance-display warning-yellow";
                    
                    // Berechne Pause: Je größer das Objekt, desto kleiner die Pause
                    // Bereich von ca. 0.7s (weit weg) bis 0.1s (nah dran)
                    let pause = 0.8 - (largestObjectArea * 1.2);
                    pause = Math.max(0.1, pause); 

                    if (now > nextBeepTime) {
                        playBeep(0.1);
                        nextBeepTime = now + pause;
                    }
                }
            } else {
                distText.innerText = "OK";
                distText.className = "distance-display warning-green";
            }

            requestAnimationFrame(detectFrame);
        }

        startBtn.onclick = () => {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            startBtn.style.display = "none";
            init();
        };
    </script>
</body>
</html>
