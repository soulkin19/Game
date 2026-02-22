
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>HEXAGON HELL -CHAOS-</title>
    <style>
        :root {
            --neon-pink: #ff0055;
            --neon-blue: #00f2ff;
            --bg-dark: #0a0a0a;
        }
        * { touch-action: none; -webkit-tap-highlight-color: transparent; outline: none; box-sizing: border-box; }
        body { margin: 0; background: #000; color: #fff; font-family: 'Segoe UI', 'Roboto', sans-serif; overflow: hidden; display: flex; align-items: center; justify-content: center; height: 100vh; width: 100%; position: fixed; }
        
        #game-container { position: relative; width: 600px; height: 600px; max-width: 95vw; max-height: 85vh; border-radius: 15px; overflow: hidden; box-shadow: 0 0 50px rgba(0,0,0,0.5); }
        canvas { background: #000; display: block; width: 100%; height: 100%; }
        
        /* UI共通 */
        .overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%; 
            background: rgba(0, 0, 0, 0.85); display: flex; flex-direction: column; 
            align-items: center; justify-content: center; z-index: 2000; backdrop-filter: blur(8px);
            transition: all 0.3s ease;
        }

        /* スタート画面 */
        #start-screen h1 { font-size: 3.5rem; color: var(--neon-pink); text-shadow: 0 0 20px var(--neon-pink); margin: 0; font-style: italic; }
        
        /* ボタン */
        .btn {
            background: none; border: 2px solid #fff; color: #fff; padding: 12px 40px;
            font-size: 1.2rem; cursor: pointer; border-radius: 50px; font-weight: bold;
            transition: 0.2s; letter-spacing: 2px; text-transform: uppercase; margin: 10px;
        }
        .btn:hover { background: #fff; color: #000; box-shadow: 0 0 20px #fff; }
        .btn-primary { border-color: var(--neon-blue); color: var(--neon-blue); }
        .btn-primary:hover { background: var(--neon-blue); color: #000; box-shadow: 0 0 20px var(--neon-blue); }

        /* リザルトカード */
        .card { 
            background: #111; padding: 30px; border: 1px solid #333; border-radius: 20px; 
            width: 85%; max-width: 400px; text-align: center;
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
        }
        .score-val { font-size: 3rem; font-weight: 900; color: var(--neon-pink); margin: 10px 0; }

        /* ランキング */
        .ranking-box { 
            background: #000; border-radius: 10px; padding: 15px; margin: 15px 0; 
            height: 180px; overflow-y: auto; border: 1px solid #222;
        }
        .rank-item { 
            display: flex; justify-content: space-between; padding: 8px; 
            border-bottom: 1px solid #111; font-size: 0.9rem; font-family: monospace;
        }
        .rank-name { color: var(--neon-blue); }

        /* 入力エリア */
        .input-group { display: flex; gap: 5px; margin-top: 10px; }
        input[type="text"] {
            background: #111; border: 1px solid #333; color: #fff; padding: 10px; 
            border-radius: 5px; flex: 1; outline: none;
        }
        input:focus { border-color: var(--neon-blue); }

        /* プレイ中のUI */
        #ui-layer { position: absolute; top: 0; left: 0; width: 100%; padding: 20px; pointer-events: none; z-index: 10; display: flex; justify-content: space-between; }
        .stat-box { background: rgba(0,0,0,0.5); padding: 5px 15px; border-radius: 10px; border-left: 4px solid var(--neon-pink); }
        #pause-trigger { pointer-events: auto; background: none; border: 1px solid #fff; color: #fff; cursor: pointer; padding: 5px 15px; border-radius: 5px; }
    </style>
</head>
<body>

    <div id="game-container">
        <div id="ui-layer">
            <div class="stat-box">
                <div id="phase-ui" style="font-size: 0.8rem; color: var(--neon-pink);">PHASE 1</div>
                <div id="score-ui" style="font-size: 1.5rem; font-weight: bold;">0</div>
            </div>
            <button id="pause-trigger">|| PAUSE</button>
        </div>

        <div id="start-screen" class="overlay">
            <h1>HEXAGON<br>HELL</h1>
            <p style="color: #666; letter-spacing: 4px; margin-top: 5px;">CHAOS EDITION</p>
            <div style="margin-top: 40px;">
                <button class="btn btn-primary" id="start-btn">START MISSION</button>
            </div>
        </div>

        <div id="pause-screen" class="overlay" style="display:none;">
            <h2 style="font-size: 2rem; letter-spacing: 10px;">PAUSED</h2>
            <button class="btn" id="resume-btn">RESUME</button>
            <button class="btn" style="font-size: 0.8rem; border:none; opacity: 0.5;" onclick="location.reload()">ABORT MISSION</button>
        </div>

        <div id="result-screen" class="overlay" style="display:none;">
            <div class="card">
                <h2 style="margin:0; font-size: 1rem; color: #666;">MISSION FAILED</h2>
                <div class="score-val" id="current-score">0</div>
                
                <div class="ranking-box" id="rank-list-container">
                    </div>

                <div id="submission-area">
                    <div class="input-group">
                        <input type="text" id="player-name" placeholder="YOUR NAME" maxlength="10">
                        <button class="btn-primary" id="submit-btn" style="padding: 5px 15px; border-radius: 5px; cursor:pointer;">SEND</button>
                    </div>
                </div>

                <button class="btn" style="width: 100%; margin: 20px 0 0 0;" onclick="location.reload()">RETRY</button>
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

        // --- Game Setup ---
        const canvas = document.getElementById('game-canvas');
        const ctx = canvas.getContext('2d');
        const scoreUI = document.getElementById('score-ui');
        const phaseUI = document.getElementById('phase-ui');
        
        canvas.width = 600; canvas.height = 600;

        let score = 0, angle = 0, rotationDir = 0.08, obstacles = [], phase = 1;
        let gameState = 'START'; 
        const centerX = 300, centerY = 300, orbitRadius = 90;

        // --- Controller ---
        document.getElementById('start-btn').onclick = () => {
            document.getElementById('start-screen').style.display = 'none';
            gameState = 'PLAYING';
        };

        document.getElementById('pause-trigger').onclick = () => {
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

        // --- Ranking System (Fixed & Improved) ---
        async function fetchRankings() {
            const container = document.getElementById('rank-list-container');
            container.innerHTML = '<div style="padding:20px;">LOADING...</div>';
            try {
                // ここでエラーが出る場合、Firebaseコンソールのログに「インデックスが必要」というURLが出ます
                const q = query(collection(db, "world_ranking"), orderBy("score", "desc"), limit(10));
                const querySnapshot = await getDocs(q);
                let html = '';
                querySnapshot.forEach((doc) => {
                    const data = doc.data();
                    html += `<div class="rank-item"><span class="rank-name">${data.name}</span><span>${Math.floor(data.score)}</span></div>`;
                });
                container.innerHTML = html || 'NO DATA';
            } catch (e) {
                console.error("Ranking Error:", e);
                container.innerHTML = '<div style="color:red; font-size:0.7rem;">FAIL TO LOAD RANKING<br>Check Firestore Rules & Index</div>';
            }
        }

        document.getElementById('submit-btn').onclick = async () => {
            const nameField = document.getElementById('player-name');
            const name = nameField.value.trim() || "ANON";
            const btn = document.getElementById('submit-btn');
            
            btn.disabled = true;
            btn.innerText = "...";

            try {
                // 送信処理
                await addDoc(collection(db, "world_ranking"), {
                    name: name,
                    score: Math.floor(score),
                    createdAt: serverTimestamp()
                });
                
                document.getElementById('submission-area').innerHTML = "<p style='color:var(--neon-blue); font-size:0.8rem;'>SCORE SYNCED!</p>";
                await fetchRankings(); // 更新
            } catch (e) {
                alert("Submit Error. Check console.");
                console.error("Submit Error:", e);
                btn.disabled = false;
                btn.innerText = "SEND";
            }
        };

        // --- Core Engine ---
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
                if (score > 20 && phase === 1) { phase = 2; phaseUI.innerText = "PHASE 2"; }
                if (score > 50 && phase === 2) { phase = 3; phaseUI.innerText = "PHASE 3"; }

                if (Math.random() < 0.03 + (phase * 0.01)) createEnemy();

                const px = centerX + Math.cos(angle) * orbitRadius;
                const py = centerY + Math.sin(angle) * orbitRadius;

                for (let i = obstacles.length - 1; i >= 0; i--) {
                    let ob = obstacles[i];
                    ob.x += ob.vx; ob.y += ob.vy;
                    const dist = Math.hypot(px - ob.x, py - ob.y);
                    
                    if (dist < 22) {
                        gameState = 'GAMEOVER';
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
            ctx.fillStyle = 'rgba(0,0,0,0.3)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Orbit line
            ctx.strokeStyle = '#1a1a1a';
            ctx.setLineDash([10, 10]);
            ctx.lineWidth = 2;
            ctx.beginPath(); ctx.arc(centerX, centerY, orbitRadius, 0, Math.PI*2); ctx.stroke();

            // Enemies
            obstacles.forEach(ob => {
                ctx.fillStyle = ob.color;
                ctx.shadowBlur = 15;
                ctx.shadowColor = ob.color;
                ctx.fillRect(ob.x - 12, ob.y - 12, 24, 24);
            });

            // Player
            ctx.shadowBlur = 20;
            ctx.shadowColor = varColor('--neon-blue');
            ctx.fillStyle = varColor('--neon-blue');
            ctx.beginPath(); ctx.arc(px, py, 12, 0, Math.PI*2); ctx.fill();
            ctx.shadowBlur = 0;
        }

        function varColor(name) { return getComputedStyle(document.documentElement).getPropertyValue(name).trim(); }

        update();
    </script>
</body>
