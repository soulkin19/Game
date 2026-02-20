<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>HEXAGON HELL -CHAOS-</title>
    <style>
        * { touch-action: none; -webkit-tap-highlight-color: transparent; outline: none; box-sizing: border-box; }
        body { margin: 0; background: #000; color: #fff; font-family: 'Courier New', monospace; overflow: hidden; display: flex; flex-direction: column; align-items: center; justify-content: center; height: 100vh; width: 100%; position: fixed; }
        
        #game-container { position: relative; width: 600px; height: 600px; max-width: 95vw; max-height: 70vh; }
        canvas { background: #000; border: 4px solid #333; width: 100%; height: 100%; display: block; }
        
        #ui { position: absolute; top: 20px; text-align: center; pointer-events: none; width: 100%; z-index: 10; }
        .score-display { font-size: 4rem; font-weight: bold; text-shadow: 0 0 20px #ff0055; margin: 0; }
        .phase-display { font-size: 1.2rem; color: #ff0055; font-weight: bold; text-transform: uppercase; }

        /* カラーピッカー */
        #settings { margin-top: 20px; display: flex; align-items: center; gap: 10px; z-index: 3000; }
        #settings label { font-size: 0.9rem; font-weight: bold; color: #0ff; }
        input[type="color"] { background: none; border: 2px solid #333; cursor: pointer; width: 40px; height: 40px; border-radius: 5px; }

        #result-screen { 
            position: absolute; top: 0; left: 0; width: 100%; height: 100%; 
            background: rgba(0, 0, 0, 0.95); display: none; flex-direction: column; 
            align-items: center; justify-content: center; z-index: 2000;
        }
        .result-card { background: #050505; padding: 30px; border: 2px solid #0ff; border-radius: 10px; text-align: center; width: 80%; }
        .game-btn { 
            padding: 15px; font-size: 1.1rem; font-weight: bold; border: none; cursor: pointer; border-radius: 5px; font-family: inherit; width: 100%; margin-top: 10px;
        }
        .retry-btn { background: #fff; color: #000; }
        .share-btn { background: #000; color: #fff; border: 1px solid #fff; }
    </style>
</head>
<body>
    <div id="game-container">
        <div id="ui">
            <div id="phase-ui" class="phase-display">PHASE: 1 (easy)</div>
            <div id="score-ui" class="score-display">0</div>
        </div>

        <div id="result-screen">
            <div class="result-card">
                <h2 style="color:#0ff">BROKEN</h2>
                <div id="final-stats" style="font-size:1.5rem; margin-bottom:20px;"></div>
                <div class="btn-container">
                    <button class="game-btn retry-btn" id="retry-trigger">RETRY</button>
                    <button class="game-btn share-btn" id="share-trigger">SHARE ON X</button>
                </div>
            </div>
        </div>
        <canvas id="game-canvas"></canvas>
    </div>

    <div id="settings">
        <label>PLAYER COLOR:</label>
        <input type="color" id="playerColor" value="#ffffff">
    </div>

    <script>
        const canvas = document.getElementById('game-canvas');
        const ctx = canvas.getContext('2d');
        const scoreUI = document.getElementById('score-ui');
        const phaseUI = document.getElementById('phase-ui');
        const resultScreen = document.getElementById('result-screen');
        const finalStats = document.getElementById('final-stats');
        const colorPicker = document.getElementById('playerColor');

        canvas.width = 600; canvas.height = 600;

        let score = 0, gameActive = true, angle = 0, rotationDir = 0.08, obstacles = [], shakeTime = 0, phase = 1;
        let p4Timer = 0, frameCount = 0;
        let bestScore = localStorage.getItem('hexagon_best_v2') || 0;
        const centerX = 300, centerY = 300, orbitRadius = 90;

        function initGame() {
            score = 0; phase = 1; obstacles = []; gameActive = true; rotationDir = 0.08; p4Timer = 0;
            resultScreen.style.display = 'none';
            scoreUI.innerText = "0"; phaseUI.innerText = "PHASE: 1 (easy)";
        }

        canvas.addEventListener('pointerdown', (e) => {
            if (e.cancelable) e.preventDefault();
            if (gameActive) rotationDir *= -1;
        });

        document.getElementById('retry-trigger').onclick = () => initGame();
        document.getElementById('share-trigger').onclick = () => {
            const text = `HEXAGON HELL\nSCORE: ${Math.floor(score)}`;
            window.open(`https://twitter.com/intent/tweet?text=${encodeURIComponent(text)}`, '_blank');
        };

        function createEnemy() {
            if (phase === 4 && p4Timer > 0) return; // P4移行中は出現させない
            
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

            // フェーズ移行
            if (score > 10 && phase === 1) { phase = 2; phaseUI.innerText = "PHASE: 2 (normal)"; }
            if (score > 20 && phase === 2) { phase = 3; phaseUI.innerText = "PHASE: 3 (hard)"; }
            if (score > 25 && phase === 3) { 
                phase = 4; 
                obstacles = []; // 弾を全削除
                p4Timer = 180; // 3秒間 (60fps * 3)
                phaseUI.innerText = "PHASE: 4 (WARNING)"; 
                phaseUI.style.color = "#0ff";
            }

            if (p4Timer > 0) {
                p4Timer--;
                if (p4Timer === 0) phaseUI.innerText = "PHASE: 4 (GLITCH ABYSS)";
            }

            angle += rotationDir;
            const px = centerX + Math.cos(angle) * orbitRadius;
            const py = centerY + Math.sin(angle) * orbitRadius;

            // 出現率：PHASE 4は数を絞る
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
                    finalStats.innerHTML = `SCORE: ${Math.floor(score)}`;
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

            // 軌道
            ctx.strokeStyle = phase >= 4 ? `hsl(${frameCount % 360}, 100%, 50%)` : '#333';
            ctx.setLineDash([5, 5]);
            ctx.beginPath(); ctx.arc(centerX, centerY, orbitRadius, 0, Math.PI*2); ctx.stroke();

            // 敵
            obstacles.forEach(ob => {
                ctx.fillStyle = ob.color;
                if(phase >= 2) { ctx.shadowBlur = 15; ctx.shadowColor = ob.color; }
                ctx.fillRect(ob.x - 12, ob.y - 12, 24, 24);
                ctx.shadowBlur = 0;
            });

            // 自機（カラーピッカーの色を適用）
            ctx.fillStyle = colorPicker.value;
            ctx.shadowBlur = 20;
            ctx.shadowColor = colorPicker.value;
            ctx.beginPath(); ctx.arc(px, py, 12, 0, Math.PI*2); ctx.fill();
            ctx.shadowBlur = 0;
        }
        requestAnimationFrame(update);
    </script>
</body>
</html>
