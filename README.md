<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <!-- ビューポート設定で各デバイスにフィット -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>インベーダー風ゲーム</title>
  <style>
    body {
      margin: 0;
      overflow: hidden;
      background: black;
      color: white;
      font-family: sans-serif;
    }
    canvas {
      display: block;
      margin: 0 auto;
      background: #000;
    }
    /* タッチ操作用コントロール */
    #controls {
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      display: flex;
      gap: 20px;
      z-index: 10;
    }
    .button {
      width: 60px;
      height: 60px;
      background: rgba(255, 255, 255, 0.5);
      border-radius: 50%;
      text-align: center;
      line-height: 60px;
      font-size: 30px;
      user-select: none;
    }
    /* リスタート用ボタン（ゲームオーバー時に表示） */
    #restartBtn {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      padding: 20px 40px;
      font-size: 24px;
      background: rgba(255,255,255,0.8);
      border: none;
      border-radius: 10px;
      display: none;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <!-- ゲーム描画用キャンバス -->
  <canvas id="gameCanvas" width="480" height="640"></canvas>
  
  <!-- タッチ操作用のボタン -->
  <div id="controls">
    <div id="leftBtn" class="button">&#8592;</div>
    <div id="fireBtn" class="button">&#9679;</div>
    <div id="rightBtn" class="button">&#8594;</div>
  </div>

  <!-- ゲームオーバー時に表示されるリスタートボタン -->
  <button id="restartBtn">Restart</button>

  <!-- 効果音用オーディオ（data URIを利用。必要に応じて置き換えてください） -->
  <audio id="shootSound" src="data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEAESsAACJWAAACABAAZGF0YQgAAAAA"></audio>
  <audio id="explosionSound" src="data:audio/wav;base64,UklGRkQAAABXQVZFZm10IBAAAAABAAEAESsAACJWAAACABAAZGF0YUQAAAAA"></audio>
  <audio id="enemyShootSound" src="data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEAESsAACJWAAACABAAZGF0YQgAAAAA"></audio>
  
  <script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");

    // プレイヤー（自機）の設定
    const ship = {
      x: canvas.width / 2 - 15,
      y: canvas.height - 40,
      width: 30,
      height: 20,
      speed: 5
    };

    // 自機弾、敵弾、爆発エフェクト用の配列
    const bullets = [];
    const enemyBullets = [];
    const explosions = [];

    // 敵（インベーダー）の設定
    const invaderRows = 4;
    const invaderCols = 8;
    const invaderWidth = 30;
    const invaderHeight = 20;
    const invaderPadding = 10;
    const invaderOffsetLeft = 30;
    const invaderOffsetTop = 30;

    // ゲーム管理用
    let score = 0;
    let gameOver = false;
    let stage = 1;
    let stageCleared = false;

    // 敵全体の動きを管理するための形成オフセットと速度（難易度で変化）
    let formationOffsetX = invaderOffsetLeft;
    let formationOffsetY = invaderOffsetTop;
    let formationDirection = 1; // 1:右方向, -1:左方向
    let invaderSpeed = 1;
    const dropDistance = 20;
    const boundaryMargin = 10;

    // 敵の各個体の状態（status: 1=生存、0=破壊済み）
    const invaders = [];
    for (let r = 0; r < invaderRows; r++) {
      invaders[r] = [];
      for (let c = 0; c < invaderCols; c++) {
        invaders[r][c] = { status: 1 };
      }
    }

    // 敵弾の速度
    const enemyBulletSpeed = 4;

    // 左右操作のフラグ
    let leftPressed = false;
    let rightPressed = false;

    // プレイヤーの描画
    function drawShip() {
      ctx.fillStyle = "#00FF00";
      ctx.fillRect(ship.x, ship.y, ship.width, ship.height);
    }

    // 自機弾の描画
    function drawBullets() {
      ctx.fillStyle = "#FFFF00";
      bullets.forEach(bullet => {
        ctx.fillRect(bullet.x, bullet.y, 4, 10);
      });
    }

    // 敵弾の描画
    function drawEnemyBullets() {
      ctx.fillStyle = "#00FFFF";
      enemyBullets.forEach(bullet => {
        ctx.fillRect(bullet.x, bullet.y, 4, 10);
      });
    }

    // 爆発エフェクトの描画
    function drawExplosions() {
      explosions.forEach(exp => {
        ctx.save();
        ctx.globalAlpha = exp.alpha;
        ctx.beginPath();
        ctx.arc(exp.x, exp.y, exp.radius, 0, Math.PI * 2);
        ctx.fillStyle = "#FFA500";
        ctx.fill();
        ctx.restore();
      });
    }

    // 敵の描画（形成オフセットを利用）
    function drawInvaders() {
      for (let r = 0; r < invaderRows; r++) {
        for (let c = 0; c < invaderCols; c++) {
          if (invaders[r][c].status === 1) {
            const invX = formationOffsetX + c * (invaderWidth + invaderPadding);
            const invY = formationOffsetY + r * (invaderHeight + invaderPadding);
            ctx.fillStyle = "#FF0000";
            ctx.fillRect(invX, invY, invaderWidth, invaderHeight);
          }
        }
      }
    }

    // スコアおよびステージの描画
    function drawScore() {
      ctx.fillStyle = "white";
      ctx.font = "20px sans-serif";
      ctx.textAlign = "left";
      ctx.fillText("Score: " + score + "   Stage: " + stage, 10, 25);
    }

    // 自機弾の更新
    function updateBullets() {
      for (let i = 0; i < bullets.length; i++) {
        bullets[i].y -= 7;
        if (bullets[i].y < 0) {
          bullets.splice(i, 1);
          i--;
        }
      }
    }

    // 敵弾の更新と自機との衝突判定
    function updateEnemyBullets() {
      for (let i = 0; i < enemyBullets.length; i++) {
        enemyBullets[i].y += enemyBulletSpeed;
        if (enemyBullets[i].y > canvas.height) {
          enemyBullets.splice(i, 1);
          i--;
        } else {
          // 自機との衝突判定
          if (
            enemyBullets[i].x > ship.x &&
            enemyBullets[i].x < ship.x + ship.width &&
            enemyBullets[i].y > ship.y &&
            enemyBullets[i].y < ship.y + ship.height
          ) {
            gameOver = true;
            enemyBullets.splice(i, 1);
            i--;
          }
        }
      }
    }

    // 爆発エフェクトの更新
    function updateExplosions() {
      for (let i = 0; i < explosions.length; i++) {
        explosions[i].radius += 1;
        explosions[i].alpha -= 0.05;
        if (explosions[i].alpha <= 0) {
          explosions.splice(i, 1);
          i--;
        }
      }
    }

    // 自機弾と敵の衝突判定（命中時はスコア加算と爆発エフェクト＆効果音）
    function collisionDetection() {
      bullets.forEach((bullet, bIndex) => {
        for (let r = 0; r < invaderRows; r++) {
          for (let c = 0; c < invaderCols; c++) {
            if (invaders[r][c].status === 1) {
              const invX = formationOffsetX + c * (invaderWidth + invaderPadding);
              const invY = formationOffsetY + r * (invaderHeight + invaderPadding);
              if (
                bullet.x > invX &&
                bullet.x < invX + invaderWidth &&
                bullet.y > invY &&
                bullet.y < invY + invaderHeight
              ) {
                invaders[r][c].status = 0;
                bullets.splice(bIndex, 1);
                score += 10;
                explosions.push({
                  x: invX + invaderWidth / 2,
                  y: invY + invaderHeight / 2,
                  radius: 0,
                  alpha: 1.0
                });
                // 効果音（爆発）
                const expSound = document.getElementById("explosionSound");
                expSound.currentTime = 0;
                expSound.play();
                return;
              }
            }
          }
        }
      });
    }

    // 敵全体の移動処理（画面端で左右反転し、下に降りる）
    function updateInvaders() {
      const formationRight = formationOffsetX + (invaderCols - 1) * (invaderWidth + invaderPadding) + invaderWidth;
      const formationLeft = formationOffsetX;
      if ((formationRight > canvas.width - boundaryMargin && formationDirection === 1) ||
          (formationLeft < boundaryMargin && formationDirection === -1)) {
        formationDirection *= -1;
        formationOffsetY += dropDistance;
      } else {
        formationOffsetX += invaderSpeed * formationDirection;
      }
    }

    // 敵が自機に向けて弾を撃つ処理（下段の敵からランダムに選択）
    function enemyShooting() {
      // ステージが上がると発射確率も上昇
      if (Math.random() < 0.005 * stage) {
        let aliveEnemies = [];
        for (let c = 0; c < invaderCols; c++) {
          for (let r = invaderRows - 1; r >= 0; r--) {
            if (invaders[r][c].status === 1) {
              const invX = formationOffsetX + c * (invaderWidth + invaderPadding);
              const invY = formationOffsetY + r * (invaderHeight + invaderPadding);
              aliveEnemies.push({ x: invX, y: invY });
              break;
            }
          }
        }
        if (aliveEnemies.length > 0) {
          const shooter = aliveEnemies[Math.floor(Math.random() * aliveEnemies.length)];
          enemyBullets.push({ x: shooter.x + invaderWidth / 2 - 2, y: shooter.y + invaderHeight });
          const enemyShot = document.getElementById("enemyShootSound");
          enemyShot.currentTime = 0;
          enemyShot.play();
        }
      }
    }

    // ゲームオーバー判定（敵が自機に近づいた場合）
    function checkGameOver() {
      for (let r = 0; r < invaderRows; r++) {
        for (let c = 0; c < invaderCols; c++) {
          if (invaders[r][c].status === 1) {
            const invY = formationOffsetY + r * (invaderHeight + invaderPadding);
            if (invY + invaderHeight >= ship.y) {
              gameOver = true;
              return;
            }
          }
        }
      }
    }

    // ステージクリア判定：全ての敵が倒されたら次ステージへ
    function checkStageClear() {
      let remaining = false;
      for (let r = 0; r < invaderRows; r++) {
        for (let c = 0; c < invaderCols; c++) {
          if (invaders[r][c].status === 1) {
            remaining = true;
            break;
          }
        }
        if (remaining) break;
      }
      if (!remaining) {
        stageCleared = true;
        setTimeout(nextStage, 2000);
      }
    }

    // 描画処理（すべての要素を再描画）
    function draw() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      drawShip();
      drawBullets();
      drawEnemyBullets();
      drawInvaders();
      drawExplosions();
      drawScore();
    }

    // メインループ
    function update() {
      if (!gameOver && !stageCleared) {
        moveShip();
        updateBullets();
        updateEnemyBullets();
        updateExplosions();
        collisionDetection();
        updateInvaders();
        enemyShooting();
        checkGameOver();
        checkStageClear();
      }
      draw();
      if (gameOver) {
        ctx.fillStyle = "rgba(0, 0, 0, 0.7)";
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = "white";
        ctx.font = "40px sans-serif";
        ctx.textAlign = "center";
        ctx.fillText("Game Over", canvas.width / 2, canvas.height / 2 - 20);
        ctx.font = "20px sans-serif";
        ctx.fillText("Score: " + score, canvas.width / 2, canvas.height / 2 + 20);
        document.getElementById("restartBtn").style.display = "block";
      }
      if (stageCleared && !gameOver) {
        ctx.fillStyle = "rgba(0, 0, 0, 0.7)";
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = "white";
        ctx.font = "40px sans-serif";
        ctx.textAlign = "center";
        ctx.fillText("Stage Clear", canvas.width / 2, canvas.height / 2 - 20);
      }
      requestAnimationFrame(update);
    }
    update();

    // 自機の移動処理
    function moveShip() {
      if (leftPressed && ship.x > 0) {
        ship.x -= ship.speed;
      }
      if (rightPressed && ship.x < canvas.width - ship.width) {
        ship.x += ship.speed;
      }
    }

    // タッチ操作用ボタンのイベント設定
    const leftBtn = document.getElementById("leftBtn");
    const rightBtn = document.getElementById("rightBtn");
    const fireBtn = document.getElementById("fireBtn");

    leftBtn.addEventListener("touchstart", (e) => {
      e.preventDefault();
      leftPressed = true;
    });
    leftBtn.addEventListener("touchend", (e) => {
      e.preventDefault();
      leftPressed = false;
    });
    rightBtn.addEventListener("touchstart", (e) => {
      e.preventDefault();
      rightPressed = true;
    });
    rightBtn.addEventListener("touchend", (e) => {
      e.preventDefault();
      rightPressed = false;
    });
    fireBtn.addEventListener("touchstart", (e) => {
      e.preventDefault();
      bullets.push({ x: ship.x + ship.width / 2 - 2, y: ship.y });
      const shootSound = document.getElementById("shootSound");
      shootSound.currentTime = 0;
      shootSound.play();
    });

    // デスクトップ用マウス操作
    leftBtn.addEventListener("mousedown", () => { leftPressed = true; });
    leftBtn.addEventListener("mouseup", () => { leftPressed = false; });
    rightBtn.addEventListener("mousedown", () => { rightPressed = true; });
    rightBtn.addEventListener("mouseup", () => { rightPressed = false; });
    fireBtn.addEventListener("mousedown", () => {
      bullets.push({ x: ship.x + ship.width / 2 - 2, y: ship.y });
      const shootSound = document.getElementById("shootSound");
      shootSound.currentTime = 0;
      shootSound.play();
    });

    // リスタートボタンのクリックイベント
    document.getElementById("restartBtn").addEventListener("click", restartGame);

    // 次のステージへ進む処理（難易度アップ）
    function nextStage() {
      stage++;
      invaderSpeed += 0.5; // 敵の移動速度アップ
      formationOffsetX = invaderOffsetLeft;
      formationOffsetY = invaderOffsetTop;
      formationDirection = 1;
      // 敵状態のリセット
      for (let r = 0; r < invaderRows; r++) {
        for (let c = 0; c < invaderCols; c++) {
          invaders[r][c].status = 1;
        }
      }
      // 自機弾、敵弾をクリア
      bullets.length = 0;
      enemyBullets.length = 0;
      stageCleared = false;
    }

    // ゲームリスタート処理：各変数を初期状態に戻す
    function restartGame() {
      ship.x = canvas.width / 2 - 15;
      ship.y = canvas.height - 40;
      bullets.length = 0;
      enemyBullets.length = 0;
      explosions.length = 0;
      score = 0;
      stage = 1;
      gameOver = false;
      stageCleared = false;
      formationOffsetX = invaderOffsetLeft;
      formationOffsetY = invaderOffsetTop;
      formationDirection = 1;
      invaderSpeed = 1;
      for (let r = 0; r < invaderRows; r++) {
        for (let c = 0; c < invaderCols; c++) {
          invaders[r][c].status = 1;
        }
      }
      document.getElementById("restartBtn").style.display = "none";
    }
  </script>
</body>
</html>
