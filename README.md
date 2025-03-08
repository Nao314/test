<!DOCTYPE html>
<html>
<head>
    <title>ブロックバランスゲーム</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            background-color: #f0f0f0;
            font-family: Arial, sans-serif;
        }
        canvas {
            border: 2px solid #333;
            background-color: #fff;
        }
        #gameContainer {
            position: relative;
            text-align: center;
        }
        #score, #instructions {
            margin: 10px 0;
        }
    </style>
</head>
<body>
    <div id="gameContainer">
        <div id="score">スコア: 0</div>
        <canvas id="gameCanvas" width="600" height="400"></canvas>
        <div id="instructions">←→キーでブロックを移動。プラットフォームの上にバランスよく積み重ねよう！</div>
    </div>

    <script>
        // ゲーム変数
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreElement = document.getElementById('score');
        
        let score = 0;
        let gameOver = false;
        let gameStarted = false;
        
        // プラットフォーム変数
        const platformWidth = 200;
        const platformHeight = 20;
        let platformX = canvas.width / 2 - platformWidth / 2;
        const platformY = canvas.height - 50;
        let platformAngle = 0;
        const maxAngle = Math.PI / 8; // 最大傾斜角度
        
        // ブロック変数
        const blockSize = 30;
        let blocks = [];
        let currentBlock = null;
        let blockFallSpeed = 2;
        
        // 物理変数
        const gravity = 0.2;
        
        // ゲームループ
        function gameLoop() {
            if (!gameOver) {
                update();
                draw();
                requestAnimationFrame(gameLoop);
            } else {
                drawGameOver();
            }
        }
        
        // ゲーム初期化
        function init() {
            createNewBlock();
            gameStarted = true;
            gameLoop();
        }
        
        // 新しいブロックを作成
        function createNewBlock() {
            currentBlock = {
                x: canvas.width / 2 - blockSize / 2,
                y: 0,
                width: blockSize,
                height: blockSize,
                color: getRandomColor(),
                onPlatform: false,
                velocity: { x: 0, y: 0 },
                angle: 0,
                mass: 1 + Math.random() * 0.5 // ランダムな質量
            };
        }
        
        // ランダムな色を生成
        function getRandomColor() {
            const colors = ['#FF5733', '#33FF57', '#3357FF', '#FF33A8', '#33A8FF', '#A833FF'];
            return colors[Math.floor(Math.random() * colors.length)];
        }
        
        // ゲーム状態の更新
        function update() {
            if (!gameStarted) return;
            
            // 現在のブロックを下に移動
            if (currentBlock && !currentBlock.onPlatform) {
                currentBlock.y += blockFallSpeed;
                
                // プラットフォームとの衝突チェック
                if (checkPlatformCollision(currentBlock)) {
                    handlePlatformCollision();
                }
                
                // 他のブロックとの衝突チェック
                for (let i = 0; i < blocks.length; i++) {
                    if (checkBlockCollision(currentBlock, blocks[i])) {
                        handleBlockCollision(blocks[i]);
                        break;
                    }
                }
                
                // ブロックが画面外に落ちたかチェック
                if (currentBlock.y > canvas.height) {
                    createNewBlock();
                }
            }
            
            // 重量分布に基づいてプラットフォームの角度を更新
            updatePlatformAngle();
            
            // プラットフォーム上のブロックを更新
            updateBlocksOnPlatform();
            
            // ゲームオーバーチェック
            if (Math.abs(platformAngle) >= maxAngle * 0.95) {
                gameOver = true;
            }
        }
        
        // ブロックの位置に基づいてプラットフォームの角度を更新
        function updatePlatformAngle() {
            let leftWeight = 0;
            let rightWeight = 0;
            const platformCenter = platformX + platformWidth / 2;
            
            // 各サイドの重みを計算
            for (let i = 0; i < blocks.length; i++) {
                if (blocks[i].onPlatform) {
                    const blockCenter = blocks[i].x + blocks[i].width / 2;
                    const distanceFromCenter = blockCenter - platformCenter;
                    const torque = distanceFromCenter * blocks[i].mass;
                    
                    if (distanceFromCenter < 0) {
                        leftWeight += Math.abs(torque);
                    } else {
                        rightWeight += Math.abs(torque);
                    }
                }
            }
            
            // 新しい角度を計算
            const netTorque = leftWeight - rightWeight;
            const targetAngle = netTorque * 0.001;
            
            // 角度を徐々に変化させる
            platformAngle += (targetAngle - platformAngle) * 0.1;
            
            // 角度を制限
            platformAngle = Math.max(-maxAngle, Math.min(maxAngle, platformAngle));
        }
        
        // プラットフォーム上のブロックの位置を更新
        function updateBlocksOnPlatform() {
            const platformCenter = platformX + platformWidth / 2;
            
            for (let i = 0; i < blocks.length; i++) {
                if (blocks[i].onPlatform) {
                    // プラットフォームの角度に基づいて新しい位置を計算
                    const blockCenter = blocks[i].x + blocks[i].width / 2;
                    const distanceFromCenter = blockCenter - platformCenter;
                    const heightOffset = Math.sin(platformAngle) * distanceFromCenter;
                    
                    blocks[i].angle = platformAngle;
                    blocks[i].y = platformY - blocks[i].height - heightOffset;
                    
                    // 傾きが大きすぎる場合、ブロックが落下する可能性あり
                    if (Math.abs(platformAngle) > maxAngle * 0.8) {
                        if ((platformAngle > 0 && distanceFromCenter < 0) || 
                            (platformAngle < 0 && distanceFromCenter > 0)) {
                            // 傾きの反対側は安定している
                        } else {
                            // ランダムな確率で落下
                            if (Math.random() < 0.05 * Math.abs(platformAngle) / maxAngle) {
                                blocks[i].onPlatform = false;
                                blocks[i].velocity.y = 1;
                                blocks[i].velocity.x = Math.sign(distanceFromCenter) * 2;
                            }
                        }
                    }
                } else {
                    // 落下中のブロックに重力を適用
                    blocks[i].velocity.y += gravity;
                    blocks[i].x += blocks[i].velocity.x;
                    blocks[i].y += blocks[i].velocity.y;
                    blocks[i].angle += 0.05;
                    
                    // 画面外に落ちたブロックを削除
                    if (blocks[i].y > canvas.height) {
                        blocks.splice(i, 1);
                        i--;
                    }
                }
            }
        }
        
        // ブロックとプラットフォームの衝突をチェック
        function checkPlatformCollision(block) {
            if (block.onPlatform) return false;
            
            const platformLeftX = platformX;
            const platformRightX = platformX + platformWidth;
            const blockBottom = block.y + block.height;
            const blockLeft = block.x;
            const blockRight = block.x + block.width;
            
            // プラットフォームの傾斜を考慮
            const platformLeftY = platformY + Math.sin(platformAngle) * (-platformWidth / 2);
            const platformRightY = platformY + Math.sin(platformAngle) * (platformWidth / 2);
            
            // 線形補間でブロック中心のY座標を計算
            const blockCenterX = (blockLeft + blockRight) / 2;
            const blockRatio = (blockCenterX - platformLeftX) / platformWidth;
            
            if (blockRatio >= 0 && blockRatio <= 1) {
                const platformY = platformLeftY * (1 - blockRatio) + platformRightY * blockRatio;
                
                if (blockBottom >= platformY && blockBottom <= platformY + platformHeight &&
                    blockRight >= platformLeftX && blockLeft <= platformRightX) {
                    return true;
                }
            }
            
            return false;
        }
        
        // プラットフォームとの衝突処理
        function handlePlatformCollision() {
            currentBlock.onPlatform = true;
            
            // プラットフォームの中心からの距離を計算
            const platformCenter = platformX + platformWidth / 2;
            const blockCenter = currentBlock.x + currentBlock.width / 2;
            
            // プラットフォームの傾斜に合わせてブロックを配置
            const distanceFromCenter = blockCenter - platformCenter;
            const heightOffset = Math.sin(platformAngle) * distanceFromCenter;
            currentBlock.y = platformY - currentBlock.height - heightOffset;
            
            blocks.push(currentBlock);
            score++;
            scoreElement.textContent = `スコア: ${score}`;
            createNewBlock();
            
            // 難易度を増加
            if (score % 5 === 0) {
                blockFallSpeed += 0.2;
            }
        }
        
        // 2つのブロック間の衝突をチェック
        function checkBlockCollision(block1, block2) {
            if (block1.onPlatform || block1 === block2) return false;
            
            return (block1.x < block2.x + block2.width &&
                   block1.x + block1.width > block2.x &&
                   block1.y + block1.height > block2.y &&
                   block1.y < block2.y + block2.height);
        }
        
        // 他のブロックとの衝突処理
        function handleBlockCollision(otherBlock) {
            currentBlock.onPlatform = true;
            
            // 積み重ねる位置を計算
            currentBlock.y = otherBlock.y - currentBlock.height;
            currentBlock.angle = otherBlock.angle;
            
            blocks.push(currentBlock);
            score++;
            scoreElement.textContent = `スコア: ${score}`;
            createNewBlock();
        }
        
        // ゲーム要素を描画
        function draw() {
            // キャンバスをクリア
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // 背景を描画
            drawBackground();
            
            // プラットフォームを描画
            ctx.save();
            ctx.translate(platformX + platformWidth / 2, platformY + platformHeight / 2);
            ctx.rotate(platformAngle);
            ctx.fillStyle = '#8B4513';
            ctx.fillRect(-platformWidth / 2, -platformHeight / 2, platformWidth, platformHeight);
            
            // プラットフォームの支柱
            ctx.fillStyle = '#654321';
            ctx.fillRect(-5, -platformHeight / 2, 10, 40);
            ctx.restore();
            
            // ブロックを描画
            for (let i = 0; i < blocks.length; i++) {
                drawBlock(blocks[i]);
            }
            
            // 現在落下中のブロックを描画
            if (currentBlock && !currentBlock.onPlatform) {
                drawBlock(currentBlock);
            }
            
            // ゲーム開始メッセージを描画
            if (!gameStarted) {
                drawStartScreen();
            }
        }
        
        // 背景を描画
        function drawBackground() {
            // 空
            const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
            gradient.addColorStop(0, '#87CEEB');
            gradient.addColorStop(1, '#E0F7FF');
            ctx.fillStyle = gradient;
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            
            // 地面
            ctx.fillStyle = '#8FBC8F';
            ctx.fillRect(0, platformY + 30, canvas.width, canvas.height - platformY - 30);
        }
        
        // ブロックを描画
        function drawBlock(block) {
            ctx.save();
            if (block.onPlatform) {
                // プラットフォーム上のブロックを回転
                ctx.translate(block.x + block.width / 2, block.y + block.height / 2);
                ctx.rotate(block.angle);
                ctx.fillStyle = block.color;
                ctx.fillRect(-block.width / 2, -block.height / 2, block.width, block.height);
                
                // ブロックの枠線
                ctx.strokeStyle = 'rgba(0, 0, 0, 0.3)';
                ctx.lineWidth = 2;
                ctx.strokeRect(-block.width / 2, -block.height / 2, block.width, block.height);
            } else {
                // 落下中のブロックを描画
                ctx.translate(block.x + block.width / 2, block.y + block.height / 2);
                ctx.rotate(block.angle);
                ctx.fillStyle = block.color;
                ctx.fillRect(-block.width / 2, -block.height / 2, block.width, block.height);
                
                // ブロックの枠線
                ctx.strokeStyle = 'rgba(0, 0, 0, 0.3)';
                ctx.lineWidth = 2;
                ctx.strokeRect(-block.width / 2, -block.height / 2, block.width, block.height);
            }
            ctx.restore();
        }
        
        // 開始画面を描画
        function drawStartScreen() {
            ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            
            ctx.fillStyle = '#fff';
            ctx.font = '36px Arial, sans-serif';
            ctx.textAlign = 'center';
            ctx.fillText('ブロックバランスゲーム', canvas.width / 2, canvas.height / 2 - 40);
            
            ctx.font = '24px Arial, sans-serif';
            ctx.fillText('←→キーでブロックを移動', canvas.width / 2, canvas.height / 2 + 20);
            ctx.fillText('バランスを保ちながら積み重ねよう', canvas.width / 2, canvas.height / 2 + 60);
            
            ctx.font = '18px Arial, sans-serif';
            ctx.fillText('何かキーを押してスタート', canvas.width / 2, canvas.height / 2 + 120);
        }
        
        // ゲームオーバー画面を描画
        function drawGameOver() {
            ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            
            ctx.fillStyle = '#fff';
            ctx.font = '36px Arial, sans-serif';
            ctx.textAlign = 'center';
            ctx.fillText('ゲームオーバー', canvas.width / 2, canvas.height / 2 - 20);
            
            ctx.font = '24px Arial, sans-serif';
            ctx.fillText(`最終スコア: ${score}`, canvas.width / 2, canvas.height / 2 + 20);
            
            ctx.font = '18px Arial, sans-serif';
            ctx.fillText('何かキーを押して再プレイ', canvas.width / 2, canvas.height / 2 + 60);
        }
        
        // キーボード入力の処理
        document.addEventListener('keydown', function(event) {
            if (!gameStarted) {
                init();
                return;
            }
            
            if (gameOver) {
                resetGame();
                return;
            }
            
            if (currentBlock && !currentBlock.onPlatform) {
                if (event.key === 'ArrowLeft') {
                    currentBlock.x -= 10;
                    if (currentBlock.x < 0) {
                        currentBlock.x = 0;
                    }
                } else if (event.key === 'ArrowRight') {
                    currentBlock.x += 10;
                    if (currentBlock.x + currentBlock.width > canvas.width) {
                        currentBlock.x = canvas.width - currentBlock.width;
                    }
                } else if (event.key === 'ArrowDown') {
                    // 落下速度を一時的に上げる
                    blockFallSpeed = 8;
                }
            }
        });
        
        // キーを離した時の処理
        document.addEventListener('keyup', function(event) {
            if (event.key === 'ArrowDown') {
                // 通常の落下速度に戻す
                blockFallSpeed = 2 + Math.floor(score / 5) * 0.2;
            }
        });
        
        // ゲームをリセット
        function resetGame() {
            score = 0;
            gameOver = false;
            gameStarted = false;
            platformAngle = 0;
            blocks = [];
            blockFallSpeed = 2;
            scoreElement.textContent = `スコア: ${score}`;
            init();
        }
        
        // 初期画面を描画
        draw();
    </script>
</body>
</html>
