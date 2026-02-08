<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>NEON STRIKER: LO-FI EDITION</title>
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
            <p id="car-name" style="color: #ff00ff;">MODEL: FENNEC</p>
            <button style="width:100%; margin-bottom:10px;" onclick="swapCar()">🔄 NEXT CAR</button>
            <div style="display:flex; gap:10px;">
                <button onclick="startGame(0.04)">EASY</button>
                <button onclick="startGame(0.09)">MEDIUM</button>
                <button onclick="startGame(0.18)">PRO</button>
            </div>
        </div>
    </div>

    <div id="hud"><div id="scoreboard"><span id="s-blue" style="color:#00f2fe">0</span> - <span id="s-orange" style="color:#ff8c00">0</span></div></div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js", "cannon": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/+esm" } }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import * as CANNON from 'cannon';

        // --- AUDIO ENGINE (Lo-Fi) ---
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
                osc.frequency.setValueAtTime(55, audioCtx.currentTime); // Low Chill Bass
                gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.5);
                osc.connect(gain); gain.connect(audioCtx.destination);
                osc.start(); osc.stop(audioCtx.currentTime + 0.5);
                setTimeout(loop, 1000); // 60 BPM
            };
            loop();
        }

        // --- GAME LOGIC ---
        let gameState = 'MENU', botDifficulty = 0.08, score = { blue: 0, orange: 0 };
        let carType = 0; const carNames = ["FENNEC", "OCTANE", "DOMINUS"];
        const keys = {};

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const world = new CANNON.World();
        world.gravity.set(0, -22, 0);

        // Arena & Walls
        scene.add(new THREE.GridHelper(160, 40, 0x00f2fe, 0x111122));
        function createWall(w, h, d, x, z, color) {
            const mesh = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), new THREE.MeshStandardMaterial({ color, transparent: true, opacity: 0.1 }));
            mesh.position.set(x, h/2, z); scene.add(mesh);
            const body = new CANNON.Body({ mass: 0, shape: new CANNON.Box(new CANNON.Vec3(w/2, h/2, d/2)) });
            body.position.set(x, h/2, z); world.addBody(body);
        }
        createWall(100, 20, 2, 0, 80, 0xff8c00); createWall(100, 20, 2, 0, -80, 0x00f2fe);
        createWall(2, 20, 160, 50, 0, 0x333333); createWall(2, 20, 160, -50, 0, 0x333333);

        function buildCar(type, color) {
            const group = new THREE.Group();
            const mat = new THREE.MeshStandardMaterial({ color });
            const b = new THREE.Mesh(new THREE.BoxGeometry(3, 1, 4.5), mat);
            b.position.y = 0.5; group.add(b);
            const wheels = [[1.5, 0.4, 1.5], [-1.5, 0.4, 1.5], [1.5, 0.4, -1.5], [-1.5, 0.4, -1.5]].map(p => {
                const w = new THREE.Mesh(new THREE.CylinderGeometry(0.5, 0.5, 0.4), new THREE.MeshStandardMaterial({color:0x111}));
                w.rotateZ(Math.PI/2); w.position.set(...p); group.add(w); return w;
            });
            return { group, wheels };
        }

        let player = buildCar(0, 0x00eaff); scene.add(player.group);
        let bot = buildCar(0, 0xff8c00); scene.add(bot.group);

        const playerBody = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(1.5, 0.8, 2.3)) });
        playerBody.position.set(0, 2, 50); playerBody.angularDamping = 0.99; world.addBody(playerBody);
        const botBody = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(1.5, 0.8, 2.3)) });
        botBody.position.set(0, 2, -50); world.addBody(botBody);

        const ballBody = new CANNON.Body({ mass: 2, shape: new CANNON.Sphere(2.5) });
        ballBody.position.set(0, 10, 0); world.addBody(ballBody);
        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.5), new THREE.MeshStandardMaterial({color:0xfff}));
        scene.add(ballMesh);

        scene.add(new THREE.AmbientLight(0xffffff, 0.8));

        window.swapCar = () => {
            carType = (carType + 1) % 3;
            document.getElementById('car-name').innerText = "MODEL: " + carNames[carType];
        };

        window.openGarage = () => {
            gameState = 'MENU';
            document.getElementById('menu').style.display = 'flex';
            document.getElementById('hud').style.display = 'none';
        };

        window.startGame = (diff) => {
            botDifficulty = diff; gameState = 'PLAYING';
            document.getElementById('menu').
