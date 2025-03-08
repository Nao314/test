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

    // 弾および爆発エフェクト用の配列
    const bullets = [];
    const explosions = [];

    // インベーダーの設定
    const invaderRows = 4;
    const invaderCols = 8;
    const invaderWidth = 30;
    const invaderHeight = 20;
    const invaderPadding = 10;
    const invaderOffsetLeft = 30;
    const invaderOffsetTop = 30;
    let score = 0;
    let gameOver = false;

    // 敵全体の動きを管理するための形成オフセット
    let formationOffsetX = invaderOffsetLeft;
    let formationOffsetY = invaderOffsetTop;
    let formationDirection = 1; // 1:右方向, -1:左方向
    const invaderSpeed = 1;
    const dropDistance = 20;
    const boundaryMargin = 10;

    // インベーダーの各個体の状態（status: 1=生存、0=破壊済み）
    const invaders = [];
    for (let r = 0; r < invaderRows; r++) {
      invaders[r] = [];
      for (let c = 0; c < invaderCols; c++) {
        invaders[r][c] = { status: 1 };
      }
    }

    // 左右操作のフラグ
    let leftPressed = false;
    let rightPressed = false;

    // プレイヤーの描画
    function drawShip() {
      ctx.fillStyle = "#00FF00";
      ctx.fillRect(ship.x, ship.y, ship.width, ship.height);
    }

    // 弾の描画
    function drawBullets() {
      ctx.fillStyle = "#FFFF00";
      bullets.forEach(bullet => {
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

    // インベーダーの描画（形成オフセットを利用）
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

    // スコアの描画
    function drawScore() {
      ctx.fillStyle = "white";
      ctx.font = "20px sans-serif";
      ctx.fillText("Score: " + score, 10, 25);
    }

    // 弾の更新
    function updateBullets() {
      for (let i = 0; i < bullets.length; i++) {
        bullets[i].y -= 7;
        if (bullets[i].y < 0) {
          bullets.splice(i, 1);
          i--;
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

    // 弾とインベーダーの衝突判定と、衝突時の処理（スコア加算・爆発エフェクト追加）
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
                // 爆発エフェクトを生成（敵の中心位置で開始）
                explosions.push({
                  x: invX + invaderWidth / 2,
                  y: invY + invaderHeight / 2,
                  radius: 0,
                  alpha: 1.0
                });
                return;
              }
            }
          }
        }
      });
    }

    // インベーダーの移動処理（画面端で左右反転し、下に降りる）
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

    // ゲームオーバー判定（敵がプレイヤーに近づいた場合）
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

    // 描画処理（全要素を再描画）
    function draw() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      drawShip();
      drawBullets();
      drawInvaders();
      drawExplosions();
      drawScore();
    }

    // メインループ：更新→描画
    function update() {
      if (!gameOver) {
        moveShip();
        updateBullets();
        updateExplosions();
        collisionDetection();
        updateInvaders();
        checkGameOver();
      }
      draw();
      if (gameOver) {
        // ゲームオーバー時のオーバーレイ表示
        ctx.fillStyle = "rgba(0, 0, 0, 0.7)";
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = "white";
        ctx.font = "40px sans-serif";
        ctx.textAlign = "center";
        ctx.fillText("Game Over", canvas.width / 2, canvas.height / 2 - 20);
        ctx.font = "20px sans-serif";
        ctx.fillText("Score: " + score, canvas.width / 2, canvas.height / 2 + 20);
        // リスタートボタンを表示
        document.getElementById("restartBtn").style.display = "block";
      }
      requestAnimationFrame(update);
    }
    update();

    // プレイヤーの移動処理
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
    });

    // デスクトップ用マウス操作
    leftBtn.addEventListener("mousedown", () => { leftPressed = true; });
    leftBtn.addEventListener("mouseup", () => { leftPressed = false; });
    rightBtn.addEventListener("mousedown", () => { rightPressed = true; });
    rightBtn.addEventListener("mouseup", () => { rightPressed = false; });
    fireBtn.addEventListener("mousedown", () => {
      bullets.push({ x: ship.x + ship.width / 2 - 2, y: ship.y });
    });

    // リスタートボタンのクリックイベント
    document.getElementById("restartBtn").addEventListener("click", restartGame);

    // ゲームリスタート処理：各変数を初期状態に戻す
    function restartGame() {
      ship.x = canvas.width / 2 - 15;
      ship.y = canvas.height - 40;
      bullets.length = 0;
      explosions.length = 0;
      score = 0;
      gameOver = false;
      formationOffsetX = invaderOffsetLeft;
      formationOffsetY = invaderOffsetTop;
      formationDirection = 1;
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
