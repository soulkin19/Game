<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>HEXAGON HELL -CHAOS-</title>
    <style>
        * { touch-action: none; -webkit-tap-highlight-color: transparent; outline: none; }
        body { margin: 0; background: #000; color: #fff; font-family: 'Courier New', monospace; overflow: hidden; display: flex; align-items: center; justify-content: center; height: 100vh; position: fixed; width: 100%; }
        canvas { background: #000; border: 4px solid #333; max-width: 95vw; max-height: 80vh; box-shadow: 0 0 50px rgba(255, 0, 85, 0.3); }
        
        #ui { position: absolute; top: 5%; text-align: center; pointer-events: none; width: 100%; z-index: 10; }
        .score { font-size: 4rem; font-weight: bold; text-shadow: 0 0 20px #ff0055; margin: 0; }
        .phase { font-size: 1.2rem; color: #ff0055; font-weight: bold; text-transform: uppercase; }

        /* リザルト画面 */
        #result { 
            position: absolute; display: none; flex-direction: column; align-items: center; justify-content: center;
            width: 100%; height: 100%; background: rgba(0,0,0,0.9); z-index: 100; text-align: center;
        }
        .result-title { font-size: 3rem; color: #ff0055; margin-bottom: 10px; animation: pulse 1s infinite; }
        .result-stats { font-size: 1.5rem; margin: 20px 0; line-height: 1.6; }
        .btn-group { display: flex; gap: 10px; }
        button { 
            padding: 15px 25px; font-size: 1rem; font-weight: bold; border: none; cursor: pointer;
            border-radius: 5px; transition: 0.2s; font-family: inherit;
        }
        .retry-btn { background: #fff; color: #000; }
        .share-btn { background: #1da1f2; color: #fff; }
        button:active { transform: scale(0.9); }
        @keyframes pulse { 0% { opacity: 1; } 50% { opacity: 0.5; } 100% { opacity: 1; } }
    </style>
</head>
<body>
    <div id="ui">
        <div id="phase" class="phase">PHASE: 1 (easy)</div>
        <div id="score" class="score">0</div>
    </div>

    <div id="result">
        <div class="result-title">BROKEN</div>
        <div class="result-stats" id="stats">SCORE: 0<br>BEST: 0</div>
        <div class="btn-group">
            <button class="retry-btn" onclick="location.reload()">RETRY</button>
            <button class="share-btn" id="shareBtn">SHARE ON X</button>
        </div>
    </div>

    <canvas id="game"></canvas>

    <script>
        const canvas = document.getElementById('game');
        const ctx = canvas.getContext('2d');
        const scoreEl = document.getElementById('score');
        const phaseEl = document.getElementById('phase');
        const resultEl = document.getElementById('result');
        const statsEl = document.getElementById('stats');
        const shareBtn = document.getElementById('shareBtn');

        canvas.width = 600;
        canvas.height = 600;

        let score = 0;
        let gameActive = true;
        let angle = 0;
        let rotationDir = 0.08;
        let obstacles = [];
        let shakeTime = 0;
        let phase = 1;
        let bestScore = localStorage.getItem('hexagon_best') || 0;

        const centerX = canvas.width / 2;
        const centerY = canvas.height / 2;
        const orbitRadius = 90;

        const handleInput = (e) => {
            if (e.cancelable) e.preventDefault();
            if (!gameActive) return;
            rotationDir *= -1;
        };

        window.addEventListener('mousedown', handleInput, { passive: false });
        window.addEventListener('touchstart', handleInput, { passive: false });

        function spawnObstacle(x, y, vx, vy, color = '#ff0055', splitCount = 0) {
            obstacles.push({ x, y, vx, vy, color, splitCount });
        }

        function createEnemy() {
            const side = Math.floor(Math.random() * 4);
            let x, y, vx, vy;
            let baseSpeed = phase === 1 ? 1.5 : phase === 2 ? 2.5 : phase === 3 ? 3.5 : 4.0;
            let speed = (baseSpeed + Math.random() * 1.5) + (phase * 0.5);

            if (side === 0) { x = -20; y = Math.random() * 600; vx = speed; vy = 0; }
            else if (side === 1) { x = 620; y = Math.random() * 600; vx = -speed; vy = 0; }
            else if (side === 2) { x = Math.random() * 600; y = -20; vx = 0; vy = speed; }
            else { x = Math.random() * 600; y = 620; vx = 0; vy = -speed; }

            spawnObstacle(x, y, vx, vy);
        }

        function showResult() {
            gameActive = false;
            if (score > bestScore) {
                bestScore = score;
                localStorage.setItem('hexagon_best', bestScore);
            }
            statsEl.innerHTML = `SCORE: ${score}<br>BEST: ${bestScore}`;
            resultEl.style.display = 'flex';
            
            shareBtn.onclick = () => {
                const text = `HEXAGON HELL を突破！\nSCORE: ${score}\nPHASE: ${phase}\nBEST: ${bestScore}\n#HEXAGON_HELL`;
                const url = `https://twitter.com/intent/tweet?text=${encodeURIComponent(text)}`;
                window.open(url, '_blank');
            };
        }

        function update() {
            if (!gameActive) return;

            if (score > 25 && phase === 1) { phase = 2; phaseEl.innerText = "PHASE: 2 (normal)"; }
            if (score > 50 && phase === 2) { phase = 3; phaseEl.innerText = "PHASE: 3 (hard)"; }
            if (score > 100 && phase === 3) { phase = 4; phaseEl.innerText = "PHASE: 4 (GLITCH ABYSS)"; }

            angle += rotationDir;
            const px = centerX + Math.cos(angle) * orbitRadius;
            const py = centerY + Math.sin(angle) * orbitRadius;

            let spawnRate = phase === 1 ? 0.03 : 0.04 + (phase * 0.015);
            if (Math.random() < spawnRate) createEnemy();

            for (let i = obstacles.length - 1; i >= 0; i--) {
                let ob = obstacles[i];
                ob.x += ob.vx;
                ob.y += ob.vy;

                // スコア貯めやすく：かすりボーナス
                const distToPlayer = Math.hypot(px - ob.x, py - ob.y);
                if (distToPlayer < 40 && distToPlayer > 22) {
                    score += 0.1; // ニアミスでスコア加算
                    scoreEl.innerText = Math.floor(score);
                }

                if (phase >= 4 && ob.splitCount < 2) {
                    const distToCenter = Math.hypot(ob.x - centerX, ob.y - centerY);
                    const triggerDist = ob.splitCount === 0 ? 250 : 130;
                    if (distToCenter < triggerDist) {
                        const s = Math.hypot(ob.vx, ob.vy) * 1.2;
                        const c = `hsl(${Math.random()*360}, 100%, 50%)`;
                        spawnObstacle(ob.x, ob.y, s, 0, c, ob.splitCount + 1);
                        spawnObstacle(ob.x, ob.y, -s, 0, c, ob.splitCount + 1);
                        spawnObstacle(ob.x, ob.y, 0, s, c, ob.splitCount + 1);
                        spawnObstacle(ob.x, ob.y, 0, -s, c, ob.splitCount + 1);
                        obstacles.splice(i, 1);
                        continue;
                    }
                }

                if (distToPlayer < 22) {
                    shakeTime = 30;
                    showResult();
                }

                if (ob.x < -150 || ob.x > 750 || ob.y < -150 || ob.y > 750) {
                    obstacles.splice(i, 1);
                    score += 1;
                    scoreEl.innerText = Math.floor(score);
                    if (Math.floor(score) % 10 === 0) shakeTime = 5;
                }
            }

            let sx = 0, sy = 0;
            if (shakeTime > 0) {
                sx = (Math.random() - 0.5) * shakeTime;
                sy = (Math.random() - 0.5) * shakeTime;
                shakeTime--;
            }
            draw(px, py, sx, sy);
        }

        function draw(px, py, sx, sy) {
            ctx.setTransform(1, 0, 0, 1, sx, sy);
            
            // PHASE 4 の背景演出
            let bgAlpha = 0.3 - (phase * 0.05);
            if (phase >= 4 && Math.random() < 0.05) bgAlpha = 0.8; // ストロボ効果
            ctx.fillStyle = `rgba(0, 0, 0, ${bgAlpha})`;
            ctx.fillRect(-100, -100, canvas.width + 200, canvas.height + 200);

            if (phase >= 3 && Math.random() < 0.1) {
                ctx.fillStyle = 'rgba(255, 0, 85, 0.2)';
                ctx.fillRect(0, Math.random() * 600, 600, 5);
            }

            ctx.strokeStyle = phase >= 4 ? '#0ff' : '#333';
            ctx.setLineDash([5, 5]);
            ctx.beginPath();
            ctx.arc(centerX, centerY, orbitRadius, 0, Math.PI * 2);
            ctx.stroke();

            obstacles.forEach(ob => {
                ctx.fillStyle = ob.color;
                ctx.shadowBlur = phase >= 2 ? 15 : 0;
                ctx.shadowColor = ob.color;
                ctx.fillRect(ob.x - 12, ob.y - 12, 24, 24);
            });

            ctx.fillStyle = '#fff';
            ctx.shadowBlur = 20;
            ctx.shadowColor = '#0ff';
            ctx.beginPath();
            ctx.arc(px, py, 12, 0, Math.PI * 2);
            ctx.fill();
            ctx.shadowBlur = 0;

            requestAnimationFrame(update);
        }

        update();
    </script>
</body>
</html>
