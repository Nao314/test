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
            touch-action: none; /* タッチ操作による画面のスクロールを防止 */
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
        #scoreDisplay {
            position: absolute;
            top: 10px;
            left: 10px;
            font-size: 20px;
            color: white;
            text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.8);
            z-index: 10;
        }
        #levelDisplay {
            position: absolute;
            top: 10px;
            right: 10px;
            font-size: 20px;
            color: white;
            text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.8);
            z-index: 10;
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
        .controlButton img {
            width: 40px;
            height: 40px;
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
        <canvas id="gameCanvas"></canvas>
        
        <div id="controlsOverlay">
            <div id="leftControl" class="controlButton">
                <div style="font-size: 30px; color: white;">←</div>
            </div>
            <div id="rightControl" class="controlButton">
                <div style="font-size: 30px; color: white;">→</div>
            </div>
        </div>
        
        <div id="messageOverlay">
            <div id="messageText">バランスキーパー</div>
            <div style="font-size: 18px; text-align: center; padding: 0 20px;">
                台を左右に動かして、落ちてくるブロックをキャッチしよう！<br>
                バランスを崩さないように注意！
            </div>
            <button id="startButton">ゲームスタート</button>
        </div>
    </div>

    <script>
        // キャンバスの設定
        const gameContainer = document.getElementById('gameContainer');
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreDisplay = document.getElementById('scoreDisplay');
        const levelDisplay = document.getElementById('levelDisplay');
        const messageOverlay = document.getElementById('messageOverlay');
        const messageText = document.getElementById('messageText');
        const startButton = document.getElementById('startButton');
        const leftControl = document.getElementById('leftControl');
        const rightControl = document.getElementById('rightControl');
        
        // キャンバスのサイズをコンテナに合わせる
        function resizeCanvas() {
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
        
        // ゲーム変数
        let score = 0;
        let level = 1;
        let gameOver = false;
        let gameRunning = false;
        let animationFrameId = null;
        
        // 台（プラットフォーム）変数
        let platformWidth = Math.min(canvas.width * 0.3, 180);
        let platformHeight = canvas.height * 0.02;
        let platformY = canvas.height * 0.85;
        let platformX = canvas.width / 2 - platformWidth / 2;
        let platformSpeed = 10;
        let platformAngle = 0;
        const maxAngle = Math.PI / 6; // 最大傾き角度
        
        // 制御変数
        let leftPressed = false;
        let rightPressed = false;
        
        // ブロック変数
        let blocks = [];
        const blockColors = ['#FF5733', '#33FF57', '#3357FF', '#FF33A8', '#33A8FF', '#A833FF', '#FFFF33', '#33FFFF'];
        const gravity = 0.3; // 重力
        let spawnInterval; // ブロック生成間隔
        let lastSpawnTime = 0;
        let spawnDelay = 2000; // ミリ秒
        
        // ゲーム開始
        function startGame() {
            score = 0;
            level = 1;
            gameOver = false;
            gameRunning = true;
            blocks = [];
            platformAngle = 0;
            platformX = canvas.width / 2 - platformWidth / 2;
            
            // スコアとレベルの表示を更新
            scoreDisplay.textContent = `スコア: ${score}`;
            levelDisplay.textContent = `レベル: ${level}`;
            
            // メッセージオーバーレイを非表示
            messageOverlay.style.display = 'none';
            
            // ゲームループを開始
            lastSpawnTime = Date.now();
            if (animationFrameId !== null) {
                cancelAnimationFrame(animationFrameId);
            }
            gameLoop();
        }
        
        // ゲームループ
        function gameLoop() {
            if (!gameRunning) return;
            
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
            
            // ゲームオーバーでない場合は次のフレームを要求
            if (!gameOver) {
                animationFrameId = requestAnimationFrame(gameLoop);
            } else {
                showGameOver();
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
            const moveAmount = direction * platformSpeed;
            platformX += moveAmount;
            
            // 画面外に出ないように制限
            platformX = Math.max(0, Math.min(platformX, canvas.width - platformWidth));
            
            // 台の上にあるブロックも一緒に移動
            for (let i = 0; i < blocks.length; i++) {
                if (blocks[i].onPlatform) {
                    blocks[i].x += moveAmount;
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
                        blocks[i].x += moveAmount;
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
                    
                    // 画面下に落ちたらゲームオーバー
                    if (block.y > canvas.height) {
                        // ブロックを削除
                        blocks.splice(i, 1);
                        i--;
                        score = Math.max(0, score - 5); // スコア減少
                        scoreDisplay.textContent = `スコア: ${score}`;
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
                    
                    // レベルアップチェック
                    if (score >= level * 100) {
                        levelUp();
                    }
                    
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
            const weightDifference = leftWeight - rightWeight;
            const targetAngle = (weightDifference * 0.0005) * (1 + level * 0.1); // レベルに応じて感度が上がる
            
            // 角度を滑らかに変化させる
            platformAngle += (targetAngle - platformAngle) * 0.1;
            
            // 角度を制限
            platformAngle = Math.max(-maxAngle, Math.min(maxAngle, platformAngle));
        }
        
        // レベルアップ
        function levelUp() {
            level++;
            levelDisplay.textContent = `レベル: ${level}`;
            
            // レベルアップメッセージを一時的に表示
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
            gameContainer.appendChild(levelUpMsg);
            
            // メッセージを数秒後に削除
            setTimeout(() => {
                gameContainer.removeChild(levelUpMsg);
            }, 1500);
            
            // 難易度の調整
            platformSpeed = 10 + level;
            gravity = 0.3 + level * 0.05;
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
        }
        
        // ゲームオーバー表示
        function showGameOver() {
            gameRunning = false;
            
            // メッセージオーバーレイを表示
            messageOverlay.style.display = 'flex';
            messageText.textContent = 'ゲームオーバー';
            
            // リスタートボタン
            if (!document.getElementById('restartButton')) {
                const restartButton = document.createElement('button');
                restartButton.id = 'restartButton';
                restartButton.textContent = 'もう一度プレイ';
                restartButton.addEventListener('click', startGame);
                messageOverlay.appendChild(restartButton);
            } else {
                document.getElementById('restartButton').style.display = 'block';
            }
            
            // 最終スコア表示
            const finalScore = document.createElement('div');
            finalScore.textContent = `最終スコア: ${score}`;
            finalScore.style.fontSize = '24px';
            finalScore.style.marginTop = '20px';
            messageOverlay.appendChild(finalScore);
            
            // スタートボタンを非表示
            if (startButton) {
                startButton.style.display = 'none';
            }
        }
        
        // タッチ/マウス操作のイベントリスナー
        
        // 左側のコントロール
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
        
        // 右側のコントロール
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
        
        // マウスアップでの制御解除（両方）
        document.addEventListener('mouseup', function() {
            leftPressed = false;
            rightPressed = false;
        });
        
        // キーボード操作も可能に
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
        
        // ページ読み込み完了時の処理
        document.addEventListener('DOMContentLoaded', function() {
            // 初期描画
            draw();
        });
        
        // スタートボタンのイベントリスナー
        startButton.addEventListener('click', function(e) {
            e.preventDefault();
            startGame();
        });
        
        // 初期描画
        draw();
        
        // スタートボタンを確実に動作させるため、追加のイベントリスナー
        document.getElementById('startButton').onclick = function() {
            startGame();
            return false;
        };
    </script>
</body>
</html>
