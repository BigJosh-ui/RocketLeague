<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>NEON STRIKER: ULTIMATE</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Orbitron', sans-serif; color: white; }
        #menu, #hud { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; align-items: center; pointer-events: none; }
        #menu { background: radial-gradient(circle, rgba(10,10,30,0.95) 0%, rgba(0,0,0,1) 100%); pointer-events: all; z-index: 10; }
        .panel { background: rgba(20, 20, 40, 0.9); padding: 40px; border: 3px solid #00f2fe; border-radius: 20px; text-align: center; min-width: 450px; box-shadow: 0 0 50px rgba(0, 242, 254, 0.3); }
        h1 { font-size: 3rem; color: #00f2fe; text-shadow: 0 0 20px #00f2fe; margin-bottom: 20px; }
        button { background: transparent; border: 2px solid #00f2fe; color: #00f2fe; padding: 15px 25px; font-family: 'Orbitron'; cursor: pointer; border-radius: 8px; pointer-events: all; transition: 0.3s; font-weight: 900; letter-spacing: 2px; }
        button:hover { background: #00f2fe; color: #000; box-shadow: 0 0 30px #00f2fe; }
        #garage-trigger { position: absolute; bottom: 30px; left: 30px; border-color: #ff00ff; color: #ff00ff; z-index: 100; pointer-events: all; }
        #music-trigger { position: absolute; top: 30px; left: 30px; border-color: #00ff88; color: #00ff88; z-index: 100; pointer-events: all; }
        #scoreboard { position: absolute; top: 20px; font-size: 40px; background: rgba(0,0,0,0.7); padding: 10px 50px; border-radius: 15px; border: 2px solid #333; display:none; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;900&display=swap" rel="stylesheet">
</head>
<body>
    <button id="music-trigger" onclick="toggleMusic()">🎵 LO-FI: OFF</button>
    <button id="garage-trigger" onclick="openGarage()">🏠 GARAGE</button>

    <div id="menu">
        <div class="panel">
            <h1>NEON STRIKER</h1>
            <p id="car-name" style="color: #ff00ff; font-size: 24px;">MODEL: FENNEC</p>
            <button style="width:100%; margin-bottom:15px; border-color:#ff00ff; color:#ff00ff;" onclick="swapCar()">🔄 SWAP CAR</button>
            <button onclick="startGame()">START MATCH</button>
        </div>
    </div>

    <div id="hud"><div id="scoreboard"><span id="s-blue" style="color:#00f2fe">0</span> - <span id="s-orange" style="color:#ff8c00">0</span></div></div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js", "cannon": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/+esm" } }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import * as CANNON from 'cannon';

        let audioCtx, isPlaying = false, gameState = 'MENU', score = { blue: 0, orange: 0 }, carType = 0, particles = [], shake = 0;
        const carNames = ["FENNEC", "OCTANE", "DOMINUS"], keys = {};

        // --- MUSIC ---
        window.toggleMusic = () => {
            if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            if (isPlaying) { audioCtx.suspend(); document.getElementById('music-trigger').innerText = "🎵 LO-FI: OFF"; }
            else { audioCtx.resume(); playLoFi(); document.getElementById('music-trigger').innerText = "🎵 LO-FI: ON"; }
            isPlaying = !isPlaying;
        };
        function playLoFi() {
            const loop = () => { if (!isPlaying) return;
                const osc = audioCtx.createOscillator(); const gain = audioCtx.createGain();
                osc.type = 'triangle'; osc.frequency.setValueAtTime(55, audioCtx.currentTime); 
                gain.gain.setValueAtTime(0.05, audioCtx.currentTime); gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 1.2);
                osc.connect(gain); gain.connect(audioCtx.destination); osc.start(); osc.stop(audioCtx.currentTime + 1.2);
                setTimeout(loop, 1500); }; loop();
        }

        // --- ENGINE ---
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight); document.body.appendChild(renderer.domElement);

        const world = new CANNON.World(); world.gravity.set(0, -25, 0);

        // --- STADIUM ---
        const floor = new THREE.Mesh(new THREE.BoxGeometry(100, 2, 160), new THREE.MeshStandardMaterial({ color: 0x050510 }));
        floor.position.y = -1; scene.add(floor);
        scene.add(new THREE.GridHelper(160, 40, 0x00f2fe, 0x002222));
        world.addBody(new CANNON.Body({ mass: 0, shape: new CANNON.Box(new CANNON.Vec3(50, 1, 80)) }));

        function createWall(w, h, d, x, z, color) {
            const m = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), new THREE.MeshStandardMaterial({ color, transparent: true, opacity: 0.1 }));
            m.position.set(x, h/2, z); scene.add(m);
            const b = new CANNON.Body({ mass: 0, shape: new CANNON.Box(new CANNON.Vec3(w/2, h/2, d/2)) });
            b.position.set(x, h/2, z); world.addBody(b);
        }
        createWall(100, 25, 2, 0, 80, 0xff8c00); createWall(100, 25, 2, 0, -80, 0x00f2fe);
        createWall(2, 25, 160, 50, 0, 0x333); createWall(2, 25, 160, -50, 0, 0x333);

        // --- CAR BUILDER ---
        function buildCar(type) {
            const group = new THREE.Group(); const mat = new THREE.MeshStandardMaterial({ color: 0x00f2fe });
            if(type === 0) { // Fennec
                const b = new THREE.Mesh(new THREE.BoxGeometry(3.2, 1.4, 4.8), mat); b.position.y = 0.7; group.add(b);
                const c = new THREE.Mesh(new THREE.BoxGeometry(2.6, 1, 2.5), mat); c.position.set(0, 1.7, -0.5); group.add(c);
            } else if(type === 1) { // Octane
                const b = new THREE.Mesh(new THREE.BoxGeometry(2.8, 0.8, 4.2), mat); b.position.y = 0.4; group.add(b);
                const c = new THREE.Mesh(new THREE.SphereGeometry(1.2, 8, 8), mat); c.scale.set(1, 0.7, 1.6); c.position.set(0, 1.1, 0); group.add(c);
            } else { // Dominus
                const b = new THREE.Mesh(new THREE.BoxGeometry(3.6, 0.7, 5.8), mat); b.position.y = 0.35; group.add(b);
                const c = new THREE.Mesh(new THREE.BoxGeometry(2.4, 0.5, 1.8), mat); c.position.set(0, 0.9, 0.6); group.add(c);
            }
            scene.add(group); return group;
        }

        let playerMesh = buildCar(0);
        const playerBody = new CANNON.Body({ mass: 60, shape: new CANNON.Box(new CANNON.Vec3(1.8, 0.8, 2.8)) });
        playerBody.position.set(0, 3, 50); playerBody.angularDamping = 0.95; world.addBody(playerBody);

        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.8, 32), new THREE.MeshStandardMaterial({color: 0xffffff}));
        scene.add(ballMesh);
        const ballBody = new CANNON.Body({ mass: 8, shape: new CANNON.Sphere(2.8) });
        ballBody.position.set(0, 10, 0); world.addBody(ballBody);

        scene.add(new THREE.AmbientLight(0xffffff, 1));

        window.swapCar = () => { carType = (carType + 1) % 3; document.getElementById('car-name').innerText = "MODEL: " + carNames[carType]; scene.remove(playerMesh); playerMesh = buildCar(carType); };
        window.openGarage = () => { gameState = 'MENU'; document.getElementById('menu').style.display = 'flex'; document.getElementById('scoreboard').style.display = 'none'; };
        window.startGame = () => { gameState = 'PLAYING'; document.getElementById('menu').style.display = 'none'; document.getElementById('scoreboard').style.display = 'block'; };

        window.addEventListener('keydown', e => keys[e.code] = true);
        window.addEventListener('keyup', e => keys[e.code] = false);

        function update() {
            requestAnimationFrame(update);
            if(gameState === 'PLAYING') {
                world.fixedStep();
                const fwd = new THREE.Vector3(0,0,-1).applyQuaternion(playerMesh.quaternion);
                if(keys['KeyW']) { playerBody.velocity.x += fwd.x * 2.2; playerBody.velocity.z += fwd.z * 2.2; }
                if(keys['KeyS']) { playerBody.velocity.x -= fwd.x * 1.5; playerBody.velocity.z -= fwd.z * 1.5; }
                if(keys['KeyA']) playerBody.angularVelocity.y = 7;
                else if(keys['KeyD']) playerBody.angularVelocity.y = -7;
                else playerBody.angularVelocity.y *= 0.7;

                playerMesh.position.copy(playerBody.position); playerMesh.quaternion.copy(playerBody.quaternion);
                ballMesh.position.copy(ballBody.position);

                const camPos = playerMesh.position.clone().add(new THREE.Vector3(0, 12, 28).applyQuaternion(playerMesh.quaternion));
                camera.position.lerp(camPos, 0.1); if(shake > 0) { camera.position.x += (Math.random()-0.5)*shake; shake *= 0.9; }
                camera.lookAt(ballMesh.position);

                if(ballBody.position.z > 78 && Math.abs(ballBody.position.x) < 15) { score.blue++; ballBody.position.set(0,15,0); ballBody.velocity.set(0,0,0); shake = 2; }
                if(ballBody.position.z < -78 && Math.abs(ballBody.position.x) < 15) { score.orange++; ballBody.position.set(0,15,0); ballBody.velocity.set(0,0,0); shake = 2; }
                document.getElementById('s-blue').innerText = score.blue; document.getElementById('s-orange').innerText = score.orange;
            }
            renderer.render(scene, camera);
        }
        update();
    </script>
</body>
</html>
