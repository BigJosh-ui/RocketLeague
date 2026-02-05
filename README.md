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
        .settings-panel { background: #111; border: 2px solid #ff00ff; padding: 20px; border-radius: 15px; }

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
            <h3>BOT SETTINGS</h3>
            <button id="bot-toggle" onclick="botActive = !botActive; this.innerText = botActive ? 'BOT: ON' : 'BOT: OFF'">BOT: ON</button>
            <hr>
            <button onclick="botDifficulty = 0.02">EASY</button>
            <button onclick="botDifficulty = 0.05">MEDIUM</button>
            <button onclick="botDifficulty = 0.08">HARD</button>
            <br><br>
            <button style="border-color: red; color: red;" onclick="toggleSettings()">CLOSE</button>
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
        let botDifficulty = 0.05; // Medium
        let score = { blue: 0, orange: 0
