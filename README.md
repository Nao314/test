<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>簡単なクリックゲーム</title>
  <style>
    body {
      text-align: center;
      font-family: Arial, sans-serif;
      margin-top: 50px;
    }
    #game-area {
      margin: auto;
      width: 300px;
      height: 300px;
      border: 2px solid #333;
      position: relative;
    }
    .target {
      width: 50px;
      height: 50px;
      background-color: red;
      border-radius: 50%;
      position: absolute;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <h1>簡単なクリックゲーム</h1>
  <p>赤いターゲットをクリックしてスコアを獲得しましょう！</p>
  <p>スコア: <span id="score">0</span></p>
  <div id="game-area">
    <div id="target" class="target"></div>
  </div>
  <script>
    const scoreElement = document.getElementById('score');
    const target = document.getElementById('target');
    const gameArea = document.getElementById('game-area');
    let score = 0;

    // ゲームエリア内でランダムな位置にターゲットを移動させる関数
    function moveTarget() {
      const maxX = gameArea.clientWidth - target.clientWidth;
      const maxY = gameArea.clientHeight - target.clientHeight;
      const randomX = Math.floor(Math.random() * (maxX + 1));
      const randomY = Math.floor(Math.random() * (maxY + 1));
      target.style.left = randomX + 'px';
      target.style.top = randomY + 'px';
    }

    // ターゲットがクリックされたときの処理
    target.addEventListener('click', function() {
      score++;
      scoreElement.textContent = score;
      moveTarget();
    });

    // 初期状態でターゲットを移動
    moveTarget();
  </script>
</body>
</html>
