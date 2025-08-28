<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Tetris — Single File</title>
<style>
  :root{
    --bg:#0b1220;
    --panel:#0f1b2b;
    --accent:#4fd1c5;
    --muted:#98a0b3;
    --cell-size:28px;
  }
  *{box-sizing:border-box;font-family:Inter,ui-sans-serif,system-ui,Arial}
  body{margin:0;min-height:100vh;background:linear-gradient(180deg,#031024 0%,var(--bg) 100%);color:#e6eef6;display:flex;align-items:center;justify-content:center;padding:24px}
  .wrap{display:flex;gap:24px;align-items:flex-start}
  .game {
    background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(0,0,0,0.04));
    border-radius:12px;padding:16px;display:flex;gap:16px;align-items:flex-start;box-shadow:0 8px 30px rgba(0,0,0,0.6);
  }
  canvas{background:#071018;border-radius:6px;image-rendering:pixelated}
  .sidebar{width:220px;background:var(--panel);padding:12px;border-radius:8px;display:flex;flex-direction:column;gap:12px}
  .box{background:transparent;padding:8px;border-radius:6px}
  h1{font-size:18px;margin:0;color:var(--accent)}
  .stat{display:flex;justify-content:space-between;font-size:14px;color:var(--muted);padding:6px 0}
  .controls{display:flex;gap:8px;flex-wrap:wrap}
  button{background:transparent;border:1px solid rgba(255,255,255,0.06);padding:8px 10px;border-radius:8px;color:#eaf6f4;cursor:pointer}
  button.secondary{background:rgba(255,255,255,0.02)}
  .small{font-size:13px;color:var(--muted)}
  .preview{
    width:calc(var(--cell-size) * 4);
    height:calc(var(--cell-size) * 4);
    background:#04101a;border-radius:6px;display:flex;align-items:center;justify-content:center;
  }
  footer.small{color:var(--muted);font-size:12px;margin-top:8px}
  .logo{font-weight:600;color:#bfeee8}
  .key{display:inline-block;padding:4px 6px;border-radius:4px;background:rgba(255,255,255,0.03);border:1px solid rgba(255,255,255,0.02);font-size:12px;margin-right:6px}
</style>
</head>
<body>
  <div class="wrap">
    <div class="game" role="application" aria-label="Tetris game">
      <canvas id="board" width="224" height="560" style="width:224px;height:560px"></canvas>
      <div class="sidebar" aria-hidden="false">
        <div class="box">
          <h1>TETRIS</h1>
          <div class="small">Controls</div>
          <div class="small" style="margin-top:8px">
            <span class="key">←</span>Move
            <span class="key">→</span>Move
            <span class="key">↑</span>Rotate
            <span class="key">↓</span>Soft drop
            <span class="key">Space</span>Hard drop
            <span class="key">P</span>Pause
          </div>
        </div>

        <div class="box">
          <div class="stat"><div>Score</div><div id="score">0</div></div>
          <div class="stat"><div>Level</div><div id="level">1</div></div>
          <div class="stat"><div>Lines</div><div id="lines">0</div></div>
        </div>

        <div class="box">
          <div class="small">Next</div>
          <div class="preview" id="nextPreview"></div>
        </div>

        <div class="box controls">
          <button id="startBtn">Start</button>
          <button id="pauseBtn" class="secondary">Pause</button>
          <button id="restartBtn" class="secondary">Restart</button>
        </div>

        <footer class="small">Made with ❤️ — drop in codepen, edit, have fun!</footer>
      </div>
    </div>
  </div>

<script>
/*
  Tetris single-file implementation.
  Grid: 10 x 20 (standard)
  Each cell uses a 'cellSize' consistent with canvas pixels.
*/

(() => {
  // config
  const COLS = 10;
  const ROWS = 20;
  const cellSize = 28; // matches CSS variable for clarity
  const canvas = document.getElementById('board');
  const ctx = canvas.getContext('2d');
  const nextPreview = document.getElementById('nextPreview');
  const scoreEl = document.getElementById('score');
  const levelEl = document.getElementById('level');
  const linesEl = document.getElementById('lines');
  const startBtn = document.getElementById('startBtn');
  const pauseBtn = document.getElementById('pauseBtn');
  const restartBtn = document.getElementById('restartBtn');

  canvas.width = COLS * cellSize;
  canvas.height = ROWS * cellSize;

  // Colors for tetrominoes
  const COLORS = {
    I: '#3ad6f2',
    J: '#3b7cff',
    L: '#ff9a3c',
    O: '#ffd23f',
    S: '#56d86c',
    T: '#b96cff',
    Z: '#ff5c67',
    X: '#222' // used for filled background if needed
  };

  // Tetromino shapes (rotation states)
  const SHAPES = {
    I: [
      [[0,0,0,0],[1,1,1,1],[0,0,0,0],[0,0,0,0]],
      [[0,0,1,0],[0,0,1,0],[0,0,1,0],[0,0,1,0]]
    ],
    J: [
      [[1,0,0],[1,1,1],[0,0,0]],
      [[0,1,1],[0,1,0],[0,1,0]],
      [[0,0,0],[1,1,1],[0,0,1]],
      [[0,1,0],[0,1,0],[1,1,0]]
    ],
    L: [
      [[0,0,1],[1,1,1],[0,0,0]],
      [[0,1,0],[0,1,0],[0,1,1]],
      [[0,0,0],[1,1,1],[1,0,0]],
      [[1,1,0],[0,1,0],[0,1,0]]
    ],
    O: [
      [[1,1],[1,1]]
    ],
    S: [
      [[0,1,1],[1,1,0],[0,0,0]],
      [[0,1,0],[0,1,1],[0,0,1]]
    ],
    T: [
      [[0,1,0],[1,1,1],[0,0,0]],
      [[0,1,0],[0,1,1],[0,1,0]],
      [[0,0,0],[1,1,1],[0,1,0]],
      [[0,1,0],[1,1,0],[0,1,0]]
    ],
    Z: [
      [[1,1,0],[0,1,1],[0,0,0]],
      [[0,0,1],[0,1,1],[0,1,0]]
    ]
  };

  const PIECES = Object.keys(SHAPES);

  // Utility: make empty grid
  function createGrid(cols, rows) {
    const g = new Array(rows);
    for (let r=0;r<rows;r++){
      g[r] = new Array(cols).fill('');
    }
    return g;
  }

  // Game state
  let grid = createGrid(COLS, ROWS);
  let current = null;
  let next = null;
  let dropInterval = 800; // ms
  let dropTimer = null;
  let lastTime = 0;
  let score = 0;
  let lines = 0;
  let level = 1;
  let isPaused = false;
  let isRunning = false;

  // Random bag generator (standard 7-bag)
  function generateBag(){
    const bag = PIECES.slice();
    for (let i=bag.length-1;i>0;i--){
      const j = Math.floor(Math.random() * (i+1));
      [bag[i], bag[j]] = [bag[j], bag[i]];
    }
    return bag;
  }
  let bag = generateBag();

  function nextPieceFromBag(){
    if (bag.length === 0) bag = generateBag();
    const type = bag.pop();
    const shape = SHAPES[type];
    return {
      type,
      rotationIndex: 0,
      shape: shape[0],
      shapes: shape,
      x: Math.floor((COLS - shape[0][0].length) / 2),
      y: - shape[0].length // spawn above visible area
    };
  }

  // Spawn new piece
  function spawn(){
    current = next || nextPieceFromBag();
    next = nextPieceFromBag();
    // reset rotation to match current rotationIndex
    current.shape = current.shapes[current.rotationIndex % current.shapes.length];
    // center horizontally
    current.x = Math.floor((COLS - current.shape[0].length) / 2);
    current.y = -current.shape.length;
    if (collides(current, current.x, current.y)){
      // game over
      gameOver();
    }
    renderNext();
  }

  // collision test
  function collides(piece, x, y){
    const shape = piece.shape;
    for (let r=0;r<shape.length;r++){
      for (let c=0;c<shape[r].length;c++){
        if (shape[r][c]){
          const gx = x + c;
          const gy = y + r;
          if (gx < 0 || gx >= COLS || gy >= ROWS) return true;
          if (gy >= 0 && grid[gy][gx]) return true;
        }
      }
    }
    return false;
  }

  // merge piece into grid
  function lockPiece(){
    const s = current.shape;
    for (let r=0;r<s.length;r++){
      for (let c=0;c<s[r].length;c++){
        if (s[r][c]){
          const gx = current.x + c;
          const gy = current.y + r;
          if (gy >= 0 && gy < ROWS && gx >=0 && gx < COLS){
            grid[gy][gx] = current.type;
          }
        }
      }
    }
    clearLines();
    spawn();
  }

  function clearLines(){
    let cleared = 0;
    for (let r = ROWS-1; r >= 0; r--){
      if (grid[r].every(cell => cell !== '')){
        grid.splice(r,1);
        grid.unshift(new Array(COLS).fill(''));
        cleared++;
        r++; // re-check same row index after splice
      }
    }
    if (cleared > 0){
      // scoring per Tetris standard (simplified)
      const pointsMap = {1:40,2:100,3:300,4:1200};
      score += (pointsMap[cleared] || 0) * level;
      lines += cleared;
      // level up every 10 lines
      const newLevel = Math.floor(lines / 10) + 1;
      if (newLevel !== level){
        level = newLevel;
        dropInterval = Math.max(80, 800 - (level-1) * 60); // speed up
      }
      updateHUD();
    }
  }

  function updateHUD(){
    scoreEl.textContent = score;
    levelEl.textContent = level;
    linesEl.textContent = lines;
  }

  // piece movement
  function move(dx){
    if (!current) return;
    if (!collides(current, current.x + dx, current.y)){
      current.x += dx;
      draw();
    }
  }

  function rotate(dir = 1){
    if (!current) return;
    const nextIndex = (current.rotationIndex + dir + current.shapes.length) % current.shapes.length;
    const nextShape = current.shapes[nextIndex];
    // try wall kicks: basic horizontal test offsets
    const kicks = [0, -1, 1, -2, 2];
    for (let k of kicks){
      const nx = current.x + k;
      const temp = { ...current, shape: nextShape, rotationIndex: nextIndex, x: nx };
      if (!collides(temp, nx, current.y)){
        current.shape = nextShape;
        current.rotationIndex = nextIndex;
        current.x = nx;
        draw();
        return;
      }
    }
  }

  function softDrop(){
    if (!current) return;
    if (!collides(current, current.x, current.y + 1)){
      current.y++;
      score += 1; // soft drop small reward
      updateHUD();
      draw();
    } else {
      lockPiece();
    }
  }

  function hardDrop(){
    if (!current) return;
    while (!collides(current, current.x, current.y + 1)){
      current.y++;
      score += 2; // harder reward
    }
    lockPiece();
    updateHUD();
    draw();
  }

  // drawing
  function drawCell(x,y,type){
    const px = x * cellSize;
    const py = y * cellSize;
    ctx.fillStyle = '#05141a';
    ctx.fillRect(px+1, py+1, cellSize-2, cellSize-2);
    if (!type) return;
    ctx.fillStyle = COLORS[type] || '#888';
    ctx.strokeStyle = 'rgba(0,0,0,0.2)';
    ctx.lineWidth = 2;
    // main
    ctx.fillRect(px+2, py+2, cellSize-4, cellSize-4);
    // border highlight
    ctx.strokeRect(px+2, py+2, cellSize-4, cellSize-4);
  }

  function clearCanvas(){
    ctx.clearRect(0,0,canvas.width,canvas.height);
  }

  function draw(){
    clearCanvas();
    // draw grid cells
    for (let r=0;r<ROWS;r++){
      for (let c=0;c<COLS;c++){
        const t = grid[r][c];
        drawCell(c,r,t);
      }
    }
    // draw current piece
    if (current){
      const s = current.shape;
      for (let r=0;r<s.length;r++){
        for (let c=0;c<s[r].length;c++){
          if (s[r][c]){
            const gx = current.x + c;
            const gy = current.y + r;
            if (gy >= 0){
              drawCell(gx, gy, current.type);
            }
          }
        }
      }
    }
  }

  // Next piece preview drawing (small canvas inside a div)
  function renderNext(){
    nextPreview.innerHTML = '';
    const cvs = document.createElement('canvas');
    const scale = 14; // smaller cell for preview
    cvs.width = 4 * scale;
    cvs.height = 4 * scale;
    cvs.style.width = `${4*scale}px`;
    cvs.style.height = `${4*scale}px`;
    cvs.style.imageRendering = 'pixelated';
    nextPreview.appendChild(cvs);
    const nx = cvs.getContext('2d');
    nx.fillStyle = '#04101a';
    nx.fillRect(0,0,cvs.width,cvs.height);

    const s = next.shape;
    const w = s[0].length;
    const h = s.length;
    const offsetX = Math.floor((4 - w)/2);
    const offsetY = Math.floor((4 - h)/2);
    for (let r=0;r<h;r++){
      for (let c=0;c<w;c++){
        if (s[r][c]){
          nx.fillStyle = COLORS[next.type] || '#fff';
          nx.fillRect((offsetX + c) * scale + 1, (offsetY + r) * scale + 1, scale-2, scale-2);
        }
      }
    }
  }

  // game loop / falling
  function tick(){
    if (!isRunning || isPaused) return;
    if (!current) spawn();
    if (!collides(current, current.x, current.y + 1)){
      current.y++;
    } else {
      lockPiece();
    }
    draw();
    // schedule next tick based on current dropInterval
    clearTimeout(dropTimer);
    dropTimer = setTimeout(tick, dropInterval);
  }

  // input
  const keyState = {};
  window.addEventListener('keydown', (e) => {
    if (!isRunning) return;
    if (e.repeat) return; // prevent repeating except for soft drop behavior manually
    if (e.key === 'ArrowLeft'){
      move(-1);
    } else if (e.key === 'ArrowRight'){
      move(1);
    } else if (e.key === 'ArrowUp'){
      rotate(1);
    } else if (e.key === 'ArrowDown'){
      // start soft drop; apply once and then allow hold via repeat
      softDrop();
    } else if (e.code === 'Space'){
      e.preventDefault();
      hardDrop();
    } else if (e.key.toLowerCase() === 'p'){
      togglePause();
    }
  });

  // basic mouse / button controls
  startBtn.addEventListener('click', () => {
    startGame();
  });
  pauseBtn.addEventListener('click', () => {
    togglePause();
  });
  restartBtn.addEventListener('click', () => {
    resetGame();
    startGame();
  });

  // start / pause / reset
  function startGame(){
    if (isRunning && !isPaused) return;
    if (!isRunning){
      resetGame();
      isRunning = true;
      spawn();
    }
    isPaused = false;
    pauseBtn.textContent = 'Pause';
    updateHUD();
    draw();
    clearTimeout(dropTimer);
    dropTimer = setTimeout(tick, dropInterval);
  }

  function togglePause(){
    if (!isRunning) return;
    isPaused = !isPaused;
    pauseBtn.textContent = isPaused ? 'Resume' : 'Pause';
    if (!isPaused){
      clearTimeout(dropTimer);
      dropTimer = setTimeout(tick, dropInterval);
    } else {
      clearTimeout(dropTimer);
    }
  }

  function resetGame(){
    grid = createGrid(COLS, ROWS);
    bag = generateBag();
    next = nextPieceFromBag();
    current = null;
    score = 0;
    lines = 0;
    level = 1;
    dropInterval = 800;
    isPaused = false;
    isRunning = false;
    clearTimeout(dropTimer);
    updateHUD();
    renderNext();
    clearCanvas();
  }

  function gameOver(){
    isRunning = false;
    clearTimeout(dropTimer);
    ctx.fillStyle = 'rgba(0,0,0,0.6)';
    ctx.fillRect(0, canvas.height/2 - 40, canvas.width, 90);
    ctx.fillStyle = '#fff';
    ctx.font = '24px Inter, Arial';
    ctx.textAlign = 'center';
    ctx.fillText('GAME OVER', canvas.width/2, canvas.height/2);
    ctx.font = '14px Inter, Arial';
    ctx.fillText('Click Start to play again', canvas.width/2, canvas.height/2 + 26);
  }

  // initialize small
  resetGame();
  // draw empty board
  draw();

  // expose to window for debug
  window.tetris = { startGame, resetGame, move, rotate, softDrop, hardDrop };

  // Optional: auto-focus for keyboard
  canvas.tabIndex = 0;
  canvas.style.outline = 'none';
  canvas.addEventListener('focus', () => {});
  canvas.focus();<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
 
<script src="https://cdnjs.cloudflare.com/ajax/libs/howler/2.2.4/howler.min.js"></script>
<script>
  var music = new Howl({
    src: ['retro-song.mp3'],
    loop: true,
    volume: 0.5
  });

  music.play();
</script>

})()
</script>
</body>
</html>
