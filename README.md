<!DOCTYPE html>
<html>
<head>
    <title>バランスキーパー</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
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
            touch-action: none;
            overflow: hidden;
        }
        #gameContainer {
            position: relative;
            width: 100%;
            max-width: 600px;
            height: 100vh;
            max-height: 800px;
            background-color: #87CEEB;
            overflow: hidden;
            touch-action: none;
        }
        canvas {
            display: block;
            background-color: transparent;
            touch-action: none;
        }
        #scoreDisplay, #levelDisplay, #missDisplay {
            position: absolute;
            top: 10px;
            font-size: 20px;
            color: white;
            text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.8);
            z-index: 10;
        }
        #scoreDisplay {
            left: 10px;
        }
        #levelDisplay {
            right: 10px;
        }
        #missDisplay {
            left: 50%;
            transform: translateX(-50%);
            transition: color 0.3s, font-weight 0.3s;
        }
        #controlsOverlay {
            position: absolute;
            bottom: 0;
            left: 0;
            width: 100%;
            height: 100px;
            display: flex;
            justify-content: space-between;
            z-index: 10;
            opacity: 0.3;
        }
        .controlButton {
            width: 50%;
            height: 100%;
            display: flex;
            justify-content: center;
            align-items: center;
            background-color: rgba(255, 255, 255, 0.2);
            border: 1px solid rgba(255, 255, 255, 0.5);
        }
        .controlButton div {
            font-size: 30px;
            color: white;
        }
        #messageOverlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            background-color: rgba(0, 0, 0, 0.7);
            color: white;
            z-index: 20;
        }
        #messageText {
            font-size: 36px;
            margin-bottom: 20px;
            text-align: center;
            padding: 0 20px;
        }
        #startButton, #restartButton {
            padding: 15px 30px;
            font-size: 24px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 10px;
            cursor: pointer;
            margin-top: 20px;
            position: relative;
            z-index: 30;
            user-select: none;
            -webkit-user-select: none;
            -webkit-tap-highlight-color: transparent;
            touch-action: manipulation;
        }
        #startButton:hover, #restartButton:hover {
            background-color: #45a049;
        }
        #startButton:active, #restartButton:active {
            background-color: #367c39;
        }
    </style>
</head>
<body>
    <div id="gameContainer">
        <div id="scoreDisplay">スコア: 0</div>
        <div id="levelDisplay">レベル: 1</div>
        <div id="missDisplay">ミス: 0 / 5</div>
        <canvas id="gameCanvas"></canvas>
        
        <div id="controlsOverlay">
            <div id="leftControl" class="controlButton">
                <div>←</div>
            </div>
            <div id="rightControl" class="controlButton">
                <div>→</div>
            </div>
        </div>
        
        <div id="messageOverlay">
            <div id="messageText">バランスキーパー</div>
            <div style="font-size: 18px; text-align: center; padding: 0 20px;">
                台を左右に動かして、落ちてくるブロックをキャッチしよう！<br>
                バランスを崩さないように注意！<br>
                ブロックを5回落とすとゲームオーバー！
            </div>
            <button id="startButton">ゲームスタート</button>
        </div>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', function() {
            // スコア更新用のヘルパー関数（一貫した更新を行うため）
            function updateScore(points) {
                const oldScore = score;
                score += points;
                score = Math.max(0, score); // 0未満にならないよう保証
                scoreDisplay.textContent = `スコア: ${score}`;
                console.log(`Score updated: ${oldScore} → ${score} (changed by ${points})`);
            }
            
            // ゲーム変数
            let canvas = document.getElementById('gameCanvas');
            let ctx = canvas.getContext('2d');
            let scoreDisplay = document.getElementById('scoreDisplay');
            let levelDisplay = document.getElementById('levelDisplay');
            let missDisplay = document.getElementById('missDisplay');
            let messageOverlay = document.getElementById('messageOverlay');
            let messageText = document.getElementById('messageText');
            let startButton = document.getElementById('startButton');
            let leftControl = document.getElementById('leftControl');
            let rightControl = document.getElementById('rightControl');
            
            let score = 0;
            let level = 1;
            let gameOver = false;
            let gameRunning = false;
            let animationFrameId = null;
            let platformWidth;
            let platformHeight;
            let platformY;
            let platformX;
            let platformSpeed = 10;
            let platformAngle = 0;
            const maxAngle = Math.PI / 6;
            let leftPressed = false;
            let rightPressed = false;
            let blocks = [];
            const blockColors = ['#FF5733', '#33FF57', '#3357FF', '#FF33A8', '#33A8FF', '#A833FF', '#FFFF33', '#33FFFF'];
            let gravity = 0.3;  // レベルアップ時に更新するため let を使用
            let lastSpawnTime = 0;
            let spawnDelay = 2000;
            // ブロックを落とした回数
            let droppedBlocksCount = 0;
            // 許容できる落下回数
            const maxDroppedBlocks = 5;
            
            // キャンバスのサイズ設定
            function resizeCanvas() {
                const gameContainer = document.getElementById('gameContainer');
                canvas.width = gameContainer.clientWidth;
                canvas.height = gameContainer.clientHeight;
                
                // ゲーム内の要素のサイズを更新
                platformWidth = Math.min(canvas.width * 0.3, 180);
                platformHeight = canvas.height * 0.02;
                platformY = canvas.height * 0.85;
                platformX = canvas.width / 2 - platformWidth / 2;
            }
            
            window.addEventListener('resize', resizeCanvas);
            resizeCanvas();
            
            // ゲーム開始
            function startGame() {
                console.log('Game starting...');
                
                // ゲーム変数をリセット
                score = 0;
                level = 1;
                droppedBlocksCount = 0;
                gameOver = false;
                gameRunning = true;
                blocks = [];
                platformAngle = 0;
                platformX = canvas.width / 2 - platformWidth / 2;
                gravity = 0.3; // 初期の重力にリセット
                
                // スコアとレベルの表示を更新
                scoreDisplay.textContent = `スコア: ${score}`;
                levelDisplay.textContent = `レベル: ${level}`;
                missDisplay.textContent = `ミス: ${droppedBlocksCount} / ${maxDroppedBlocks}`;
                
                // メッセージオーバーレイを非表示
                messageOverlay.style.display = 'none';
                
                // 前回のゲームループをキャンセル
                if (animationFrameId !== null) {
                    cancelAnimationFrame(animationFrameId);
                    animationFrameId = null;
                }
                
                // ゲームループを開始
                lastSpawnTime = Date.now();
                gameLoop();
            }
            
            // ゲームループ
            function gameLoop() {
                if (!gameRunning) return;
                
                try {
                    // キー入力に基づいて台を移動
                    if (leftPressed) {
                        movePlatform(-1);
                    }
                    if (rightPressed) {
                        movePlatform(1);
                    }
                    
                    // 現在時刻を取得
                    const currentTime = Date.now();
                    
                    // 一定間隔でブロックを生成
                    if (currentTime - lastSpawnTime > spawnDelay) {
                        spawnBlock();
                        lastSpawnTime = currentTime;
                        // レベルに応じてスポーン間隔を調整
                        spawnDelay = 2000 - (level * 100);
                        spawnDelay = Math.max(spawnDelay, 500); // 最小スポーン間隔
                    }
                    
                    // ゲーム状態を更新
                    update();
                    
                    // 描画
                    draw();
                    
                    // レベルアップのチェック
                    checkLevelUp();
                    
                    // ゲームオーバーでない場合は次のフレームを要求
                    if (!gameOver) {
                        animationFrameId = requestAnimationFrame(gameLoop);
                    } else {
                        showGameOver();
                    }
                } catch (err) {
                    console.error('Error in game loop:', err);
                    if (!gameOver && gameRunning) {
                        animationFrameId = requestAnimationFrame(gameLoop);
                    }
                }
            }
            
            // レベルアップ条件チェック
            function checkLevelUp() {
                const requiredScore = level * 50;
                if (score >= requiredScore) {
                    console.log(`Level up check: score=${score}, required=${requiredScore}, level=${level}`);
                    if (level * 50 <= score) {
                        levelUp();
                    }
                }
            }
            
            // ブロック生成
            function spawnBlock() {
                const minSize = Math.min(canvas.width, canvas.height) * 0.05;
                const maxSize = Math.min(canvas.width, canvas.height) * 0.1;
                const size = minSize + Math.random() * (maxSize - minSize);
                const x = Math.random() * (canvas.width - size);
                const color = blockColors[Math.floor(Math.random() * blockColors.length)];
                const mass = size / minSize;
                
                blocks.push({
                    x: x,
                    y: -size,
                    width: size,
                    height: size,
                    color: color,
                    onPlatform: false,
                    resting: false,
                    velocity: { x: 0, y: 1 + Math.random() * level * 0.5 },
                    angle: 0,
                    mass: mass,
                    restingOn: null,
                    isStable: false
                });
            }
            
            // 台の移動
            function movePlatform(direction) {
                const oldPlatformX = platformX;
                const moveAmount = direction * platformSpeed;
                const newPlatformX = platformX + moveAmount;
                platformX = Math.max(0, Math.min(newPlatformX, canvas.width - platformWidth));
                const actualMoveAmount = platformX - oldPlatformX;
                if (actualMoveAmount !== 0) {
                    for (let i = 0; i < blocks.length; i++) {
                        if (blocks[i].onPlatform) {
                            blocks[i].x += actualMoveAmount;
                        } else if (blocks[i].restingOn !== null) {
                            let shouldMove = false;
                            let currentBlock = blocks[i];
                            while (currentBlock.restingOn !== null) {
                                const restingIndex = blocks.findIndex(b => b === currentBlock.restingOn);
                                if (restingIndex !== -1) {
                                    currentBlock = blocks[restingIndex];
                                    if (currentBlock.onPlatform) {
                                        shouldMove = true;
                                        break;
                                    }
                                } else {
                                    break;
                                }
                            }
                            if (shouldMove) {
                                blocks[i].x += actualMoveAmount;
                            }
                        }
                    }
                }
            }
            
            // ゲーム状態更新
            function update() {
                for (let i = 0; i < blocks.length; i++) {
                    const block = blocks[i];
                    
                    if (!block.resting) {
                        block.velocity.y += gravity;
                        block.x += block.velocity.x;
                        block.y += block.velocity.y;
                        
                        if (block.x < 0) {
                            block.x = 0;
                            block.velocity.x = -block.velocity.x * 0.5;
                        } else if (block.x + block.width > canvas.width) {
                            block.x = canvas.width - block.width;
                            block.velocity.x = -block.velocity.x * 0.5;
                        }
                        
                        checkPlatformCollision(block);
                        
                        for (let j = 0; j < blocks.length; j++) {
                            if (i !== j && blocks[j].resting) {
                                checkBlockCollision(block, blocks[j]);
                            }
                        }
                        
                        // ブロックが画面下に落ちた場合
                        if (block.y > canvas.height) {
                            blocks.splice(i, 1);
                            i--;
                            
                            // ミスカウント更新
                            droppedBlocksCount++;
                            missDisplay.textContent = `ミス: ${droppedBlocksCount} / ${maxDroppedBlocks}`;
                            
                            missDisplay.style.color = 'red';
                            missDisplay.style.fontWeight = 'bold';
                            setTimeout(() => {
                                missDisplay.style.color = 'white';
                                missDisplay.style.fontWeight = 'normal';
                            }, 500);
                            
                            // 直接のスコア更新はせず、updateScoreのみで行う
                            updateScore(-5);
                            
                            if (droppedBlocksCount >= maxDroppedBlocks) {
                                gameOver = true;
                            }
                        }
                    }
                }
                
                updatePlatformAngle();
                checkUnstableBlocks();
                
                if (Math.abs(platformAngle) >= maxAngle * 0.95) {
                    gameOver = true;
                }
            }
            
            // 不安定なブロックをチェック
            function checkUnstableBlocks() {
                for (let i = 0; i < blocks.length; i++) {
                    if (blocks[i].resting && !blocks[i].isStable) {
                        const xCenter = blocks[i].x + blocks[i].width / 2;
                        let supportFound = false;
                        
                        if (blocks[i].onPlatform) {
                            if (xCenter >= platformX && xCenter <= platformX + platformWidth) {
                                supportFound = true;
                            }
                        } else if (blocks[i].restingOn !== null) {
                            const supportBlock = blocks[i].restingOn;
                            if (xCenter >= supportBlock.x && xCenter <= supportBlock.x + supportBlock.width) {
                                supportFound = true;
                            }
                        }
                        
                        if (!supportFound) {
                            blocks[i].resting = false;
                            blocks[i].restingOn = null;
                            blocks[i].velocity.y = 0.5;
                            blocks[i].velocity.x = (Math.random() - 0.5) * 2;
                        } else {
                            blocks[i].isStable = true;
                        }
                    }
                }
            }
            
            // プラットフォームとの衝突チェック
            function checkPlatformCollision(block) {
                const platformLeftY = platformY - Math.sin(platformAngle) * (platformWidth / 2);
                const platformRightY = platformY + Math.sin(platformAngle) * (platformWidth / 2);
                const blockCenterX = block.x + block.width / 2;
                const blockLeft = block.x;
                const blockRight = block.x + block.width;
                
                if (blockRight >= platformX && blockLeft <= platformX + platformWidth) {
                    const contactX = Math.max(platformX, Math.min(platformX + platformWidth, blockCenterX));
                    const ratio = (contactX - platformX) / platformWidth;
                    const platformYAtBlock = platformLeftY * (1 - ratio) + platformRightY * ratio;
                    const blockBottom = block.y + block.height;
                    
                    if (blockBottom >= platformYAtBlock - 5 && 
                        blockBottom <= platformYAtBlock + platformHeight + block.velocity.y + 5 && 
                        block.velocity.y > 0) {
                        
                        block.y = platformYAtBlock - block.height;
                        block.velocity.y = 0;
                        block.velocity.x = 0;
                        block.resting = true;
                        block.onPlatform = true;
                        block.restingOn = null;
                        
                        updateScore(10);
                        return true;
                    }
                }
                return false;
            }
            
            // ブロック同士の衝突チェック
            function checkBlockCollision(movingBlock, staticBlock) {
                const movingCenter = {
                    x: movingBlock.x + movingBlock.width / 2,
                    y: movingBlock.y + movingBlock.height / 2
                };
                const staticCenter = {
                    x: staticBlock.x + staticBlock.width / 2,
                    y: staticBlock.y + staticBlock.height / 2
                };
                const dx = Math.abs(movingCenter.x - staticCenter.x);
                const dy = Math.abs(movingCenter.y - staticCenter.y);
                const halfWidthSum = (movingBlock.width + staticBlock.width) / 2;
                const halfHeightSum = (movingBlock.height + staticBlock.height) / 2;
                
                if (dx < halfWidthSum && dy < halfHeightSum) {
                    if (movingBlock.y + movingBlock.height <= staticBlock.y + 20 && 
                        movingBlock.velocity.y > 0) {
                        
                        movingBlock.y = staticBlock.y - movingBlock.height;
                        movingBlock.velocity.y = 0;
                        movingBlock.velocity.x = 0;
                        movingBlock.resting = true;
                        movingBlock.onPlatform = false;
                        movingBlock.restingOn = staticBlock;
                        
                        updateScore(5);
                        return true;
                    } else if (movingBlock.y + movingBlock.height > staticBlock.y + 20) {
                        movingBlock.velocity.x = -movingBlock.velocity.x * 0.5;
                        if (movingCenter.x < staticCenter.x) {
                            movingBlock.x = staticBlock.x - movingBlock.width;
                        } else {
                            movingBlock.x = staticBlock.x + staticBlock.width;
                        }
                        return true;
                    }
                }
                return false;
            }
            
            // 台の角度更新
            function updatePlatformAngle() {
                let leftWeight = 0;
                let rightWeight = 0;
                const platformCenter = platformX + platformWidth / 2;
                
                for (let i = 0; i < blocks.length; i++) {
                    if (blocks[i].resting) {
                        const blockCenter = blocks[i].x + blocks[i].width / 2;
                        const distanceFromCenter = blockCenter - platformCenter;
                        let isOnPlatform = false;
                        
                        if (blocks[i].onPlatform) {
                            isOnPlatform = true;
                        } else {
                            let currentBlock = blocks[i];
                            while (currentBlock.restingOn !== null) {
                                const restingIndex = blocks.findIndex(b => b === currentBlock.restingOn);
                                if (restingIndex !== -1) {
                                    currentBlock = blocks[restingIndex];
                                    if (currentBlock.onPlatform) {
                                        isOnPlatform = true;
                                        break;
                                    }
                                } else {
                                    break;
                                }
                            }
                        }
                        
                        if (isOnPlatform) {
                            const torque = distanceFromCenter * blocks[i].mass;
                            if (distanceFromCenter < 0) {
                                leftWeight += Math.abs(torque);
                            } else {
                                rightWeight += Math.abs(torque);
                            }
                        }
                    }
                }
                
                const weightDifference = rightWeight - leftWeight;
                const targetAngle = (weightDifference * 0.0005) * (1 + level * 0.1);
                platformAngle += (targetAngle - platformAngle) * 0.1;
                platformAngle = Math.max(-maxAngle, Math.min(maxAngle, platformAngle));
            }
            
            // レベルアップ
            function levelUp() {
                console.log(`Executing levelUp: current level=${level}, score=${score}`);
                const requiredScore = level * 50;
                if (score < requiredScore) {
                    console.log('Score too low for level up, returning');
                    return;
                }
                const newLevel = level + 1;
                console.log(`Increasing level from ${level} to ${newLevel}`);
                level = newLevel;
                levelDisplay.textContent = `レベル: ${level}`;
                
                platformSpeed = 10 + level;
                gravity = 0.3 + level * 0.05;
                
                try {
                    const levelUpMsg = document.createElement('div');
                    levelUpMsg.textContent = `レベル ${level}！`;
                    levelUpMsg.style.position = 'absolute';
                    levelUpMsg.style.top = '50%';
                    levelUpMsg.style.left = '50%';
                    levelUpMsg.style.transform = 'translate(-50%, -50%)';
                    levelUpMsg.style.color = 'yellow';
                    levelUpMsg.style.fontSize = '48px';
                    levelUpMsg.style.fontWeight = 'bold';
                    levelUpMsg.style.textShadow = '2px 2px 4px rgba(0, 0, 0, 0.5)';
                    levelUpMsg.style.zIndex = '15';
                    levelUpMsg.style.pointerEvents = 'none';
                    levelUpMsg.style.opacity = '1';
                    
                    const gameContainer = document.getElementById('gameContainer');
                    if (gameContainer) {
                        gameContainer.appendChild(levelUpMsg);
                        setTimeout(function() {
                            try {
                                levelUpMsg.style.transition = 'opacity 0.5s';
                                levelUpMsg.style.opacity = '0';
                                setTimeout(function() {
                                    try {
                                        if (levelUpMsg.parentNode) {
                                            levelUpMsg.parentNode.removeChild(levelUpMsg);
                                        }
                                    } catch (err) {
                                        console.error('Error removing level up message:', err);
                                    }
                                }, 500);
                            } catch (err) {
                                console.error('Error fading out level up message:', err);
                                if (levelUpMsg.parentNode) {
                                    levelUpMsg.parentNode.removeChild(levelUpMsg);
                                }
                            }
                        }, 1000);
                    }
                    
                    updateScore(20);
                    console.log(`Level up complete. New level: ${level}, score: ${score}`);
                } catch (err) {
                    console.error('Error in levelUp function:', err);
                }
            }
            
            // 描画
            function draw() {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
                gradient.addColorStop(0, '#6495ED');
                gradient.addColorStop(1, '#87CEEB');
                ctx.fillStyle = gradient;
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                
                const edgeMarkerWidth = 5;
                ctx.fillStyle = 'rgba(255, 255, 255, 0.3)';
                ctx.fillRect(0, 0, edgeMarkerWidth, canvas.height);
                ctx.fillRect(canvas.width - edgeMarkerWidth, 0, edgeMarkerWidth, canvas.height);
                
                ctx.fillStyle = '#8B4513';
                ctx.beginPath();
                ctx.moveTo(platformX + platformWidth / 2 - 5, platformY);
                ctx.lineTo(platformX + platformWidth / 2 + 5, platformY);
                ctx.lineTo(platformX + platformWidth / 2 + 10, canvas.height);
                ctx.lineTo(platformX + platformWidth / 2 - 10, canvas.height);
                ctx.closePath();
                ctx.fill();
                
                ctx.save();
                ctx.translate(platformX + platformWidth / 2, platformY);
                ctx.rotate(platformAngle);
                ctx.fillStyle = '#A0522D';
                ctx.fillRect(-platformWidth / 2, -platformHeight / 2, platformWidth, platformHeight);
                ctx.strokeStyle = 'rgba(0, 0, 0, 0.3)';
                ctx.lineWidth = 2;
                ctx.strokeRect(-platformWidth / 2, -platformHeight / 2, platformWidth, platformHeight);
                ctx.restore();
                
                for (let i = 0; i < blocks.length; i++) {
                    const block = blocks[i];
                    ctx.save();
                    ctx.translate(block.x + block.width / 2, block.y + block.height / 2);
                    if (block.onPlatform) {
                        ctx.rotate(platformAngle);
                    } else if (block.resting && block.restingOn !== null) {
                        ctx.rotate(block.restingOn.angle || 0);
                    } else {
                        block.angle += 0.01 * block.velocity.y;
                        ctx.rotate(block.angle);
                    }
                    ctx.fillStyle = block.color;
                    ctx.fillRect(-block.width / 2, -block.height / 2, block.width, block.height);
                    ctx.strokeStyle = 'rgba(0, 0, 0, 0.3)';
                    ctx.lineWidth = 2;
                    ctx.strokeRect(-block.width / 2, -block.height / 2, block.width, block.height);
                    ctx.restore();
                }
                
                const angleIndicatorWidth = 100;
                const angleIndicatorHeight = 20;
                const angleIndicatorX = canvas.width / 2 - angleIndicatorWidth / 2;
                const angleIndicatorY = 50;
                ctx.fillStyle = 'rgba(0, 0, 0, 0.3)';
                ctx.fillRect(angleIndicatorX, angleIndicatorY, angleIndicatorWidth, angleIndicatorHeight);
                const tiltPercentage = (platformAngle / maxAngle) * 0.5 + 0.5;
                const tiltWidth = tiltPercentage * angleIndicatorWidth;
                let tiltColor;
                if (tiltPercentage > 0.7 || tiltPercentage < 0.3) {
                    tiltColor = '#FF5733';
                } else if (tiltPercentage > 0.6 || tiltPercentage < 0.4) {
                    tiltColor = '#FFC300';
                } else {
                    tiltColor = '#33FF57';
                }
                ctx.fillStyle = tiltColor;
                ctx.fillRect(angleIndicatorX, angleIndicatorY, tiltWidth, angleIndicatorHeight);
                ctx.fillStyle = 'white';
                ctx.fillRect(angleIndicatorX + angleIndicatorWidth / 2 - 1, angleIndicatorY, 2, angleIndicatorHeight);
                
                try {
                    const progressBarWidth = 200;
                    const progressBarHeight = 10;
                    const progressBarX = canvas.width / 2 - progressBarWidth / 2;
                    const progressBarY = 80;
                    ctx.fillStyle = 'rgba(0, 0, 0, 0.3)';
                    ctx.fillRect(progressBarX, progressBarY, progressBarWidth, progressBarHeight);
                    const nextLevelScore = level * 50;
                    const previousLevelScore = (level - 1) * 50;
                    let progressPercentage = 0;
                    const levelDiff = nextLevelScore - previousLevelScore;
                    if (levelDiff > 0) {
                        const currentLevelProgress = score - previousLevelScore;
                        progressPercentage = Math.min(1, Math.max(0, currentLevelProgress / levelDiff));
                    }
                    const progressWidth = progressPercentage * progressBarWidth;
                    ctx.fillStyle = '#4CAF50';
                    ctx.fillRect(progressBarX, progressBarY, progressWidth, progressBarHeight);
                    ctx.fillStyle = 'white';
                    ctx.font = '12px Arial';
                    ctx.textAlign = 'center';
                    const pointsToNextLevel = Math.max(0, nextLevelScore - score);
                    if (pointsToNextLevel > 0) {
                        ctx.fillText(`次のLv.${level+1}まであと${pointsToNextLevel}点`, canvas.width / 2, progressBarY + 25);
                    } else {
                        ctx.fillText(`Lv.${level+1}に上がる準備完了！`, canvas.width / 2, progressBarY + 25);
                    }
                } catch (err) {
                    console.error('Error drawing progress bar:', err);
                }
            }
            
            // ゲームオーバー表示
            function showGameOver() {
                gameRunning = false;
                messageOverlay.style.display = 'flex';
                while (messageOverlay.children.length > 3) {
                    messageOverlay.removeChild(messageOverlay.lastChild);
                }
                messageText.textContent = 'ゲームオーバー';
                const reasonText = droppedBlocksCount >= maxDroppedBlocks
                    ? `ブロックを${maxDroppedBlocks}回落としてしまいました`
                    : 'バランスが崩れてしまいました';
                const reasonElement = document.createElement('div');
                reasonElement.textContent = reasonText;
                reasonElement.style.fontSize = '20px';
                reasonElement.style.marginTop = '10px';
                reasonElement.style.marginBottom = '10px';
                messageOverlay.appendChild(reasonElement);
                const restartButton = document.createElement('button');
                restartButton.id = 'restartButton';
                restartButton.textContent = 'もう一度プレイ';
                restartButton.addEventListener('click', startGame);
                messageOverlay.appendChild(restartButton);
                const finalScore = document.createElement('div');
                finalScore.textContent = `最終スコア: ${score}`;
                finalScore.style.fontSize = '24px';
                finalScore.style.marginTop = '20px';
                messageOverlay.appendChild(finalScore);
                if (startButton) {
                    startButton.style.display = 'none';
                }
            }
            
            // イベントリスナーの設定
            startButton.addEventListener('click', startGame);
            leftControl.addEventListener('touchstart', function(e) {
                e.preventDefault();
                leftPressed = true;
            });
            leftControl.addEventListener('touchend', function(e) {
                e.preventDefault();
                leftPressed = false;
            });
            leftControl.addEventListener('mousedown', function() {
                leftPressed = true;
            });
            rightControl.addEventListener('touchstart', function(e) {
                e.preventDefault();
                rightPressed = true;
            });
            rightControl.addEventListener('touchend', function(e) {
                e.preventDefault();
                rightPressed = false;
            });
            rightControl.addEventListener('mousedown', function() {
                rightPressed = true;
            });
            document.addEventListener('mouseup', function() {
                leftPressed = false;
                rightPressed = false;
            });
            document.addEventListener('keydown', function(e) {
                if (e.key === 'ArrowLeft' || e.key === 'a') {
                    leftPressed = true;
                } else if (e.key === 'ArrowRight' || e.key === 'd') {
                    rightPressed = true;
                } else if (e.key === ' ' && !gameRunning) {
                    startGame();
                }
            });
            document.addEventListener('keyup', function(e) {
                if (e.key === 'ArrowLeft' || e.key === 'a') {
                    leftPressed = false;
                } else if (e.key === 'ArrowRight' || e.key === 'd') {
                    rightPressed = false;
                }
            });
            draw();
        });
    </script>
</body>
</html>
