<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>NEON STRIKER: LO-FI GARAGE</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Orbitron', sans-serif; color: white; }
        #menu, #hud { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; align-items: center; pointer-events: none; }
        #menu { background: radial-gradient(circle, rgba(10,10,30,0.85) 0%, rgba(0,0,0,1) 100%); pointer-events: all; z-index: 10; }
        .panel { background: rgba(20, 20, 40, 0.95); padding: 30px; border: 3px solid #00f2fe; border-radius: 20px; text-align: center; min-width: 450px; box-shadow: 0 0 50px rgba(0, 242, 254, 0.4); }
        h1 { font-size: 2.5rem; color: #00f2fe; text-shadow: 0 0 20px #00f2fe; margin: 0 0 15px 0; }
        button { background: transparent; border: 2px solid #00f2fe; color: #00f2fe; padding: 12px 20px; font-family: 'Orbitron'; cursor: pointer; border-radius: 8px; pointer-events: all; transition: 0.3s; font-weight: bold; }
        button:hover { background: #00f2fe; color: #000; box-shadow: 0 0 20px #00f2fe; }
        #garage-trigger { position: absolute; bottom: 20px; left: 20px; border-color: #ff00ff; color: #ff00ff; z-index: 100; pointer-events: all; }
        #music-trigger { position: absolute; top: 20px; left: 20px; border-color: #00ff88; color: #00ff88; z-index: 100; pointer-events: all; }
        #hud { display: none; }
        #scoreboard { position: absolute; top: 20px; font-size: 48px; font-weight: 900; background: rgba(0,0,0,0.6); padding: 10px 40px; border-radius: 15px; border: 2px solid #333; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;900&display=swap" rel="stylesheet">
</head>
<body>
    <button id="music-trigger" onclick="toggleMusic()">🎵 MUSIC: OFF</button>
    <button id="garage-trigger" onclick="openGarage()">🏠 GARAGE</button>

    <div id="menu">
        <div class="panel">
            <h1>NEON STRIKER</h1>
            <p id="car-name" style="color: #ff00ff; letter-spacing: 2px;">MODEL: FENNEC</p>
            <button style="width:100%; margin-bottom:10px;" onclick="swapCar()">🔄 NEXT CAR</button>
            <div style="display:flex; gap:10px; justify-content: center;">
                <button onclick="startGame(0.04)">EASY</button>
                <button onclick="startGame(0.09)">MEDIUM</button>
                <button onclick="startGame(0.18)">PRO</button>
            </div>
            <p style="font-size: 11px; color: #777; margin-top: 15px;">STADIUM & WALLS LOADED</p>
        </div>
    </div>

    <div id="hud"><div id="scoreboard"><span id="s-blue" style="color:#00f2fe">0</span> - <span id="s-orange" style="color:#ff8c00">0</span></div></div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js", "cannon": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/+esm" } }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import * as CANNON from 'cannon';

        // --- LO-FI MUSIC ENGINE ---
        let audioCtx, isPlaying = false;
        window.toggleMusic = () => {
            if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            if (isPlaying) { audioCtx.suspend(); document.getElementById('music-trigger').innerText = "🎵 MUSIC: OFF"; }
            else { audioCtx.resume(); playLoFi(); document.getElementById('music-trigger').innerText = "🎵 MUSIC: ON"; }
            isPlaying = !isPlaying;
        };

        function playLoFi() {
            const loop = () => {
                if (!isPlaying) return;
                const osc = audioCtx.createOscillator();
                const gain = audioCtx.createGain();
                osc.type = 'triangle';
                osc.frequency.setValueAtTime(55, audioCtx.currentTime); 
                gain.gain.setValueAtTime(0.05, audioCtx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.8);
                osc.connect(gain); gain.connect(audioCtx.destination);
                osc.start(); osc.stop(audioCtx.currentTime + 0.8);
                setTimeout(loop, 1200); 
            };
            loop();
        }

        // --- GAME VARIABLES ---
        let gameState = 'MENU', botDiff = 0.08, score = { blue: 0, orange: 0 };
        let carType = 0; const carNames = ["FENNEC", "OCTANE", "DOMINUS"];
        const keys = {};

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const world = new CANNON.World();
        world.gravity.set(0, -22, 0);

        // --- STADIUM & WALLS ---
        scene.add(new THREE.GridHelper(160, 40, 0x00f2fe, 0x111122));
        function createWall(w, h, d, x, z, color) {
            const mesh = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), new THREE.MeshStandardMaterial({ color, transparent: true, opacity: 0.1 }));
            mesh.position.set(x, h/2, z); scene.add(mesh);
            const body = new CANNON.Body({ mass: 0, shape: new CANNON.Box(new CANNON.Vec3(w/2, h/2, d/2)) });
            body.position.set(x, h/2, z); world.addBody(body);
        }
        createWall(100, 25, 2, 0, 80, 0xff8c00); 
        createWall(100, 25, 2, 0, -80, 0x00f2fe);
        createWall(2, 25, 160, 50, 0, 0x333333);
        createWall(2, 25, 160, -50, 0, 0x333333);

        // --- CAR MODELS ---
        function buildCar(type, color) {
            const group = new THREE.Group();
            const mat = new THREE.MeshStandardMaterial({ color });
            if(type === 0) { // FENNEC
                const b = new THREE.Mesh(new THREE.BoxGeometry(3.2, 1.2, 4.8), mat); b.position.y = 0.6; group.add(b);
                const c = new THREE.Mesh(new THREE.BoxGeometry(2.8, 1, 2.5), mat); c.position.set(0, 1.6, -0.5); group.add(c);
            } else if(type === 1) { // OCTANE
                const b = new THREE.Mesh(new THREE.BoxGeometry(2.8, 0.8, 4.2), mat); b.position.y = 0.5; group.add(b);
                const c = new THREE.Mesh(new THREE.SphereGeometry(1.1, 8, 8), mat); c.scale.set(1, 0.7, 1.4); c.position.set(0, 1.2, -0.2); group.add(c);
            } else { // DOMINUS
                const b = new THREE.Mesh(new THREE.BoxGeometry(3.4, 0.7, 5.4), mat); b.position.y = 0.4; group.add(b);
                const c = new THREE.Mesh(new THREE.BoxGeometry(2.5, 0.5, 1.8), mat); c.position.set(0, 1.0, 0.3); group.add(c);
            }
            scene.add(group);
            return group;
        }

        let playerMesh = buildCar(0, 0x00eaff);
        let botMesh = buildCar(0, 0xff8c00);

        const playerBody = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(1.6, 1, 2.5)) });
        playerBody.position.set(0, 2, 55); playerBody.angularDamping = 0.99; world.addBody(playerBody);
        const botBody = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(1.6, 1, 2.5)) });
        botBody.position.set(0, 2, -55); world.addBody(botBody);

        const ballBody = new CANNON.Body({ mass: 2, shape: new CANNON.Sphere(2.5) });
        ballBody.position.set(0, 15, 0); ballBody.linearDamping = 0.2; world.addBody(ballBody);
        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.5), new THREE.MeshStandardMaterial({color: 0xffffff}));
        scene.add(ballMesh);

        scene.add(new THREE.AmbientLight(0xffffff, 0.9));

        // --- CONTROLS ---
        window.swapCar = () => {
            carType = (carType + 1) % 3;
            document.getElementById('car-name').innerText = "MODEL: " + carNames[carType];
            scene.remove(playerMesh);
            playerMesh = buildCar(carType, 0x00eaff);
        };
        window.openGarage = () => {
            gameState = 'MENU';
            document.getElementById('menu').style.display = 'flex';
            document.getElementById('hud').style.display = 'none';
        };
        window.startGame = (d) => {
            botDiff = d; gameState = 'PLAYING';
            document.getElementById('menu').style.display = 'none';
            document.getElementById('hud').style.display = 'block';
        };

        window.addEventListener('keydown', e => keys[e.code] = true);
        window.addEventListener('keyup', e => keys[e.code] = false);

        function update() {
            requestAnimationFrame(update);
            if(gameState === 'PLAYING') {
                world.fixedStep();
                const fwd = new THREE.Vector3(0,0,-1).applyQuaternion(playerMesh.quaternion);
                if(keys['KeyW']) playerBody.velocity.addScaledVector(1.8, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), playerBody.velocity);
                if(keys['KeyS']) playerBody.velocity.addScaledVector(-1.2, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), playerBody.velocity);
                if(keys['KeyA']) playerBody.angularVelocity.y = 26;
                if(keys['KeyD']) playerBody.angularVelocity.y = -26;

                // Simple Bot AI
                const b2b = ballBody.position.vsub(botBody.position);
                botBody.quaternion.setFromEuler(0, Math.atan2(b2b.x, b2b.z) + Math.PI, 0);
                const bf = new THREE.Vector3(0,0,-1).applyQuaternion(botMesh.quaternion);
                botBody.velocity.addScaledVector(botDiff * 30, new CANNON.Vec3(bf.x, bf.y, bf.z), botBody.velocity);

                playerMesh.position.copy(playerBody.position); playerMesh.quaternion.copy(playerBody.quaternion);
                botMesh.position.copy(botBody.position); botMesh.quaternion.copy(botBody.quaternion);
                ballMesh.position.copy(ballBody.position);

                camera.position.lerp(playerMesh.position.clone().add(new THREE.Vector3(0,10,25).applyQuaternion(playerMesh.quaternion)), 0.1);
                camera.lookAt(ballMesh.position);

                if(ballBody.position.z > 78 && Math.abs(ballBody.position.x) < 12) { score.blue++; ballBody.position.set(0,15,0); ballBody.velocity.set(0,0,0); }
                if(ballBody.position.z < -78 && Math.abs(ballBody.position.x) < 12) { score.orange++; ballBody.position.set(0,15,0); ballBody.velocity.set(0,0,0); }
                document.getElementById('s-blue').innerText = score.blue;
                document.getElementById('s-orange').innerText = score.orange;
            }
            renderer.render(scene, camera);
        }
        update();
    </script>
</body>
</html>
