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
            const gravity = 0.3;
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
                    // エラーが発生しても続行を試みる
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
                    // レベルアップの前にスコアをチェック（既に処理済みか確認）
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
                
                // ブロックの位置はキャンバス幅内にランダム
                const x = Math.random() * (canvas.width - size);
                
                // 色はランダム
                const color = blockColors[Math.floor(Math.random() * blockColors.length)];
                
                // 質量は大きさに比例
                const mass = size / minSize;
                
                blocks.push({
                    x: x,
                    y: -size, // キャンバスの上から出現
                    width: size,
                    height: size,
                    color: color,
                    onPlatform: false,
                    resting: false,
                    velocity: { x: 0, y: 1 + Math.random() * level * 0.5 }, // レベルに応じて初速を上げる
                    angle: 0,
                    mass: mass,
                    restingOn: null,
                    isStable: false
                });
            }
            
            // 台の移動
            function movePlatform(direction) {
                const oldPlatformX = platformX; // 移動前の位置を保存
                const moveAmount = direction * platformSpeed;
                
                // 移動先の位置を計算
                const newPlatformX = platformX + moveAmount;
                
                // 画面外に出ないように制限
                platformX = Math.max(0, Math.min(newPlatformX, canvas.width - platformWidth));
                
                // 実際の移動量（制限後）
                const actualMoveAmount = platformX - oldPlatformX;
                
                // 実際に移動できた場合のみブロックも移動
                if (actualMoveAmount !== 0) {
                    // 台の上にあるブロックも一緒に移動
                    for (let i = 0; i < blocks.length; i++) {
                        if (blocks[i].onPlatform) {
                            blocks[i].x += actualMoveAmount;
                        } else if (blocks[i].restingOn !== null) {
                            // 他のブロックの上に乗っているブロックも移動
                            let shouldMove = false;
                            let currentBlock = blocks[i];
                            
                            // 下のブロックをたどって、最終的に台の上にあるか確認
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
                // ブロックを更新
                for (let i = 0; i < blocks.length; i++) {
                    const block = blocks[i];
                    
                    if (!block.resting) {
                        // 重力を適用
                        block.velocity.y += gravity;
                        
                        // 位置を更新
                        block.x += block.velocity.x;
                        block.y += block.velocity.y;
                        
                        // 画面端での跳ね返り
                        if (block.x < 0) {
                            block.x = 0;
                            block.velocity.x = -block.velocity.x * 0.5;
                        } else if (block.x + block.width > canvas.width) {
                            block.x = canvas.width - block.width;
                            block.velocity.x = -block.velocity.x * 0.5;
                        }
                        
                        // 台との衝突チェック
                        checkPlatformCollision(block);
                        
                        // 他のブロックとの衝突チェック
                        for (let j = 0; j < blocks.length; j++) {
                            if (i !== j && blocks[j].resting) {
                                checkBlockCollision(block, blocks[j]);
                            }
                        }
                        
                        // 画面下に落ちたらブロックを削除してミスカウントを増やす
                        if (block.y > canvas.height) {
                            blocks.splice(i, 1);
                            i--;
                            score = Math.max(0, score - 5); // スコア減少
                            scoreDisplay.textContent = `スコア: ${score}`;
                            
                            // ミスカウントを増やす
                            droppedBlocksCount++;
                            missDisplay.textContent = `ミス: ${droppedBlocksCount} / ${maxDroppedBlocks}`;
                            
                            // ミス表示を強調
                            missDisplay.style.color = 'red';
                            missDisplay.style.fontWeight = 'bold';
                            setTimeout(() => {
                                missDisplay.style.color = 'white';
                                missDisplay.style.fontWeight = 'normal';
                            }, 500);
                            
                            // 最大ミス回数に達したらゲームオーバー
                            if (droppedBlocksCount >= maxDroppedBlocks) {
                                gameOver = true;
                            }
                        }
                    }
                }
                
                // 台の角度を更新（ブロックの重さに基づく）
                updatePlatformAngle();
                
                // 不安定なブロックをチェック
                checkUnstableBlocks();
                
                // ゲームオーバーチェック
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
                            // プラットフォーム上のブロックの場合
                            if (xCenter > platformX && xCenter < platformX + platformWidth) {
                                supportFound = true;
                            }
                        } else if (blocks[i].restingOn !== null) {
                            // 他のブロック上のブロックの場合
                            const supportBlock = blocks[i].restingOn;
                            if (xCenter > supportBlock.x && 
                                xCenter < supportBlock.x + supportBlock.width) {
                                supportFound = true;
                            }
                        }
                        
                        // サポートが見つからない場合、ブロックを不安定にする
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
                // プラットフォームの傾斜を考慮した高さを計算
                const platformLeftY = platformY - Math.sin(platformAngle) * (platformWidth / 2);
                const platformRightY = platformY + Math.sin(platformAngle) * (platformWidth / 2);
                
                // ブロックの中心X座標
                const blockCenterX = block.x + block.width / 2;
                
                // ブロックがプラットフォームの幅範囲内かチェック
                if (blockCenterX >= platformX && blockCenterX <= platformX + platformWidth) {
                    // 傾斜に応じたY座標を計算
                    const ratio = (blockCenterX - platformX) / platformWidth;
                    const platformYAtBlock = platformLeftY * (1 - ratio) + platformRightY * ratio;
                    
                    const blockBottom = block.y + block.height;
                    
                    // 衝突チェック
                    if (blockBottom >= platformYAtBlock && 
                        blockBottom <= platformYAtBlock + platformHeight + block.velocity.y && 
                        block.velocity.y > 0) {
                        
                        // 衝突した場合、ブロックの位置とステータスを更新
                        block.y = platformYAtBlock - block.height;
                        block.velocity.y = 0;
                        block.velocity.x = 0;
                        block.resting = true;
                        block.onPlatform = true;
                        block.restingOn = null;
                        
                        // スコア加算
                        score += 10;
                        scoreDisplay.textContent = `スコア: ${score}`;
                        
                        return true;
                    }
                }
                
                return false;
            }
            
            // ブロック同士の衝突チェック
            function checkBlockCollision(movingBlock, staticBlock) {
                // 両方のブロックの中心座標
                const movingCenter = {
                    x: movingBlock.x + movingBlock.width / 2,
                    y: movingBlock.y + movingBlock.height / 2
                };
                
                const staticCenter = {
                    x: staticBlock.x + staticBlock.width / 2,
                    y: staticBlock.y + staticBlock.height / 2
                };
                
                // 衝突判定用の距離
                const dx = Math.abs(movingCenter.x - staticCenter.x);
                const dy = Math.abs(movingCenter.y - staticCenter.y);
                
                const halfWidthSum = (movingBlock.width + staticBlock.width) / 2;
                const halfHeightSum = (movingBlock.height + staticBlock.height) / 2;
                
                // 衝突チェック
                if (dx < halfWidthSum && dy < halfHeightSum) {
                    // 上からの衝突の場合（静止ブロックの上に乗る）
                    if (movingBlock.y + movingBlock.height <= staticBlock.y + 10 && 
                        movingBlock.velocity.y > 0) {
                        
                        movingBlock.y = staticBlock.y - movingBlock.height;
                        movingBlock.velocity.y = 0;
                        movingBlock.velocity.x = 0;
                        movingBlock.resting = true;
                        movingBlock.onPlatform = false;
                        movingBlock.restingOn = staticBlock;
                        
                        // スコア加算
                        score += 5;
                        scoreDisplay.textContent = `スコア: ${score}`;
                        
                        return true;
                    }
                    // 側面衝突の場合
                    else if (movingBlock.y + movingBlock.height > staticBlock.y + 10) {
                        // X方向の反発
                        movingBlock.velocity.x = -movingBlock.velocity.x * 0.5;
                        
                        // 左右の調整
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
                
                // 各ブロックの重さと位置から左右の重さを計算
                for (let i = 0; i < blocks.length; i++) {
                    if (blocks[i].resting) {
                        const blockCenter = blocks[i].x + blocks[i].width / 2;
                        const distanceFromCenter = blockCenter - platformCenter;
                        
                        let isOnPlatform = false;
                        
                        // 直接プラットフォーム上にあるか、間接的に影響するかをチェック
                        if (blocks[i].onPlatform) {
                            isOnPlatform = true;
                        } else {
                            // ブロックの下のブロックをたどって、最終的に台の上にあるか確認
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
                
                // 新しい角度を計算（重さの差に基づく）
                // 右の重さが大きいと右に傾く（正の角度）、左の重さが大きいと左に傾く（負の角度）
                const weightDifference = rightWeight - leftWeight;
                const targetAngle = (weightDifference * 0.0005) * (1 + level * 0.1); // レベルに応じて感度が上がる
                
                // 角度を滑らかに変化させる
                platformAngle += (targetAngle - platformAngle) * 0.1;
                
                // 角度を制限
                platformAngle = Math.max(-maxAngle, Math.min(maxAngle, platformAngle));
            }
            
            // レベルアップ
            function levelUp() {
                console.log(`Executing levelUp: current level=${level}, score=${score}`);
                
                // 次のレベルに必要なスコアを計算
                const requiredScore = level * 50;
                
                // 既にレベルアップ条件を満たしていない場合は処理しない
                if (score < requiredScore) {
                    console.log('Score too low for level up, returning');
                    return;
                }
                
                // レベルを上げる
                const newLevel = level + 1;
                console.log(`Increasing level from ${level} to ${newLevel}`);
                level = newLevel;
                levelDisplay.textContent = `レベル: ${level}`;
                
                // 難易度の調整
                platformSpeed = 10 + level;
                gravity = 0.3 + level * 0.05;
                
                try {
                    // レベルアップメッセージを表示
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
                    levelUpMsg.style.opacity = '1'; // 最初から表示
                    
                    const gameContainer = document.getElementById('gameContainer');
                    if (gameContainer) {
                        gameContainer.appendChild(levelUpMsg);
                        
                        // 安全なタイマー実装
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
                                // エラーが発生しても要素を削除
                                if (levelUpMsg.parentNode) {
                                    levelUpMsg.parentNode.removeChild(levelUpMsg);
                                }
                            }
                        }, 1000);
                    }
                    
                    // レベルアップボーナス
                    score += 20;
                    scoreDisplay.textContent = `スコア: ${score}`;
                    
                    console.log(`Level up complete. New level: ${level}, score: ${score}`);
                } catch (err) {
                    console.error('Error in levelUp function:', err);
                }
            }
            
            // 描画
            function draw() {
                // キャンバスをクリア
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                
                // 背景グラデーション
                const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
                gradient.addColorStop(0, '#6495ED');
                gradient.addColorStop(1, '#87CEEB');
                ctx.fillStyle = gradient;
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                
                // 画面端の視覚的なマーカー
                const edgeMarkerWidth = 5;
                ctx.fillStyle = 'rgba(255, 255, 255, 0.3)';
                ctx.fillRect(0, 0, edgeMarkerWidth, canvas.height);
                ctx.fillRect(canvas.width - edgeMarkerWidth, 0, edgeMarkerWidth, canvas.height);
                
                // プラットフォームの支柱
                ctx.fillStyle = '#8B4513';
                ctx.beginPath();
                ctx.moveTo(platformX + platformWidth / 2 - 5, platformY);
                ctx.lineTo(platformX + platformWidth / 2 + 5, platformY);
                ctx.lineTo(platformX + platformWidth / 2 + 10, canvas.height);
                ctx.lineTo(platformX + platformWidth / 2 - 10, canvas.height);
                ctx.closePath();
                ctx.fill();
                
                // プラットフォーム（台）の描画
                ctx.save();
                ctx.translate(platformX + platformWidth / 2, platformY);
                ctx.rotate(platformAngle);
                ctx.fillStyle = '#A0522D';
                ctx.fillRect(-platformWidth / 2, -platformHeight / 2, platformWidth, platformHeight);
                
                // プラットフォームの枠線
                ctx.strokeStyle = 'rgba(0, 0, 0, 0.3)';
                ctx.lineWidth = 2;
                ctx.strokeRect(-platformWidth / 2, -platformHeight / 2, platformWidth, platformHeight);
                ctx.restore();
                
                // ブロックの描画
                for (let i = 0; i < blocks.length; i++) {
                    const block = blocks[i];
                    
                    ctx.save();
                    ctx.translate(block.x + block.width / 2, block.y + block.height / 2);
                    
                    // ブロックの回転（傾斜に応じて）
                    if (block.onPlatform) {
                        ctx.rotate(platformAngle);
                    } else if (block.resting && block.restingOn !== null) {
                        // 下のブロックの角度に合わせる
                        ctx.rotate(block.restingOn.angle || 0);
                    } else {
                        // 自由落下中は少し回転させる
                        block.angle += 0.01 * block.velocity.y;
                        ctx.rotate(block.angle);
                    }
                    
                    // ブロックの色と枠線
                    ctx.fillStyle = block.color;
                    ctx.fillRect(-block.width / 2, -block.height / 2, block.width, block.height);
                    
                    ctx.strokeStyle = 'rgba(0, 0, 0, 0.3)';
                    ctx.lineWidth = 2;
                    ctx.strokeRect(-block.width / 2, -block.height / 2, block.width, block.height);
                    
                    ctx.restore();
                }
                
                // 傾き角度のインジケーター
                const angleIndicatorWidth = 100;
                const angleIndicatorHeight = 20;
                const angleIndicatorX = canvas.width / 2 - angleIndicatorWidth / 2;
                const angleIndicatorY = 50;
                
                // 背景
                ctx.fillStyle = 'rgba(0, 0, 0, 0.3)';
                ctx.fillRect(angleIndicatorX, angleIndicatorY, angleIndicatorWidth, angleIndicatorHeight);
                
                // 現在の傾き
                const tiltPercentage = (platformAngle / maxAngle) * 0.5 + 0.5; // 0-1の範囲に変換
                const tiltWidth = tiltPercentage * angleIndicatorWidth;
                
                // 色を傾きに応じて変更（緑から赤へ）
                let tiltColor;
                if (tiltPercentage > 0.7 || tiltPercentage < 0.3) {
                    tiltColor = '#FF5733'; // 危険な傾き
                } else if (tiltPercentage > 0.6 || tiltPercentage < 0.4) {
                    tiltColor = '#FFC300'; // 警告傾き
                } else {
                    tiltColor = '#33FF57'; // 安全な傾き
                }
                
                ctx.fillStyle = tiltColor;
                ctx.fillRect(angleIndicatorX, angleIndicatorY, tiltWidth, angleIndicatorHeight);
                
                // 中央マーカー
                ctx.fillStyle = 'white';
                ctx.fillRect(angleIndicatorX + angleIndicatorWidth / 2 - 1, angleIndicatorY, 2, angleIndicatorHeight);
                
                // 次のレベルまでの進捗バー
                const progressBarWidth = 200;
                const progressBarHeight = 10;
                const progressBarX = canvas.width / 2 - progressBarWidth / 2;
                const progressBarY = 80;
                
                // 進捗バーの背景
                ctx.fillStyle = 'rgba(0, 0, 0, 0.3)';
                ctx.fillRect(progressBarX, progressBarY, progressBarWidth, progressBarHeight);
                
                // 進捗の計算（安全に行う）
                const nextLevelScore = level * 50;
                const previousLevelScore = (level - 1) * 50;
                
                // スコアと前のレベルスコアの差分（負にならないよう保証）
                const scoreDifference = Math.max(0, score - previousLevelScore);
                // レベル間のスコア差分
                const levelScoreDifference = nextLevelScore - previousLevelScore;
                // 進捗率（0〜1の範囲に制限）
                let progressPercentage = 0;
                if (levelScoreDifference > 0) { // ゼロ除算を防止
                    progressPercentage = Math.min(1, scoreDifference / levelScoreDifference);
                }
                
                // 進捗バーの幅を計算
                const progressWidth = progressPercentage * progressBarWidth;
                
                // 進捗バーの描画
                ctx.fillStyle = '#4CAF50';
                ctx.fillRect(progressBarX, progressBarY, progressWidth, progressBarHeight);
                
                // 「次のレベルまで」のテキスト
                ctx.fillStyle = 'white';
                ctx.font = '12px Arial';
                ctx.textAlign = 'center';
                
                // 次のレベルまでの残りポイントを計算（0未満にならないように）
                const pointsToNextLevel = Math.max(0, nextLevelScore - score);
                
                if (pointsToNextLevel > 0) {
                    ctx.fillText(`次のLv.${level+1}まであと${pointsToNextLevel}点`, canvas.width / 2, progressBarY + 25);
                } else {
                    ctx.fillText(`Lv.${level+1}に上がる準備完了！`, canvas.width / 2, progressBarY + 25);
                }
            }
            
            // ゲームオーバー表示
            function showGameOver() {
                gameRunning = false;
                
                // メッセージオーバーレイを表示
                messageOverlay.style.display = 'flex';
                
                // メッセージ内容をクリア
                while (messageOverlay.children.length > 3) {
                    messageOverlay.removeChild(messageOverlay.lastChild);
                }
                
                // メッセージタイトル設定
                messageText.textContent = 'ゲームオーバー';
                
                // ゲームオーバーの理由を表示
                const reasonText = droppedBlocksCount >= maxDroppedBlocks
                    ? `ブロックを${maxDroppedBlocks}回落としてしまいました`
                    : 'バランスが崩れてしまいました';
                
                const reasonElement = document.createElement('div');
                reasonElement.textContent = reasonText;
                reasonElement.style.fontSize = '20px';
                reasonElement.style.marginTop = '10px';
                reasonElement.style.marginBottom = '10px';
                messageOverlay.appendChild(reasonElement);
                
                // リスタートボタン
                const restartButton = document.createElement('button');
                restartButton.id = 'restartButton';
                restartButton.textContent = 'もう一度プレイ';
                restartButton.addEventListener('click', startGame);
                messageOverlay.appendChild(restartButton);
                
                // 最終スコア表示
                const finalScore = document.createElement('div');
                finalScore.textContent = `最終スコア: ${score}`;
                finalScore.style.fontSize = '24px';
                finalScore.style.marginTop = '20px';
                messageOverlay.appendChild(finalScore);
                
                // レベル表示
                const finalLevel = document.createElement('div');
                finalLevel.textContent = `到達レベル: ${level}`;
                finalLevel.style.fontSize = '20px';
                finalLevel.style.marginTop = '10px';
                messageOverlay.appendChild(finalLevel);
                
                // スタートボタンを非表示
                if (startButton) {
                    startButton.style.display = 'none';
                }
            }
            
            // イベントリスナーの設定
            
            // スタートボタン
            startButton.addEventListener('click', startGame);
            
            // タッチ操作（左）
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
            
            // タッチ操作（右）
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
            
            // マウスアップでの制御解除
            document.addEventListener('mouseup', function() {
                leftPressed = false;
                rightPressed = false;
            });
            
            // キーボード操作
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
            
            // 初期描画
            draw();
        });
    </script>
</body>
</html>
