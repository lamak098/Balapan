<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>🚗 Game Balapan Mobil vs Lawan</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    body {
      background: linear-gradient(135deg, #1a1a2e, #16213e);
      min-height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      font-family: 'Segoe UI', Arial, sans-serif;
      padding: 20px;
    }
    .game-container {
      background: #0f3460;
      border-radius: 24px;
      padding: 20px;
      box-shadow: 0 20px 40px rgba(0,0,0,0.6);
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    canvas {
      display: block;
      border-radius: 16px;
      box-shadow: 0 0 30px rgba(233, 69, 96, 0.5);
      background: #2c3e50;
      max-width: 100%;
      height: auto;
    }
    .panel {
      width: 100%;
      margin-top: 20px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      flex-wrap: wrap;
      color: white;
      background: rgba(0,0,0,0.4);
      border-radius: 40px;
      padding: 15px 25px;
      backdrop-filter: blur(10px);
    }
    .score-box {
      background: #e94560;
      padding: 8px 24px;
      border-radius: 30px;
      font-weight: bold;
      font-size: 1.5rem;
      letter-spacing: 1px;
      box-shadow: 0 0 15px #e94560aa;
    }
    button {
      background: #f5f5f5;
      border: none;
      padding: 12px 28px;
      border-radius: 30px;
      font-weight: bold;
      font-size: 1.1rem;
      cursor: pointer;
      background: #e94560;
      color: white;
      letter-spacing: 0.5px;
      transition: all 0.2s ease;
      box-shadow: 0 4px 10px rgba(0,0,0,0.4);
      border: 1px solid #ff6b81;
    }
    button:hover {
      background: #ff6b81;
      transform: scale(1.03);
      box-shadow: 0 0 20px #e94560;
    }
    .controls {
      display: flex;
      gap: 15px;
      align-items: center;
      color: #ddd;
      font-size: 0.9rem;
    }
    .gameover-message {
      color: #ffcc00;
      font-weight: bold;
      font-size: 1.2rem;
      margin-left: 15px;
    }
    @media (max-width: 550px) {
      .panel {
        flex-direction: column;
        gap: 12px;
      }
    }
  </style>
</head>
<body>
<div style="display: flex; flex-direction: column; align-items: center;">
  <div class="game-container">
    <canvas id="gameCanvas" width="400" height="600" aria-label="Game balapan mobil"></canvas>
    <div class="panel">
      <div class="score-box" id="scoreDisplay">🏆 0</div>
      <div style="display: flex; align-items: center; gap: 15px; flex-wrap: wrap;">
        <div class="controls">
          <span>⬅️➡️ Gerak</span>
          <span style="color:#e94560;">|</span>
          <span>🔼 Gas</span>
        </div>
        <button id="restartButton">🔄 Mulai Ulang</button>
      </div>
      <div id="gameoverText" class="gameover-message"></div>
    </div>
  </div>
  <div style="color:#ccc; margin-top: 15px; font-size: 0.9rem;">
    🎮 Gunakan <strong>panah kiri/kanan</strong> untuk menghindar, <strong>panah atas</strong> untuk percepat.
  </div>
</div>

<script>
  (function() {
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    // Skor
    let score = 0;
    let gameActive = true;
    let gameOverFlag = false;  // untuk menampilkan teks game over

    // Ukuran canvas
    const W = canvas.width;
    const H = canvas.height;

    // Mobil pemain
    const player = {
      x: W/2 - 25,
      y: H - 120,
      width: 40,
      height: 70,
      speed: 5,
    };

    // Mobil lawan (AI)
    const opponent = {
      x: W/2 - 25,
      y: 120,
      width: 40,
      height: 70,
      speed: 2.8,
      // AI akan bergerak secara otomatis
    };

    // Array rintangan (mobil lain / penghalang)
    let obstacles = [];

    // Kecepatan dasar permainan
    const baseGameSpeed = 3.2;

    // Status kontrol
    let leftPressed = false;
    let rightPressed = false;
    let upPressed = false;

    // Batas area jalan (anggap jalan dari x=40 sampai x=W-40)
    const roadLeft = 30;
    const roadRight = W - 30 - player.width; // disesuaikan dengan lebar mobil

    // --- Fungsi reset game ---
    function resetGame() {
      gameActive = true;
      gameOverFlag = false;
      score = 0;
      updateScoreDisplay();
      document.getElementById('gameoverText').innerText = '';

      // Reset posisi pemain
      player.x = W/2 - player.width/2;
      player.y = H - 120;

      // Reset posisi lawan
      opponent.x = W/2 - opponent.width/2;
      opponent.y = 120;

      // Hapus semua rintangan
      obstacles = [];

      // Reset input
      leftPressed = false;
      rightPressed = false;
      upPressed = false;
    }

    // Skor UI
    function updateScoreDisplay() {
      document.getElementById('scoreDisplay').innerText = `🏆 ${score}`;
    }

    // --- Cek tabrakan antar dua persegi panjang ---
    function checkCollision(rect1, rect2) {
      return rect1.x < rect2.x + rect2.width &&
             rect1.x + rect1.width > rect2.x &&
             rect1.y < rect2.y + rect2.height &&
             rect1.y + rect1.height > rect2.y;
    }

    // --- Spawn rintangan baru (mobil netral yang bergerak ke bawah) ---
    function spawnObstacle() {
      // Jangan terlalu banyak rintangan
      if (obstacles.length > 4) return;

      const width = 40;
      const height = 70;
      // Posisi acak di jalan
      const minX = roadLeft;
      const maxX = roadRight;
      const x = Math.floor(Math.random() * (maxX - minX + 1)) + minX;
      // Mulai dari atas layar (di luar)
      const y = -height - Math.random() * 120;

      obstacles.push({
        x: x,
        y: y,
        width: width,
        height: height,
        speed: baseGameSpeed + Math.random() * 1.8, // sedikit variasi kecepatan
      });
    }

    // --- Update posisi pemain berdasarkan input ---
    function updatePlayerPosition() {
      if (!gameActive) return;

      let moveSpeed = player.speed;
      // Jika tombol atas ditekan, pemain bergerak lebih cepat (efek gas)
      if (upPressed) {
        moveSpeed = player.speed + 2.2;
      }

      if (leftPressed) {
        player.x -= moveSpeed;
      }
      if (rightPressed) {
        player.x += moveSpeed;
      }

      // Batasi mobil di jalan
      if (player.x < roadLeft) player.x = roadLeft;
      if (player.x > roadRight) player.x = roadRight;
    }

    // --- Update pergerakan lawan (AI) ---
    function updateOpponentAI() {
      if (!gameActive) return;

      // Lawan bergerak ke bawah secara konstan (seperti sedang balapan)
      opponent.y += opponent.speed;

      // AI mencoba menghindari rintangan dan pemain dengan gerakan horizontal sederhana
      // Cari rintangan terdekat di depan lawan (di bawahnya)
      let targetX = opponent.x;
      let threatDetected = false;

      // Periksa rintangan yang ada di depan (y > opponent.y)
      for (let obs of obstacles) {
        if (obs.y > opponent.y && obs.y < opponent.y + 180) {
          // Jika ada rintangan dalam jalur horizontal yang sama (sekitar lebar mobil)
          const opponentCenter = opponent.x + opponent.width/2;
          const obsCenter = obs.x + obs.width/2;
          const distanceX = Math.abs(opponentCenter - obsCenter);

          // Jika cukup dekat secara horizontal, hindari
          if (distanceX < (opponent.width/2 + obs.width/2 + 20)) {
            threatDetected = true;
            // Tentukan arah menghindar: menjauh dari rintangan
            if (opponentCenter < obsCenter) {
              targetX = opponent.x - 2.5; // bergerak kiri
            } else {
              targetX = opponent.x + 2.5; // bergerak kanan
            }
            break; // prioritas satu ancaman dulu
          }
        }
      }

      // Selain rintangan, lawan juga menghindari pemain jika terlalu dekat secara vertikal?
      // Bisa ditambahkan, tapi agar tidak terlalu rumit, fokus ke rintangan.

      // Jika tidak ada ancaman, lawan cenderung kembali ke tengah atau sedikit random?
      if (!threatDetected) {
        // Kembali perlahan ke area tengah jalan agar tidak menabrak tepi
        const centerX = W/2 - opponent.width/2;
        if (opponent.x < centerX - 8) {
          targetX = opponent.x + 1.4;
        } else if (opponent.x > centerX + 8) {
          targetX = opponent.x - 1.4;
        } else {
          // sedikit gerakan acak kecil biar natural
          targetX = opponent.x + (Math.random() * 1.0 - 0.5);
        }
      }

      // Terapkan targetX dengan batasan
      opponent.x = targetX;

      // Batasi lawan di jalan
      if (opponent.x < roadLeft) opponent.x = roadLeft;
      if (opponent.x > roadRight) opponent.x = roadRight;

      // Jika lawan keluar dari layar bawah (reset ke atas, seperti loop)
      if (opponent.y > H + 80) {
        opponent.y = -opponent.height - Math.random() * 100;
        opponent.x = Math.random() * (roadRight - roadLeft) + roadLeft;
        // Tambah skor karena berhasil melewati lawan?
        // Sebenarnya pemain mendapat skor saat lawan lewat? Kita atur skor bertambah saat lawan berhasil dihindari.
        // Tapi lebih baik skor bertambah seiring waktu.
      }
    }

    // --- Update rintangan & skor ---
    function updateObstacles() {
      if (!gameActive) return;

      // Gerakkan semua rintangan ke bawah
      for (let obs of obstacles) {
        obs.y += obs.speed;
      }

      // Hapus rintangan yang sudah keluar layar bawah, dan tambah skor
      obstacles = obstacles.filter(obs => {
        if (obs.y > H + 100) {
          // Pemain berhasil menghindari rintangan ini -> tambah skor
          if (gameActive) {
            score += 10;
            updateScoreDisplay();
          }
          return false; // hapus
        }
        return true;
      });

      // Spawn rintangan baru secara probabilistik
      if (gameActive && Math.random() < 0.018) {
        spawnObstacle();
      }

      // Pastikan selalu ada minimal 1 rintangan setelah beberapa saat
      if (gameActive && obstacles.length === 0 && Math.random() < 0.1) {
        spawnObstacle();
      }
    }

    // --- Cek semua tabrakan ---
    function checkAllCollisions() {
      if (!gameActive) return;

      // 1. Tabrakan pemain dengan rintangan
      for (let obs of obstacles) {
        if (checkCollision(player, obs)) {
          gameOver();
          return;
        }
      }

      // 2. Tabrakan pemain dengan lawan
      if (checkCollision(player, opponent)) {
        gameOver();
        return;
      }

      // 3. Tabrakan lawan dengan rintangan (opsional: bisa membuat lawan hancur atau reset)
      // Biar adil, jika lawan menabrak rintangan, lawan di-reset ke atas (tidak game over untuk pemain)
      for (let obs of obstacles) {
        if (checkCollision(opponent, obs)) {
          // Reset posisi lawan ke atas agar tidak hilang begitu saja
          opponent.y = -opponent.height - 20;
          opponent.x = Math.random() * (roadRight - roadLeft) + roadLeft;
          // Bisa juga beri efek visual, tapi sederhana saja.
          break;
        }
      }
    }

    // Game over
    function gameOver() {
      if (!gameActive) return; // sudah game over sebelumnya
      gameActive = false;
      gameOverFlag = true;
      document.getElementById('gameoverText').innerText = '💥 GAME OVER 💥';
    }

    // --- Loop game utama ---
    function gameLoop() {
      if (gameActive) {
        updatePlayerPosition();
        updateOpponentAI();
        updateObstacles();
        checkAllCollisions();

        // Tambah skor berdasarkan waktu (bertahan hidup)
        score += 0.2;
        updateScoreDisplay();
      }

      drawEverything();
      requestAnimationFrame(gameLoop);
    }

    // --- Menggambar semua objek ---
    function drawEverything() {
      ctx.clearRect(0, 0, W, H);

      // Gambar jalan & marka
      drawRoad();

      // Gambar rintangan (mobil lain)
      for (let obs of obstacles) {
        drawCar(obs.x, obs.y, obs.width, obs.height, '#f39c12', '#e67e22'); // oranye
      }

      // Gambar mobil lawan (AI)
      drawCar(opponent.x, opponent.y, opponent.width, opponent.height, '#c0392b', '#e74c3c'); // merah tua

      // Gambar mobil pemain (warna biru cerah)
      drawCar(player.x, player.y, player.width, player.height, '#2980b9', '#3498db');

      // Tampilkan teks game over jika perlu
      if (gameOverFlag) {
        ctx.font = 'bold 34px "Segoe UI", Arial';
        ctx.fillStyle = '#ffcc00';
        ctx.shadowColor = '#000';
        ctx.shadowBlur = 14;
        ctx.textAlign = 'center';
        ctx.fillText('GAME OVER', W/2, H/2 - 20);
        ctx.shadowBlur = 0;
        ctx.shadowColor = 'transparent';
      }

      // Tampilkan instruksi kecil jika game aktif
      if (gameActive) {
        ctx.font = '12px "Segoe UI"';
        ctx.fillStyle = '#ecf0f1';
        ctx.textAlign = 'right';
        ctx.fillText('⚡ Hindari lawan & rintangan', W-25, 30);
      }
    }

    function drawRoad() {
      // Aspal
      ctx.fillStyle = '#2d3e50';
      ctx.fillRect(0, 0, W, H);
      // Garis tepi jalan
      ctx.strokeStyle = '#f1c40f';
      ctx.lineWidth = 4;
      ctx.beginPath();
      ctx.moveTo(roadLeft, 0);
      ctx.lineTo(roadLeft, H);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(roadRight + player.width, 0);
      ctx.lineTo(roadRight + player.width, H);
      ctx.stroke();

      // Garis putus-putus tengah
      ctx.strokeStyle = '#ecf0f1';
      ctx.lineWidth = 3;
      ctx.setLineDash([25, 25]);
      ctx.beginPath();
      ctx.moveTo(W/2, 0);
      ctx.lineTo(W/2, H);
      ctx.stroke();
      ctx.setLineDash([]); // reset
    }

    // Fungsi menggambar mobil sederhana
    function drawCar(x, y, w, h, bodyColor, windowColor) {
      ctx.save();
      // Body
      ctx.fillStyle = bodyColor;
      ctx.shadowColor = '#00000040';
      ctx.shadowBlur = 8;
      // bentuk mobil dengan border radius kecil
      ctx.beginPath();
      ctx.roundRect(x, y, w, h, 12);
      ctx.fill();
      
      // Jendela depan (atas)
      ctx.fillStyle = windowColor;
      ctx.shadowBlur = 4;
      ctx.beginPath();
      ctx.roundRect(x + 6, y + 8, w - 12, 22, 8);
      ctx.fill();
      
      // Lampu depan (putih)
      ctx.fillStyle = '#f1f2f6';
      ctx.shadowBlur = 6;
      ctx.fillRect(x + 8, y + h-8, 8, 6);
      ctx.fillRect(x + w - 16, y + h-8, 8, 6);
      
      // Lampu belakang (merah) di atas? untuk mobil lawan arah bawah, lampu belakang di atas.
      ctx.fillStyle = '#e74c3c';
      ctx.fillRect(x + 8, y + 2, 8, 5);
      ctx.fillRect(x + w - 16, y + 2, 8, 5);
      
      // Roda
      ctx.fillStyle = '#1e1e1e';
      ctx.shadowBlur = 4;
      ctx.fillRect(x + 4, y + 10, 8, 14);
      ctx.fillRect(x + w - 12, y + 10, 8, 14);
      ctx.fillRect(x + 4, y + h - 22, 8, 14);
      ctx.fillRect(x + w - 12, y + h - 22, 8, 14);
      
      ctx.restore();
    }

    // Polyfill roundRect jika tidak ada
    if (!ctx.roundRect) {
      ctx.roundRect = function (x, y, w, h, r) {
        if (typeof r === 'number') r = { tl: r, tr: r, br: r, bl: r };
        ctx.beginPath();
        ctx.moveTo(x + r.tl, y);
        ctx.lineTo(x + w - r.tr, y);
        ctx.quadraticCurveTo(x + w, y, x + w, y + r.tr);
        ctx.lineTo(x + w, y + h - r.br);
        ctx.quadraticCurveTo(x + w, y + h, x + w - r.br, y + h);
        ctx.lineTo(x + r.bl, y + h);
        ctx.quadraticCurveTo(x, y + h, x, y + h - r.bl);
        ctx.lineTo(x, y + r.tl);
        ctx.quadraticCurveTo(x, y, x + r.tl, y);
        ctx.closePath();
      };
    }

    // --- Event listener untuk keyboard ---
    function handleKeyDown(e) {
      const key = e.key;
      if (key === 'ArrowLeft' || key === 'Left' || key === 'ArrowRight' || key === 'Right' || key === 'ArrowUp' || key === 'Up') {
        e.preventDefault();
      }
      if (key === 'ArrowLeft') leftPressed = true;
      if (key === 'ArrowRight') rightPressed = true;
      if (key === 'ArrowUp') upPressed = true;

      // Restart dengan R (opsional)
      if (key === 'r' || key === 'R') {
        e.preventDefault();
        resetGame();
      }
    }

    function handleKeyUp(e) {
      const key = e.key;
      if (key === 'ArrowLeft' || key === 'Left' || key === 'ArrowRight' || key === 'Right' || key === 'ArrowUp' || key === 'Up') {
        e.preventDefault();
      }
      if (key === 'ArrowLeft') leftPressed = false;
      if (key === 'ArrowRight') rightPressed = false;
      if (key === 'ArrowUp') upPressed = false;
    }

    // Tombol restart
    document.getElementById('restartButton').addEventListener('click', resetGame);

    // Pasang event listener
    window.addEventListener('keydown', handleKeyDown);
    window.addEventListener('keyup', handleKeyUp);

    // Mulai game
    resetGame(); // inisialisasi
    gameLoop();

    // Untuk mobile: cegah scroll saat sentuh canvas (opsional)
    canvas.addEventListener('touchstart', (e) => e.preventDefault());
    canvas.addEventListener('touchmove', (e) => e.preventDefault());
  })();
</script>
</body>
</html>
