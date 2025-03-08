<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <!-- ビューポート設定でiPadなどの画面にフィット -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>インベーダー風ゲーム</title>
  <style>
    body {
      margin: 0;
      overflow: hidden;
      background: black;
    }
    canvas {
      display: block;
      margin: 0 auto;
      background: #000;
    }
    /* 画面下部のタッチコントロール */
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
  </style>
</head>
<body>
  <!-- ゲーム描画用のキャンバス -->
  <canvas id="gameCanvas" width="480" height="640"></canvas>
  
  <!-- タッチ操作用のコントロール -->
  <div id="controls">
    <div id="leftBtn" class="button">&#8592;</div>
    <div id="fireBtn" class="button">&#9679;</div>
    <div id="rightBtn" class="button">&#8594;</div>
  </div>
  
  <script>
    // キャンバスとコンテキストの設定
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");

    // 自機（プレイヤー）の設定
    const ship = {
      x: canvas.width / 2 - 15,
      y: canvas.height - 40,
      width: 30,
      height: 20,
      speed: 5
    };

    // 弾とインベーダー用の配列
    const bullets = [];
    const invaders = [];
    const invaderRows = 4;
    const invaderCols = 8;
    const invaderWidth = 30;
    const invaderHeight = 20;
    const invaderPadding = 10;
    const invaderOffsetTop = 30;
    const invaderOffsetLeft = 30;
    let invaderDirection = 1; // 1:右移動、-1:左移動
    const invaderSpeed = 1;

    // インベーダーの初期配置
    for (let r = 0; r < invaderRows; r++) {
      invaders[r] = [];
      for (let c = 0; c < invaderCols; c++) {
        invaders[r][c] = { x: 0, y: 0, status: 1 };
      }
    }

    // 自機の描画
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

    // インベーダーの描画
    function drawInvaders() {
      for (let r = 0; r < invaderRows; r++) {
        for (let c = 0; c < invaderCols; c++) {
          let inv = invaders[r][c];
          if (inv.status === 1) {
            // インベーダーの位置を計算
            const invX = c * (invaderWidth + invaderPadding) + invaderOffsetLeft;
            const invY = r * (invaderHeight + invaderPadding) + invaderOffsetTop;
            inv.x = invX;
            inv.y = invY;
            ctx.fillStyle = "#FF0000";
            ctx.fillRect(invX, invY, invaderWidth, invaderHeight);
          }
        }
      }
    }

    // 弾の更新（上方向へ移動）
    function updateBullets() {
      for (let i = 0; i < bullets.length; i++) {
        bullets[i].y -= 7;
        if (bullets[i].y < 0) {
          bullets.splice(i, 1);
          i--;
        }
      }
    }

    // 弾とインベーダーの衝突判定
    function collisionDetection() {
      bullets.forEach((bullet, bIndex) => {
        for (let r = 0; r < invaderRows; r++) {
          for (let c = 0; c < invaderCols; c++) {
            let inv = invaders[r][c];
            if (inv.status === 1) {
              if (
                bullet.x > inv.x &&
                bullet.x < inv.x + invaderWidth &&
                bullet.y > inv.y &&
                bullet.y < inv.y + invaderHeight
              ) {
                inv.status = 0; // 当たったインベーダーを消去
                bullets.splice(bIndex, 1);
                return;
              }
            }
          }
        }
      });
    }

    // インベーダーの移動処理
    function updateInvaders() {
      let rightMost = 0;
      let leftMost = canvas.width;
      for (let r = 0; r < invaderRows; r++) {
        for (let c = 0; c < invaderCols; c++) {
          const inv = invaders[r][c];
          if (inv.status === 1) {
            rightMost = Math.max(rightMost, inv.x + invaderWidth);
            leftMost = Math.min(leftMost, inv.x);
          }
        }
      }
      // 端に到達したら方向転換＆下に移動
      if (rightMost > canvas.width - 10 && invaderDirection === 1) {
        invaderDirection = -1;
        for (let r = 0; r < invaderRows; r++) {
          for (let c = 0; c < invaderCols; c++) {
            invaders[r][c].y += 20;
          }
        }
      } else if (leftMost < 10 && invaderDirection === -1) {
        invaderDirection = 1;
        for (let r = 0; r < invaderRows; r++) {
          for (let c = 0; c < invaderCols; c++) {
            invaders[r][c].y += 20;
          }
        }
      }
      // 横移動
      for (let r = 0; r < invaderRows; r++) {
        for (let c = 0; c < invaderCols; c++) {
          if (invaders[r][c].status === 1) {
            invaders[r][c].x += invaderSpeed * invaderDirection;
          }
        }
      }
    }

    // ゲームループ
    function draw() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      drawShip();
      drawBullets();
      drawInvaders();
      updateBullets();
      collisionDetection();
      updateInvaders();
      requestAnimationFrame(draw);
    }
    draw();

    // タッチ・マウス操作用のフラグ
    let leftPressed = false;
    let rightPressed = false;

    // 自機の移動処理
    function moveShip() {
      if (leftPressed && ship.x > 0) {
        ship.x -= ship.speed;
      }
      if (rightPressed && ship.x < canvas.width - ship.width) {
        ship.x += ship.speed;
      }
    }
    setInterval(moveShip, 20);

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
      // 弾を発射（自機中央から出す）
      bullets.push({ x: ship.x + ship.width / 2 - 2, y: ship.y });
    });

    // デスクトップでの操作確認用（マウス操作）
    leftBtn.addEventListener("mousedown", () => { leftPressed = true; });
    leftBtn.addEventListener("mouseup", () => { leftPressed = false; });
    rightBtn.addEventListener("mousedown", () => { rightPressed = true; });
    rightBtn.addEventListener("mouseup", () => { rightPressed = false; });
    fireBtn.addEventListener("mousedown", () => {
      bullets.push({ x: ship.x + ship.width / 2 - 2, y: ship.y });
    });
  </script>
</body>
</html>
