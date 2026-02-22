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
        #pause-btn { 
            position: absolute; top: 10px; right: 10px; z-index: 100;
            background: rgba(255,255,255,0.1); color: white; border: 1px solid #fff;
            padding: 5px 15px; cursor: pointer; display: none; font-family: inherit;
        }

        .overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%; 
            background: rgba(0, 0, 0, 0.9); display: flex; flex-direction: column; 
            align-items: center; justify-content: center; z-index: 2000; text-align: center;
        }

        #start-screen { background: radial-gradient(circle, #1a0008 0%, #000 100%); }
        .title-glow { font-size: 3.5rem; color: #ff0055; text-shadow: 0 0 30px #ff0055; margin: 0; font-weight: 900; letter-spacing: -2px; }
        
        .result-card {
            background: rgba(20, 20, 20, 0.8);
            backdrop-filter: blur(10px);
            border: 2px solid #0ff;
            border-radius: 20px;
            padding: 30px;
            width: 85%;
            max-width: 420px;
            box-shadow: 0 0 40px rgba(0, 255, 255, 0.2);
        }

        .btn {
            padding: 15px 40px; font-size: 1.2rem; font-weight: bold; border: none; cursor: pointer; 
            border-radius: 50px; font-family: inherit; transition: 0.3s; margin: 10px 0;
            text-transform: uppercase; letter-spacing: 2px;
        }
        .btn-start { background: #ff0055; color: #fff; box-shadow: 0 5px 20px rgba(255, 0, 85, 0.4); }
        .btn-start:hover { transform: scale(1.05); box-shadow: 0 5px 30px rgba(255, 0, 85, 0.6); }
        .btn-retry { background: #fff; color: #000; width: 100%; }
        .btn-send { background: #0ff; color: #000; padding: 10px 20px; font-size: 0.9rem; border-radius: 5px; }

        .ranking-container { background: rgba(0,0,0,0.5); border-radius: 10px; margin: 20px 0; padding: 10px; height: 150px; overflow-y: auto; }
        .rank-row { display: flex; justify-content: space-between; padding: 5px 10px; border-bottom: 1px solid #333; font-size: 0.9rem; }
        .rank-name { color: #0ff; text-align: left; flex: 1; }

        .input-group { display: flex; gap: 10px; margin-top: 15px; }
        input[type="text"] { background: #111; border: 1px solid #444; color: #fff; padding: 10px; border-radius: 5px; flex: 1; font-family: inherit; }
    </style>
</head>
<body>

    <div id="game-container">
        <button id="pause-btn">STOP</button>
        <div id="ui">
            <div id="phase-ui" class="phase-display">PHASE: 1 (easy)</div>
            <div id="score-ui" class="score-display">0</div>
        </div>

        <div id="start-screen" class="overlay">
            <h1 class="title-glow">HEXAGON<br>HELL</h1>
            <p style="color: #666; margin-top: 10px; letter-spacing: 5px;">- CHAOS EDITION -</p>
            <button class="btn btn-start" id="start-btn" style="margin-top: 50px;">Begin Chaos</button>
        </div>

        <div id="pause-screen" class="overlay" style="display:none;">
            <h2 style="font-size: 3rem; color: #fff; text-shadow: 0 0 20px #fff;">PAUSED</h2>
            <button class="btn btn-retry" id="resume-btn" style="width: auto;">Resume</button>
        </div>

        <div id="result-screen" class="overlay" style="display:none;">
            <div class="result-card">
                <h2 style="color:#ff0055; margin:0; font-size: 1.8rem;">GAME OVER</h2>
                <div style="font-size: 3rem; font-weight: bold; margin: 10px 0;" id="final-score">0</div>
                
                <div class="ranking-container" id="rank-list-box">
                    <div style="padding-top: 60px; color: #444;">Loading World Ranking...</div>
                </div>

                <div id="submission-ui">
                    <div class="input-group">
                        <input type="text" id="player-name" placeholder="Name" maxlength="10">
                        <button class="btn-send" id="submit-btn">SEND</button>
                    </div>
                </div>

                <button class="btn btn-retry" style="margin-top: 20px;" onclick="location.reload()">Try Again</button>
            </div>
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

        const canvas = document.getElementById('game-canvas');
        const ctx = canvas.getContext('2d');
        const scoreUI = document.getElementById('score-ui');
        const phaseUI = document.getElementById('phase-ui');
        const pauseBtn = document.getElementById('pause-btn');
        canvas.width = 600; canvas.height = 600;

        let score = 0, angle = 0, rotationDir = 0.08, obstacles = [], phase = 1;
        let gameState = 'START'; 
        const centerX = 300, centerY = 300, orbitRadius = 90;

        if (window.location.hash === '#hisansuki') {
            score = 70;
            phase = 3;
            scoreUI.innerText = score;
            phaseUI.innerText = "PHASE: 4 (HELL)";
            phaseUI.style.color = "#ff00ff";
        }

        document.getElementById('start-btn').onclick = () => {
            document.getElementById('start-screen').style.display = 'none';
            pauseBtn.style.display = 'block';
            gameState = 'PLAYING';
        };

        pauseBtn.onclick = () => {
            if (gameState === 'PLAYING') {
                gameState = 'PAUSED';
                document.getElementById('pause-screen').style.display = 'flex';
            }
        };

        document.getElementById('resume-btn').onclick = () => {
            gameState = 'PLAYING';
            document.getElementById('pause-screen').style.display = 'none';
        };

        canvas.addEventListener('pointerdown', () => {
            if (gameState === 'PLAYING') rotationDir *= -1;
        });

        async function fetchRankings() {
            const box = document.getElementById('rank-list-box');
            try {
                const q = query(collection(db, "world_ranking"), orderBy("score", "desc"), limit(10));
                const snap = await getDocs(q);
                let html = '';
                snap.forEach(doc => {
                    const d = doc.data();
                    html += `<div class="rank-row"><span class="rank-name">${d.name}</span><strong>${Math.floor(d.score)}</strong></div>`;
                });
                box.innerHTML = html || 'No scores yet!';
            } catch (e) {
                box.innerHTML = '<div style="color:#ff0055; font-size:0.7rem;">Could not load rankings.<br>Check Firestore settings.</div>';
                console.error(e);
            }
        }

        document.getElementById('submit-btn').onclick = async () => {
            const nameInput = document.getElementById('player-name');
            const name = nameInput.value.trim() || "Anonymous";
            const btn = document.getElementById('submit-btn');
            btn.disabled = true;
            btn.innerText = "WAIT";

            try {
                await addDoc(collection(db, "world_ranking"), {
                    name: name,
                    score: Math.floor(score),
                    createdAt: serverTimestamp()
                });
                document.getElementById('submission-ui').innerHTML = "<div style='color:#0ff; margin-top:10px;'>SCORE SYNCED!</div>";
                fetchRankings();
            } catch (e) {
                alert("Error sending score.");
                btn.disabled = false;
                btn.innerText = "SEND";
            }
        };

        function createEnemy() {
            const side = Math.floor(Math.random() * 4);
            let x, y, vx, vy;
            let speed = (1.5 + Math.random() * 2) + (phase * 0.5);
            if (side === 0) { x = -20; y = Math.random()*600; vx = speed; vy = 0; }
            else if (side === 1) { x = 620; y = Math.random()*600; vx = -speed; vy = 0; }
            else if (side === 2) { x = Math.random()*600; y = -20; vx = 0; vy = speed; }
            else { x = Math.random()*600; y = 620; vx = 0; vy = -speed; }
            obstacles.push({ x, y, vx, vy, color: '#ff0055' });
        }

        function update() {
            if (gameState === 'PLAYING') {
                angle += rotationDir;
                if (score > 20 && phase === 1) { phase = 2; phaseUI.innerText = "PHASE: 2 (normal)"; }
                if (score > 50 && phase === 2) { phase = 3; phaseUI.innerText = "PHASE: 3 (hard)"; }
                if (score > 100 && phase === 3) { phase = 4; phaseUI.innerText = "PHASE: 4 (HELL)"; phaseUI.style.color = "#ff00ff"; }

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
                        document.getElementById('final-score').innerText = Math.floor(score);
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

            ctx.strokeStyle = '#333';
            ctx.setLineDash([5, 5]);
            ctx.beginPath(); ctx.arc(centerX, centerY, orbitRadius, 0, Math.PI*2); ctx.stroke();

            obstacles.forEach(ob => {
                ctx.fillStyle = ob.color;
                ctx.fillRect(ob.x - 12, ob.y - 12, 24, 24);
            });

            ctx.fillStyle = "#fff";
            ctx.shadowBlur = 15; ctx.shadowColor = "#fff";
            ctx.beginPath(); ctx.arc(px, py, 12, 0, Math.PI*2); ctx.fill();
            ctx.shadowBlur = 0;
        }

        update();
    </script>
</body>
</html>
