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
        #debug-indicator { color: #0f0; font-size: 0.8rem; display: none; }

        /* リザルト画面 */
        #result-screen { 
            position: absolute; top: 0; left: 0; width: 100%; height: 100%; 
            background: rgba(0, 0, 0, 0.95); display: none; flex-direction: column; 
            align-items: center; justify-content: center; z-index: 2000;
        }
        .result-card { background: #050505; padding: 30px; border: 2px solid #0ff; border-radius: 10px; text-align: center; width: 85%; box-shadow: 0 0 30px rgba(0,255,255,0.2); }
        .result-card h2 { color: #0ff; margin: 0 0 10px 0; font-size: 2.2rem; }
        
        .stats-area { margin-bottom: 25px; border-bottom: 1px solid #333; padding-bottom: 15px; }
        .stat-line { font-size: 1.2rem; margin: 5px 0; }
        .best-score { color: #ffcc00; font-weight: bold; }

        .color-settings { margin-bottom: 25px; display: flex; flex-direction: column; align-items: center; gap: 8px; }
        .color-settings label { font-size: 0.8rem; color: #aaa; letter-spacing: 1px; }
        input[type="color"] { background: none; border: 2px solid #fff; cursor: pointer; width: 50px; height: 50px; border-radius: 50%; padding: 0; overflow: hidden; }

        .btn-container { display: flex; flex-direction: column; gap: 10px; width: 100%; }
        .game-btn { padding: 15px; font-size: 1.1rem; font-weight: bold; border: none; cursor: pointer; border-radius: 5px; font-family: inherit; width: 100%; }
        .retry-btn { background: #fff; color: #000; }
        .share-btn { background: #000; color: #fff; border: 1px solid #fff; }
    </style>
</head>
<body>
    <div id="game-container">
        <div id="ui">
            <div id="phase-ui" class="phase-display">PHASE: 1 (easy)</div>
            <div id="debug-indicator">DEBUG MODE (PHASE 4)</div>
            <div id="score-ui" class="score-display">0</div>
        </div>

        <div id="result-screen">
            <div class="result-card">
                <h2>You were BROKEN</h2>
                
                <div class="stats-area">
                    <div class="stat-line">SCORE: <span id="current-score">0</span></div>
                    <div class="stat-line">BEST: <span id="best-score" class="best-score">0</span></div>
                </div>

                <div class="color-settings">
                    <label>CHANGE PLAYER COLOR</label>
                    <input type="color" id="playerColor" value="#ffffff">
                </div>

                <div class="btn-container">
                    <button class="game-btn retry-btn" id="retry-trigger">RETRY</button>
                    <button class="game-btn share-btn" id="share-trigger">Twitterでシェア</button>
                </div>
            </div>
        </div>
        <canvas id="game-canvas"></canvas>
    </div>

    <script>
        const canvas = document.getElementById('game-canvas');
        const ctx = canvas.getContext('2d');
        const scoreUI = document.getElementById('score-ui');
        const phaseUI = document.getElementById('phase-ui');
        const debugUI = document.getElementById('debug-indicator');
        const resultScreen = document.getElementById('result-screen');
        const currentScoreEl = document.getElementById('current-score');
        const bestScoreEl = document.getElementById('best-score');
        const colorPicker = document.getElementById('playerColor');

        canvas.width = 600; canvas.height = 600;

        let score = 0, gameActive = true, angle = 0, rotationDir = 0.08, obstacles = [], shakeTime = 0, phase = 1;
        let p4Timer = 0, frameCount = 0;
        let bestScore = localStorage.getItem('hexagon_best_v2') || 0;
        const centerX = 300, centerY = 300, orbitRadius = 90;

        // URLハッシュのチェック
        const isDevMode = () => window.location.hash === '#hisansuki';

        function initGame() {
            score = 0; obstacles = []; gameActive = true; rotationDir = 0.08; frameCount = 0;
            resultScreen.style.display = 'none';
            scoreUI.innerText = "0";

            if (isDevMode()) {
                phase = 4;
                p4Timer = 0; // デバッグモードは待機なしで開始
                phaseUI.innerText = "PHASE: 4 (GLITCH ABYSS)";
                phaseUI.style.color = "#0ff";
                debugUI.style.display = "block";
            } else {
                phase = 1;
                p4Timer = 0;
                phaseUI.innerText = "PHASE: 1 (easy)";
                phaseUI.style.color = "#ff0055";
                debugUI.style.display = "none";
            }
        }

        canvas.addEventListener('pointerdown', (e) => {
            if (e.cancelable) e.preventDefault();
            if (gameActive) rotationDir *= -1;
        });

        document.getElementById('retry-trigger').onclick = () => initGame();
        document.getElementById('share-trigger').onclick = () => {
            const text = `Rismer Chaosでのスコアは\nSCORE: ${Math.floor(score)}でした。みんなもhttps://soulkin19.github.io/Game/で遊ぼう！`;
            window.open(`https://twitter.com/intent/tweet?text=${encodeURIComponent(text)}`, '_blank');
        };

        function createEnemy() {
            if (phase === 4 && p4Timer > 0) return;
            const side = Math.floor(Math.random() * 4);
            let x, y, vx, vy;
            let baseSpeed = phase === 1 ? 1.5 : phase === 2 ? 2.5 : phase === 3 ? 3.5 : 5.0;
            let speed = (baseSpeed + Math.random() * 1.5) + (phase * 0.5);
            if (side === 0) { x = -20; y = Math.random()*600; vx = speed; vy = 0; }
            else if (side === 1) { x = 620; y = Math.random()*600; vx = -speed; vy = 0; }
            else if (side === 2) { x = Math.random()*600; y = -20; vx = 0; vy = speed; }
            else { x = Math.random()*600; y = 620; vx = 0; vy = -speed; }
            obstacles.push({ x, y, vx, vy, color: '#ff0055', split: 0 });
        }

        function update() {
            if (!gameActive) { requestAnimationFrame(update); return; }
            frameCount++;

            // 通常モードのフェーズ進行
            if (!isDevMode()) {
                if (score > 10 && phase === 1) { phase = 2; phaseUI.innerText = "PHASE: 2 (normal)"; }
                if (score > 20 && phase === 2) { phase = 3; phaseUI.innerText = "PHASE: 3 (hard)"; }
                if (score > 35 && phase === 3) { 
                    phase = 4; 
                    obstacles = []; 
                    p4Timer = 180; 
                    phaseUI.innerText = "PHASE: 4 (WARNING)"; 
                    phaseUI.style.color = "#0ff";
                }
            }

            if (p4Timer > 0) {
                p4Timer--;
                if (p4Timer === 0) phaseUI.innerText = "PHASE: 4 (GLITCH ABYSS)";
            }

            angle += rotationDir;
            const px = centerX + Math.cos(angle) * orbitRadius;
            const py = centerY + Math.sin(angle) * orbitRadius;

            let rate = phase === 1 ? 0.03 : (phase === 4 ? 0.025 : 0.04 + (phase * 0.015));
            if (Math.random() < rate) createEnemy();

            for (let i = obstacles.length - 1; i >= 0; i--) {
                let ob = obstacles[i];
                ob.x += ob.vx; ob.y += ob.vy;
                const dist = Math.hypot(px - ob.x, py - ob.y);
                
                if (dist < 40 && dist > 22) { 
                    score += (phase >= 4 ? 0.15 : 0.05); 
                    scoreUI.innerText = Math.floor(score);
                }

                if (phase >= 4 && ob.split < 2 && p4Timer === 0) {
                    const distToC = Math.hypot(ob.x - centerX, ob.y - centerY);
                    if (distToC < (ob.split === 0 ? 250 : 130)) {
                        const s = Math.hypot(ob.vx, ob.vy) * 1.2;
                        const col = `hsl(${(frameCount * 10) % 360}, 100%, 60%)`;
                        obstacles.push(
                            {x:ob.x, y:ob.y, vx:s, vy:0, color:col, split:ob.split+1},
                            {x:ob.x, y:ob.y, vx:-s, vy:0, color:col, split:ob.split+1},
                            {x:ob.x, y:ob.y, vx:0, vy:s, color:col, split:ob.split+1},
                            {x:ob.x, y:ob.y, vx:0, vy:-s, color:col, split:ob.split+1}
                        );
                        obstacles.splice(i, 1); continue;
                    }
                }

                if (dist < 22) {
                    gameActive = false;
                    const finalS = Math.floor(score);
                    if (finalS > bestScore) {
                        bestScore = finalS;
                        localStorage.setItem('hexagon_best_v2', bestScore);
                    }
                    currentScoreEl.innerText = finalS;
                    bestScoreEl.innerText = bestScore;
                    resultScreen.style.display = 'flex';
                }
                if (ob.x < -150 || ob.x > 750 || ob.y < -150 || ob.y > 750) {
                    obstacles.splice(i, 1); score += 1; scoreUI.innerText = Math.floor(score);
                }
            }
            draw(px, py);
            requestAnimationFrame(update);
        }

        function draw(px, py) {
            ctx.fillStyle = `rgba(0,0,0,${0.25 - (phase * 0.04) + (phase >= 4 && p4Timer === 0 && Math.random() < 0.05 ? 0.5 : 0)})`;
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            ctx.strokeStyle = phase >= 4 ? `hsl(${frameCount % 360}, 100%, 50%)` : '#333';
            ctx.setLineDash([5, 5]);
            ctx.beginPath(); ctx.arc(centerX, centerY, orbitRadius, 0, Math.PI*2); ctx.stroke();

            obstacles.forEach(ob => {
                ctx.fillStyle = ob.color;
                if(phase >= 2) { ctx.shadowBlur = 15; ctx.shadowColor = ob.color; }
                ctx.fillRect(ob.x - 12, ob.y - 12, 24, 24);
                ctx.shadowBlur = 0;
            });

            ctx.fillStyle = colorPicker.value;
            ctx.shadowBlur = 20;
            ctx.shadowColor = colorPicker.value;
            ctx.beginPath(); ctx.arc(px, py, 12, 0, Math.PI*2); ctx.fill();
            ctx.shadowBlur = 0;
        }

        // 初期実行
        initGame();
        window.addEventListener('hashchange', initGame);
        
        requestAnimationFrame(update);
    </script>
</body>
</html>
