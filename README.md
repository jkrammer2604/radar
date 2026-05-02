document.getElementById('btBtn').onclick = async () => {
    try {
        const device = await navigator.bluetooth.requestDevice({
            filters: [{ namePrefix: 'BBC micro:bit' }, { namePrefix: 'Calliope' }],
            optionalServices: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e'] 
        });

        statusMsg.innerText = "Verbinde...";
        const server = await device.gatt.connect();
        
        // Kurze Pause, damit das iPad den Dienst registrieren kann
        await new Promise(resolve => setTimeout(resolve, 500));

        const service = await server.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e');
        const char = await service.getCharacteristic('6e400001-b5a3-f393-e0a9-e50e24dcca9e');

        await char.startNotifications();
        document.getElementById('btBtn').innerText = "VERBUNDEN";
        document.getElementById('btBtn').style.background = "#28a745";

        char.addEventListener('characteristicvaluechanged', (e) => {
            let str = new TextDecoder().decode(e.target.value);
            // Deine Datenverarbeitung...
            console.log(str);
        });
    } catch (err) { 
        alert("BT Fehler: " + err.message + " (Code: " + err.code + ")"); 
    }
};
