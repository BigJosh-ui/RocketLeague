<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>ROCKET LEAGUE 2.0 - BOT MODE</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Orbitron', sans-serif; color: white; }
        #gui, #menu, #settings-menu { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; align-items: center; pointer-events: none; }
        
        #menu { background: radial-gradient(circle, rgba(10,10,30,0.95) 0%, rgba(0,0,0,1) 100%); pointer-events: all; z-index: 10; }
        .panel { background: rgba(20, 20, 40, 0.9); padding: 40px; border: 3px solid #00f2fe; border-radius: 20px; text-align: center; min-width: 400px; box-shadow: 0 0 50px rgba(0, 242, 254, 0.4); }
        h1 { font-size: 42px; margin-bottom: 10px; color: #00f2fe; text-shadow: 0 0 20px #00f2fe; }
        
        button { background: transparent; border: 2px solid #00f2fe; color: #00f2fe; padding: 12px 25px; font-family: 'Orbitron'; cursor: pointer; transition: 0.3s; font-weight: bold; border-radius: 8px; margin: 5px; }
        button:hover { background: #00f2fe; color: #000; box-shadow: 0 0 20px #00f2fe; }
        button.active { background: #00f2fe; color: #000; }

        /* HUD & Settings */
        #hud { position: absolute; inset: 0; pointer-events: none; display: none; }
        #scoreboard { position: absolute; top: 30px; left: 50%; transform: translateX(-50%); font-size: 36px; }
        #settings-icon { position: absolute; bottom: 20px; right: 20px; font-size: 30px; cursor: pointer; pointer-events: all; background: rgba(255,255,255,0.1); padding: 10px; border-radius: 50%; border: 1px solid cyan; }
        
        #settings-menu { display: none; background: rgba(0,0,0,0.8); pointer-events: all; z-index: 20; }
        .settings-panel { background: #111; border: 2px solid #ff00ff; padding: 30px; border-radius: 15px; text-align: center; }

        #goal-flash { position: absolute; inset: 0; background: white; opacity: 0; pointer-events: none; transition: opacity 0.1s; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;900&display=swap" rel="stylesheet">
</head>
<body>

    <div id="goal-flash"></div>

    <div id="menu">
        <div class="panel">
            <h1>ROCKET LEAGUE 2.0</h1>
            <button onclick="startGame()">ENTER ARENA</button>
        </div>
    </div>

    <div id="hud">
        <div id="scoreboard"><span id="s-blue">0</span> - <span id="s-orange">0</span></div>
        <div id="settings-icon" onclick="toggleSettings()">⚙️</div>
    </div>

    <div id="settings-menu">
        <div class="settings-panel">
            <h2 style="color: #ff00ff;">MATCH SETTINGS</h2>
            <button id="bot-toggle" onclick="toggleBot()">BOT: ON</button>
            <hr style="border: 0.5px solid #333; margin: 20px 0;">
            <p>DIFFICULTY</p>
            <button onclick="setDiff(0.02)">EASY</button>
            <button onclick="setDiff(0.05)">MEDIUM</button>
            <button onclick="setDiff(0.09)">HARD</button>
            <br><br>
            <button style="border-color: #555; color: #555;" onclick="toggleSettings()">BACK TO GAME</button>
        </div>
    </div>

    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "cannon": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/+esm"
            }
        }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import * as CANNON from 'cannon';

        let gameState = 'MENU';
        let botActive = true;
        let botDifficulty = 0.05; 
        let score = { blue: 0, orange: 0 };
        const keys = {};

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const world = new CANNON.World();
        world.gravity.set(0, -25, 0);

        // Arena Visuals
        scene.add(new THREE.GridHelper(400, 80, 0x00f2fe, 0x050515));
        const groundBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
        groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
        world.addBody(groundBody);

        // Ball
        const ballBody = new CANNON.Body({ mass: 2, shape: new CANNON.Sphere(2.5) });
        ballBody.position.set(0, 5, 0);
        world.addBody(ballBody);
        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.5, 32), new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0x00f2fe }));
        scene.add(ballMesh);

        // Cars
        function createCar(color, posZ) {
            const body = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(1.6, 1.1, 2.4)) });
            body.position.set(0, 2, posZ);
            world.addBody(body);
            const mesh = new THREE.Mesh(new THREE.BoxGeometry(3.2, 2.2, 4.8), new THREE.MeshStandardMaterial({ color }));
            scene.add(mesh);
            return { body, mesh };
        }

        const player = createCar(0x00eaff, 30);
        const bot = createCar(0xff8c00, -30);

        scene.add(new THREE.AmbientLight(0xffffff, 0.6));
        const sun = new THREE.DirectionalLight(0xffffff, 1);
        sun.position.set(10, 50, 10);
        scene.add(sun);

        window.startGame = () => {
            gameState = 'PLAYING';
            document.getElementById('menu').style.display = 'none';
            document.getElementById('hud').style.display = 'block';
        };

        window.toggleSettings = () => {
            const m = document.getElementById('settings-menu');
            m.style.display = m.style.display === 'flex' ? 'none' : 'flex';
        };

        window.toggleBot = () => {
            botActive = !botActive;
            document.getElementById('bot-toggle').innerText = botActive ? 'BOT: ON' : 'BOT: OFF';
            if(!botActive) bot.body.position.set(100, -10, 100); // Hide bot
            else bot.body.position.set(0, 2, -30);
        };

        window.setDiff = (d) => { botDifficulty = d; toggleSettings(); };

        window.addEventListener('keydown', e => keys[e.code] = true);
        window.addEventListener('keyup', e => keys[e.code] = false);

        function update() {
            requestAnimationFrame(update);
            if(gameState === 'PLAYING') {
                world.fixedStep();

                // Player Logic
                const fwd = new THREE.Vector3(0,0,-1).applyQuaternion(player.mesh.quaternion);
                if(keys['KeyW']) player.body.velocity.addScaledVector(0.8, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), player.body.velocity);
                if(keys['KeyS']) player.body.velocity.addScaledVector(-0.4, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), player.body.velocity);
                if(keys['KeyA']) player.body.angularVelocity.y = 4;
                if(keys['KeyD']) player.body.angularVelocity.y = -4;
                if(keys['KeyK'] && Math.abs(player.body.velocity.y) < 0.1) player.body.velocity.y = 13;

                // BOT AI LOGIC (Goal-Hungry)
                if(botActive) {
                    const botToBall = new THREE.Vector3().subVectors(ballMesh.position, bot.mesh.position);
                    const angle = Math.atan2(botToBall.x, botToBall.z);
                    bot.body.quaternion.setFromEuler(0, angle + Math.PI, 0);
                    
                    const bFwd = new THREE.Vector3(0,0,-1).applyQuaternion(bot.mesh.quaternion);
                    bot.body.velocity.addScaledVector(botDifficulty * 22, new CANNON.Vec3(bFwd.x, bFwd.y, bFwd.z), bot.body.velocity);
                    
                    // Bot Jump
                    if(ballBody.position.y > 5 && botToBall.length() < 10) bot.body.velocity.y = 12;
                    // Bot Boost
                    if(botToBall.length() > 25) bot.body.velocity.addScaledVector(0.6, new CANNON.Vec3(bFwd.x, bFwd.y, bFwd.z), bot.body.velocity);
                }

                // Sync Visuals
                player.mesh.position.copy(player.body.position);
                player.mesh.quaternion.copy(player.body.quaternion);
                bot.mesh.position.copy(bot.body.position);
                bot.mesh.quaternion.copy(bot.body.quaternion);
                ballMesh.position.copy(ballBody.position);

                // Scoring
                if(ballBody.position.z < -60) { score.blue++; resetMatch(); }
                if(ballBody.position.z > 60) { score.orange++; resetMatch(); }
                document.getElementById('s-blue').innerText = score.blue;
                document.getElementById('s-orange').innerText = score.orange;

                camera.position.lerp(player.mesh.position.clone().add(new THREE.Vector3(0,8,18).applyQuaternion(player.mesh.quaternion)), 0.1);
                camera.lookAt(ballMesh.position);
            }
            renderer.render(scene, camera);
        }

        function resetMatch() {
            ballBody.position.set(0,10,0);
            ballBody.velocity.set(0,0,0);
            player.body.position.set(0,2,30);
            if(botActive) bot.body.position.set(0,2,-30);
        }

        update();
    </script>
</body>
</html>
