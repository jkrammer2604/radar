<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>iPad Radar Assistent</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd"></script>
    <style>
        body { 
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
            text-align: center; 
            background: #000; 
            color: #fff; 
            margin: 0; 
            overflow: hidden; 
        }
        #container { position: relative; display: inline-block; width: 100%; max-width: 800px; }
        video { width: 100%; border-radius: 15px; background: #222; }
        #overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            pointer-events: none; border: 5px solid transparent; transition: border 0.1s;
        }
        #status { 
            background: rgba(0,0,0,0.7); padding: 15px; font-size: 1.2rem; 
            font-weight: bold; position: fixed; bottom: 0; width: 100%;
        }
        .btn-start {
            background: #28a745; color: white; border: none; padding: 20px 40px;
            font-size: 1.5rem; border-radius: 50px; cursor: pointer; margin-top: 20px;
        }
    </style>
</head>
<body>

    <div id="container">
        <video id="webcam" autoplay playsinline muted></video>
        <div id="overlay"></div>
    </div>

    <div id="status">
        <button class="btn-start" id="startBtn">System starten</button>
    </div>

    <script>
        const video = document.getElementById('webcam');
        const status = document.getElementById('status');
        const overlay = document.getElementById('overlay');
        const startBtn = document.getElementById('startBtn');
        
        let model;
        let audioCtx;
        let nextBeepTime = 0;

        // Sound-Funktion
        function playBeep(freq, duration) {
            if (!audioCtx) return;
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.type = 'square'; // Klingt eher nach Auto-Sensor
            osc.frequency.value = freq;
            
            gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + duration);
            
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            
            osc.start();
            osc.stop(audioCtx.currentTime + duration);
        }

        async function setupCamera() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ 
                    video: { facingMode: "environment" } // Nutzt die Rückkamera falls vorhanden
                });
                video.srcObject = stream;
                return new Promise(resolve => video.onloadedmetadata = resolve);
            } catch (err) {
                alert("Kamerafehler: " + err.message);
            }
        }

        async function predictLoop() {
            if (!model) return;

            const predictions = await model.detect(video);
            let maxArea = 0;

            predictions.forEach(p => {
                // Nur Objekte beachten, die keine Personen sind (optional, hier für alles)
                const area = (p.bbox[2] * p.bbox[3]) / (video.videoWidth * video.videoHeight);
                if (area > maxArea) maxArea = area;
            });

            const now = audioCtx.currentTime;

            if (maxArea > 0.05) { // Hindernis erkannt
                if (maxArea > 0.55) {
                    // DAUERTON (Ganz nah)
                    overlay.style.borderColor = "red";
                    status.innerText = "STOPP! KRITISCH";
                    if (now > nextBeepTime) {
                        playBeep(1200, 0.2);
                        nextBeepTime = now + 0.05;
                    }
                } else {
                    // INTERVALL (Nähert sich)
                    overlay.style.borderColor = "yellow";
                    status.innerText = "Abstand verringert sich...";
                    
                    // Berechnet Intervall: Je größer die Fläche, desto kleiner die Pause
                    let interval = 0.6 - (maxArea * 0.8);
                    interval = Math.max(0.08, interval); 

                    if (now > nextBeepTime) {
                        playBeep(800, 0.1);
                        nextBeepTime = now + interval;
                    }
                }
            } else {
                overlay.style.borderColor = "transparent";
                status.innerText = "Weg frei";
            }

            requestAnimationFrame(predictLoop);
        }

        startBtn.onclick = async () => {
            startBtn.style.display = "none";
            status.innerText = "Initialisiere KI & Kamera...";
            
            // Audio-Kontext starten
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            
            // Modell laden (Wichtig: cocoSsd kleingeschrieben)
            model = await cocoSsd.load();
            
            await setupCamera();
            predictLoop();
        };
    </script>
</body>
</html>
