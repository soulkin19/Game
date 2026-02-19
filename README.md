<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>HEXAGON HELL -CHAOS-</title>
    <style>
        * { touch-action: none; -webkit-tap-highlight-color: transparent; }
        body { margin: 0; background: #000; color: #fff; font-family: 'Courier New', monospace; overflow: hidden; display: flex; align-items: center; justify-content: center; height: 100vh; position: fixed; width: 100%; }
        canvas { background: #000; border: 4px solid #333; max-width: 95vw; max-height: 80vh; box-shadow: 0 0 50px rgba(255, 0, 85, 0.3); }
        #ui { position: absolute; top: 5%; text-align: center; pointer-events: none; width: 100%; z-index: 10; }
        .score { font-size: 4rem; font-weight: bold; text-shadow: 0 0 20px #ff0055; margin: 0; }
        .phase { font-size: 1.2rem; color: #ff0055; font-weight: bold; text-transform: uppercase; }
    </style>
</head>
<body>
    <div id="ui">
        <div id="phase" class="phase">PHASE: 1 (easy)</div>
        <div id="score" class="score">0</div>
    </div>
    <canvas id="game"></canvas>

    <script>
        const canvas = document.getElementById('game');
        const ctx = canvas.getContext('2d');
        const scoreEl = document.getElementById('score');
        const phaseEl = document.getElementById('phase');

        canvas.width = 600;
        canvas.height = 600;

        let score = 0;
        let gameActive = true;
        let angle = 0;
        let rotationDir = 0.08;
        let obstacles = [];
        let shakeTime = 0;
        let phase = 1;

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
        window.addEventListener('dblclick', (e) => e.preventDefault(), { passive: false });

        function spawnObstacle(x, y, vx, vy, color = '#ff0055', splitCount = 0) {
            // PHASE 4用の分裂回数(splitCount)を保持
            obstacles.push({ x, y, vx, vy, color, splitCount });
        }

        function createEnemy() {
            const side = Math.floor(Math.random() * 4);
            let x, y, vx, vy;
            
            let baseSpeed;
            if (phase === 1) baseSpeed = 1.5; // さらに簡単に
            else if (phase === 2) baseSpeed = 2.5; // マイルドに
            else if (phase === 3) baseSpeed = 3.5; // 標準的に
            else baseSpeed = 4.0; 

            let speed = (baseSpeed + Math.random() * 1.5) + (phase * 0.5);

            if (side === 0) { x = -20; y = Math.random() * 600; vx = speed; vy = 0; }
            else if (side === 1) { x = 620; y = Math.random() * 600; vx = -speed; vy = 0; }
            else if (side === 2) { x = Math.random() * 600; y = -20; vx = 0; vy = speed; }
            else { x = Math.random() * 600; y = 620; vx = 0; vy = -speed; }

            spawnObstacle(x, y, vx, vy);
        }

        function update() {
            if (!gameActive) return;

            if (score > 15 && phase === 1) { phase = 2; phaseEl.innerText = "PHASE: 2 (normal)"; }
            if (score > 30 && phase === 2) { phase = 3; phaseEl.innerText = "PHASE: 3 (hard)"; }
            if (score > 50 && phase === 3) { phase = 4; phaseEl.innerText = "PHASE: 4 (SPLIT HELL)"; }

            angle += rotationDir;
            const px = centerX + Math.cos(angle) * orbitRadius;
            const py = centerY + Math.sin(angle) * orbitRadius;

            let spawnRate;
            if (phase === 1) spawnRate = 0.03;
            else if (phase === 2) spawnRate = 0.04;
            else spawnRate = 0.04 + (phase * 0.015);

            if (Math.random() < spawnRate) createEnemy();

            for (let i = obstacles.length - 1; i >= 0; i--) {
                let ob = obstacles[i];
                ob.x += ob.vx;
                ob.y += ob.vy;

                // PHASE 4 特有の分裂ロジック (2回まで分裂)
                if (phase >= 4 && ob.splitCount < 2) {
                    const distToCenter = Math.hypot(ob.x - centerX, ob.y - centerY);
                    // 1回目：中央付近で分裂、2回目：軌道の少し外側で分裂
                    const triggerDist = ob.splitCount === 0 ? 250 : 130;
                    
                    if (distToCenter < triggerDist) {
                        const s = Math.hypot(ob.vx, ob.vy);
                        // 4方向に拡散
                        spawnObstacle(ob.x, ob.y, s, 0, '#ffcc00', ob.splitCount + 1);
                        spawnObstacle(ob.x, ob.y, -s, 0, '#ffcc00', ob.splitCount + 1);
                        spawnObstacle(ob.x, ob.y, 0, s, '#ffcc00', ob.splitCount + 1);
                        spawnObstacle(ob.x, ob.y, 0, -s, '#ffcc00', ob.splitCount + 1);
                        
                        obstacles.splice(i, 1);
                        continue;
                    }
                }

                const dist = Math.hypot(px - ob.x, py - ob.y);
                if (dist < 22) {
                    shakeTime = 30;
                    gameActive = false;
                    setTimeout(() => {
                        alert(`Broken!\nFINAL SCORE: ${score}`);
                        location.reload();
                    }, 100);
                }

                if (ob.x < -150 || ob.x > 750 || ob.y < -150 || ob.y > 750) {
                    obstacles.splice(i, 1);
                    score++;
                    scoreEl.innerText = score;
                    if (score % 10 === 0) shakeTime = 5;
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
            ctx.fillStyle = `rgba(0, 0, 0, ${0.3 - (phase * 0.05)})`;
            ctx.fillRect(-100, -100, canvas.width + 200, canvas.height + 200);

            if (phase >= 3 && Math.random() < 0.1) {
                ctx.fillStyle = 'rgba(255, 0, 85, 0.1)';
                ctx.fillRect(0, Math.random() * 600, 600, 2);
            }

            ctx.strokeStyle = '#333';
            ctx.setLineDash([5, 5]);
            ctx.beginPath();
            ctx.arc(centerX, centerY, orbitRadius, 0, Math.PI * 2);
            ctx.stroke();

            obstacles.forEach(ob => {
                ctx.fillStyle = ob.color;
                ctx.shadowBlur = phase >= 2 ? 10 : 0;
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
