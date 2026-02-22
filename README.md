<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>HEXAGON HELL -CHAOS-</title>
    <style>
        * { touch-action: none; -webkit-tap-highlight-color: transparent; outline: none; box-sizing: border-box; }
        body { margin: 0; background: #000; color: #fff; font-family: 'Courier New', monospace; overflow: hidden; display: flex; align-items: center; justify-content: center; height: 100vh; width: 100%; position: fixed; }
        
        #game-container { position: relative; width: 600px; height: 600px; max-width: 95vw; max-height: 85vh; }
        canvas { background: #000; border: 4px solid #333; width: 100%; height: 100%; display: block; }
        
        #ui { position: absolute; top: 20px; text-align: center; pointer-events: none; width: 100%; z-index: 10; }
        .score-display { font-size: 4rem; font-weight: bold; text-shadow: 0 0 20px #ff0055; margin: 0; }
        .phase-display { font-size: 1.2rem; color: #ff0055; font-weight: bold; text-transform: uppercase; }

        /* ボタン類 */
        #pause-trigger { 
            position: absolute; top: 10px; right: 10px; z-index: 100;
            background: rgba(255,255,255,0.1); color: white; border: 1px solid #fff;
            padding: 8px 15px; cursor: pointer; display: none; font-family: inherit;
        }

        /* オーバーレイ（開始・リザルト・一時停止） */
        .overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%; 
            background: rgba(0, 0, 0, 0.9); display: flex; flex-direction: column; 
            align-items: center; justify-content: center; z-index: 2000; text-align: center;
        }

        .card { background: #050505; padding: 25px; border: 2px solid #0ff; border-radius: 10px; width: 85%; max-height: 90%; overflow-y: auto; }
        
        /* ランキング */
        #ranking-area { margin: 15px 0; padding: 10px; background: #111; border-radius: 5px; }
        .rank-list { list-style: none; padding: 0; margin: 10px 0; font-size: 0.85rem; }
        .rank-item { display: flex; justify-content: space-between; padding: 4px 0; border-bottom: 1px solid #222; }
        .rank-name { color: #0ff; text-align: left; flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

        input[type="text"] { background: #000; border: 1px solid #0ff; color: #fff; padding: 10px; width: 100%; margin: 10px 0; font-family: inherit; }
        .btn { padding: 15px; font-size: 1rem; font-weight: bold; border: none; cursor: pointer; border-radius: 5px; font-family: inherit; width: 100%; margin-top: 5px; }
        .primary { background: #fff; color: #000; }
        .secondary { background: #000; color: #fff; border: 1px solid #fff; }
    </style>
</head>
<body>

    <div id="game-container">
        <button id="pause-trigger">PAUSE</button>

        <div id="start-screen" class="overlay">
            <h1 style="color:#ff0055; font-size: 2.5rem; margin-bottom: 0;">HEXAGON HELL</h1>
            <p style="color:#0ff; margin-bottom: 30px;">- CHAOS EDITION -</p>
            <button class="btn primary" id="start-btn">START GAME</button>
        </div>

        <div id="pause-screen" class="overlay" style="display:none;">
            <h2 style="color:#fff;">PAUSED</h2>
            <button class="btn primary" id="resume-btn">RESUME</button>
        </div>

        <div id="result-screen" class="overlay" style="display:none;">
            <div class="card">
                <h2 style="color:#0ff; margin:0;">GAME OVER</h2>
                <div style="font-size: 1.2rem; margin: 10px 0;">SCORE: <span id="current-score" style="color:#ff0055; font-weight:bold;">0</span></div>
                
                <div id="ranking-area">
                    <div style="font-size: 0.9rem; color: #aaa;">WORLD TOP 10</div>
                    <div id="rank-list-container" class="rank-list">Loading rankings...</div>
                </div>

                <div id="submission-form">
                    <input type="text" id="player-name" placeholder="ENTER YOUR NAME" maxlength="12">
                    <button class="btn primary" id="submit-btn">SEND TO RANKING</button>
                </div>

                <button class="btn secondary" style="margin-top:15px;" onclick="location.reload()">RETRY</button>
            </div>
        </div>

        <div id="ui">
            <div id="phase-ui" class="phase-display">PHASE: 1 (easy)</div>
            <div id="score-ui" class="score-display">0</div>
        </div>
        <canvas id="game-canvas"></canvas>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, addDoc, query, orderBy, limit, getDocs, serverTimestamp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

        const firebaseConfig = {
            apiKey: "AIzaSyCwhHspaG94goiCIjVj3h-Un5pBK3JTjMU",
            authDomain: "soulkin-aa3b7.firebaseapp.com",
            databaseURL: "https://soulkin-aa3b7-default-rtdb.firebaseio.com",
            projectId: "soulkin-aa3b7",
            storageBucket: "soulkin-aa3b7.firebasestorage.app",
            messagingSenderId: "358331064206",
            appId: "1:358331064206:web:d7760ea0919259418a4edf",
            measurementId: "G-S5Z8TTWYE2"
        };

        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);

        // --- Game Logic ---
        const canvas = document.getElementById('game-canvas');
        const ctx = canvas.getContext('2d');
        const scoreUI = document.getElementById('score-ui');
        const phaseUI = document.getElementById('phase-ui');
        const pauseBtn = document.getElementById('pause-trigger');
        
        canvas.width = 600; canvas.height = 600;

        let score = 0, angle = 0, rotationDir = 0.08, obstacles = [], phase = 1, frameCount = 0;
        let gameState = 'START'; // START, PLAYING, PAUSED, GAMEOVER
        const centerX = 300, centerY = 300, orbitRadius = 90;

        // --- UI Interactions ---
        document.getElementById('start-btn').onclick = () => {
            document.getElementById('start-screen').style.display = 'none';
            pauseBtn.style.display = 'block';
            gameState = 'PLAYING';
        };

        pauseBtn.onclick = () => {
            if (gameState === 'PLAYING') {
                gameState = 'PAUSED';
                document.getElementById('pause-screen').style.display = 'flex';
                pauseBtn.innerText = 'RESUME';
            } else if (gameState === 'PAUSED') {
                resumeGame();
            }
        };

        document.getElementById('resume-btn').onclick = resumeGame;

        function resumeGame() {
            gameState = 'PLAYING';
            document.getElementById('pause-screen').style.display = 'none';
            pauseBtn.innerText = 'PAUSE';
        }

        canvas.addEventListener('pointerdown', (e) => {
            if (gameState === 'PLAYING') rotationDir *= -1;
        });

        // --- Ranking Logic ---
        async function fetchRankings() {
            const container = document.getElementById('rank-list-container');
            try {
                const q = query(collection(db, "world_ranking"), orderBy("score", "desc"), limit(10));
                const querySnapshot = await getDocs(q);
                let html = '';
                querySnapshot.forEach((doc) => {
                    const data = doc.data();
                    html += `<div class="rank-item"><span class="rank-name">${data.name || 'Anonymous'}</span><span style="font-weight:bold;">${Math.floor(data.score)}</span></div>`;
                });
                container.innerHTML = html || 'No records yet.';
            } catch (e) {
                container.innerHTML = 'Failed to load.';
                console.error(e);
            }
        }

        document.getElementById('submit-btn').onclick = async () => {
            const nameInput = document.getElementById('player-name');
            const name = nameInput.value.trim() || "NONAME";
            const submitBtn = document.getElementById('submit-btn');
            
            submitBtn.disabled = true;
            submitBtn.innerText = "SENDING...";

            try {
                await addDoc(collection(db, "world_ranking"), {
                    name: name,
                    score: Math.floor(score),
                    createdAt: serverTimestamp()
                });
                document.getElementById('submission-form').innerHTML = "<p style='color:#0ff;'>SCORE SUBMITTED!</p>";
                fetchRankings();
            } catch (e) {
                alert("Error submitting score.");
                submitBtn.disabled = false;
                submitBtn.innerText = "RETRY SEND";
            }
        };

        // --- Core Game Functions ---
        function createEnemy() {
            const side = Math.floor(Math.random() * 4);
            let x, y, vx, vy;
            let baseSpeed = phase === 1 ? 1.5 : phase === 2 ? 2.5 : 3.5;
            let speed = (baseSpeed + Math.random() * 1.5) + (phase * 0.4);
            if (side === 0) { x = -20; y = Math.random()*600; vx = speed; vy = 0; }
            else if (side === 1) { x = 620; y = Math.random()*600; vx = -speed; vy = 0; }
            else if (side === 2) { x = Math.random()*600; y = -20; vx = 0; vy = speed; }
            else { x = Math.random()*600; y = 620; vx = 0; vy = -speed; }
            obstacles.push({ x, y, vx, vy, color: '#ff0055' });
        }

        function update() {
            if (gameState === 'PLAYING') {
                frameCount++;
                angle += rotationDir;

                // Phase logic
                if (score > 15 && phase === 1) { phase = 2; phaseUI.innerText = "PHASE: 2 (normal)"; phaseUI.style.color = "#ffcc00"; }
                if (score > 40 && phase === 2) { phase = 3; phaseUI.innerText = "PHASE: 3 (hard)"; phaseUI.style.color = "#ff0000"; }

                if (Math.random() < 0.03 + (phase * 0.01)) createEnemy();

                const px = centerX + Math.cos(angle) * orbitRadius;
                const py = centerY + Math.sin(angle) * orbitRadius;

                for (let i = obstacles.length - 1; i >= 0; i--) {
                    let ob = obstacles[i];
                    ob.x += ob.vx; ob.y += ob.vy;
                    const dist = Math.hypot(px - ob.x, py - ob.y);
                    
                    if (dist < 22) {
                        gameState = 'GAMEOVER';
                        pauseBtn.style.display = 'none';
                        document.getElementById('result-screen').style.display = 'flex';
                        document.getElementById('current-score').innerText = Math.floor(score);
                        fetchRankings();
                        return;
                    }

                    if (ob.x < -100 || ob.x > 700 || ob.y < -100 || ob.y > 700) {
                        obstacles.splice(i, 1);
                        score += 1;
                        scoreUI.innerText = Math.floor(score);
                    }
                }
                draw(px, py);
            }
            requestAnimationFrame(update);
        }

        function draw(px, py) {
            ctx.fillStyle = 'rgba(0,0,0,0.25)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Orbit line
            ctx.strokeStyle = '#222';
            ctx.setLineDash([5, 5]);
            ctx.beginPath(); ctx.arc(centerX, centerY, orbitRadius, 0, Math.PI*2); ctx.stroke();

            // Enemies
            obstacles.forEach(ob => {
                ctx.fillStyle = ob.color;
                ctx.shadowBlur = phase > 1 ? 10 : 0;
                ctx.shadowColor = ob.color;
                ctx.fillRect(ob.x - 12, ob.y - 12, 24, 24);
            });

            // Player
            ctx.fillStyle = "#fff";
            ctx.shadowBlur = 20; ctx.shadowColor = "#fff";
            ctx.beginPath(); ctx.arc(px, py, 12, 0, Math.PI*2); ctx.fill();
            ctx.shadowBlur = 0;
        }

        update();
    </script>
</body>
