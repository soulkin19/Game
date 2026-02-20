<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>HEXAGON HELL -CHAOS-</title>
    <style>
        * { touch-action: none; -webkit-tap-highlight-color: transparent; outline: none; box-sizing: border-box; }
        body { margin: 0; background: #000; color: #fff; font-family: 'Courier New', monospace; overflow: hidden; display: flex; align-items: center; justify-content: center; height: 100vh; width: 100%; position: fixed; }
        
        #game-container { position: relative; width: 600px; height: 600px; max-width: 95vw; max-height: 80vh; }
        canvas { background: #000; border: 4px solid #333; width: 100%; height: 100%; display: block; box-shadow: 0 0 20px rgba(0,255,255,0.2); }
        
        #ui { position: absolute; top: 20px; text-align: center; pointer-events: none; width: 100%; z-index: 10; }
        .score-display { font-size: 4rem; font-weight: bold; text-shadow: 0 0 20px #ff0055; margin: 0; transition: transform 0.1s; }
        .phase-display { font-size: 1.2rem; color: #ff0055; font-weight: bold; text-transform: uppercase; letter-spacing: 2px; }

        #result-screen { 
            position: absolute; top: 0; left: 0; width: 100%; height: 100%; 
            background: rgba(0, 0, 0, 0.95); display: none; flex-direction: column; 
            align-items: center; justify-content: center; z-index: 2000;
        }
        .result-card { background: #050505; padding: 40px; border: 2px solid #0ff; border-radius: 15px; box-shadow: 0 0 50px rgba(0, 255, 255, 0.3); text-align: center; width: 85%; }
        .result-card h2 { font-size: 2.5rem; color: #0ff; margin: 0 0 10px 0; text-shadow: 0 0 10px #0ff; }
        .stats-text { font-size: 1.4rem; margin-bottom: 30px; line-height: 1.8; color: #fff; }
        
        .btn-container { display: flex; flex-direction: column; gap: 15px; width: 100%; }
        .game-btn { 
            padding: 18px; font-size: 1.2rem; font-weight: bold; border: none; 
            cursor: pointer; border-radius: 8px; font-family: inherit; transition: 0.3s;
            width: 100%; text-transform: uppercase;
        }
        .retry-btn { background: #fff; color: #000; box-shadow: 0 4px 0 #bbb; }
        .share-btn { background: #000; color: #fff; border: 2px solid #fff; }
        .game-btn:active { transform: translateY(4px); box-shadow: none; }
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
                <h2>BROKEN</h2>
                <div id="final-stats" class="stats-text"></div>
                <div class="btn-container">
                    <button class="game-btn retry-btn" id="retry-trigger">Try Again</button>
                    <button class="game-btn share-btn" id="share-trigger">Share Result</button>
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
        const resultScreen = document.getElementById('result-screen');
        const finalStats = document.getElementById('final-stats');
        const retryBtn = document.getElementById('retry-trigger');
        const shareBtn = document.getElementById('share-trigger');

        canvas.width = 600;
        canvas.height = 600;

        let score = 0, gameActive = true, angle = 0, rotationDir = 0.08, obstacles = [], shakeTime = 0, phase = 1, frameCount = 0;
        let bestScore = localStorage.getItem('hexagon_best_v2') || 0;
        const centerX = 300, centerY = 300, orbitRadius = 90;

        function initGame() {
            score = 0; phase = 1; obstacles = []; gameActive = true; rotationDir = 0.08;
            resultScreen.style.display = 'none';
            scoreUI.innerText = "0";
            phaseUI.innerText = "PHASE: 1 (easy)";
            phaseUI.style.color = "#ff0055";
        }

        const handleInput = (e) => {
            if (e.cancelable) e.preventDefault();
            if (gameActive) {
                rotationDir *= -1;
                // 操作フィードバック
                scoreUI.style.transform = "scale(1.1)";
                setTimeout(() => scoreUI.style.transform = "scale(1)", 100);
            }
        };
        canvas.addEventListener('pointerdown', handleInput);

        retryBtn.onclick = () => initGame();
        shareBtn.onclick = () => {
            const text = `HEXAGON HELL\nScore: ${Math.floor(score)}\nPHASE: ${phase}\n#HEXAGON_HELL`;
            window.open(`https://twitter.com/intent/tweet?text=${encodeURIComponent(text)}`, '_blank');
        };

        function createEnemy() {
            const side = Math.floor(Math.random() * 4);
            let x, y, vx, vy;
            let baseSpeed = phase === 1 ? 1.5 : phase === 2 ? 2.5 : phase === 3 ? 3.5 : 4.0;
            let speed = (baseSpeed + Math.random() * 1.5) + (phase * 0.5);
            if (side === 0) { x = -20; y = Math.random()*600; vx = speed; vy = 0; }
            else if (side === 1) { x = 620; y = Math.random()*600; vx = -speed; vy = 0; }
            else if (side === 2) { x = Math.random()*600; y = -20; vx = 0; vy = speed; }
            else { x = Math.random()*600; y = 620; vx = 0; vy = -speed; }
            obstacles.push({ x, y, vx, vy, color: '#ff0055', split: 0 });
        }

        function gameOver() {
            gameActive = false;
            const finalScore = Math.floor(score);
            if (finalScore > bestScore) {
                bestScore = finalScore;
                localStorage.setItem('hexagon_best_v2', bestScore);
            }
            finalStats.innerHTML = `SCORE: ${finalScore}<br>BEST: ${bestScore}`;
            resultScreen.style.display = 'flex';
        }

        function update() {
            if (!gameActive) {
                requestAnimationFrame(update);
                return;
            }
            frameCount++;

            if (score > 10 && phase === 1) { phase = 2; phaseUI.innerText = "PHASE: 2 (normal)"; }
            if (score > 20 && phase === 2) { phase = 3; phaseUI.innerText = "PHASE: 3 (hard)"; }
            if (score > 25 && phase === 3) { 
                phase = 4; 
                phaseUI.innerText = "PHASE: 4 (GLITCH RAVE)"; 
                phaseUI.style.color = "#0ff";
            }

            angle += rotationDir;
            const px = centerX + Math.cos(angle) * orbitRadius;
            const py = centerY + Math.sin(angle) * orbitRadius;

            if (Math.random() < (phase === 1 ? 0.03 : 0.04 + (phase * 0.015))) createEnemy();

            for (let i = obstacles.length - 1; i >= 0; i--) {
                let ob = obstacles[i];
                ob.x += ob.vx; ob.y += ob.vy;
                const dist = Math.hypot(px - ob.x, py - ob.y);
                
                // かすりボーナス（PHASE 4では3倍）
                if (dist < 40 && dist > 22) { 
                    score += (phase >= 4 ? 0.15 : 0.05); 
                    scoreUI.innerText = Math.floor(score);
                }

                if (phase >= 4 && ob.split < 2) {
                    const distToC = Math.hypot(ob.x - centerX, ob.y - centerY);
                    if (distToC < (ob.split === 0 ? 250 : 130)) {
                        const s = Math.hypot(ob.vx, ob.vy) * 1.1;
                        // PHASE 4 の分裂はカラフルに
                        const col = `hsl(${(frameCount * 5) % 360}, 100%, 60%)`;
                        obstacles.push(
                            {x:ob.x, y:ob.y, vx:s, vy:0, color:col, split:ob.split+1},
                            {x:ob.x, y:ob.y, vx:-s, vy:0, color:col, split:ob.split+1},
                            {x:ob.x, y:ob.y, vx:0, vy:s, color:col, split:ob.split+1},
                            {x:ob.x, y:ob.y, vx:0, vy:-s, color:col, split:ob.split+1}
                        );
                        obstacles.splice(i, 1); continue;
                    }
                }

                if (dist < 22) { shakeTime = 30; gameOver(); }
                if (ob.x < -150 || ob.x > 750 || ob.y < -150 || ob.y > 750) {
                    obstacles.splice(i, 1); score += 1; scoreUI.innerText = Math.floor(score);
                }
            }
            if (shakeTime > 0) shakeTime--;
            draw(px, py);
            requestAnimationFrame(update);
        }

        function draw(px, py) {
            let sx = (Math.random() - 0.5) * shakeTime, sy = (Math.random() - 0.5) * shakeTime;
            
            // PHASE 4 のズーム演出（脈動）
            let zoom = 1;
            if (phase >= 4) zoom = 1 + Math.sin(frameCount * 0.2) * 0.01;
            
            ctx.setTransform(zoom, 0, 0, zoom, sx + (1-zoom)*centerX, sy + (1-zoom)*centerY);
            
            // 背景（PHASE 4はより残像を強く）
            let bgAlpha = 0.25 - (phase * 0.04);
            if (phase >= 4 && Math.random() < 0.05) bgAlpha = 0.6; 
            ctx.fillStyle = `rgba(0,0,0,${bgAlpha})`;
            ctx.fillRect(-200, -200, 1000, 1000);

            // 軌道
            ctx.strokeStyle = phase >= 4 ? `hsl(${frameCount % 360}, 100%, 50%)` : '#333';
            ctx.setLineDash([5, 5]);
            ctx.beginPath(); ctx.arc(centerX, centerY, orbitRadius, 0, Math.PI*2); ctx.stroke();

            // 敵
            obstacles.forEach(ob => {
                ctx.fillStyle = ob.color;
                if(phase >= 2) { 
                    ctx.shadowBlur = phase >= 4 ? 20 : 15; 
                    ctx.shadowColor = ob.color; 
                }
                ctx.fillRect(ob.x - 12, ob.y - 12, 24, 24);
                ctx.shadowBlur = 0;
            });

            // 自機
            ctx.fillStyle = '#fff'; ctx.shadowBlur = 20; ctx.shadowColor = '#0ff';
            ctx.beginPath(); ctx.arc(px, py, 12, 0, Math.PI*2); ctx.fill(); ctx.shadowBlur = 0;
        }
        requestAnimationFrame(update);
    </script>
</body>
</html>
