<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Simple Ludo Game - Project</title>
<style>
  :root{
    --bg:#85e912;
    --board:#ad1e1e;
    --tile:#e711b5;
    --font: "Poppins", system-ui, sans-serif;
  }
  body{
    background:var(--bg);
    font-family:var(--font);
    display:flex;
    align-items:flex-start;
    justify-content:center;
    padding:20px;
    gap:20px;
    min-height:100vh;
  }
  .container{
    display:flex;
    gap:20px;
    align-items:flex-start;
  }
  canvas{
    background:linear-gradient(180deg,#af2222,#14589c);
    border-radius:12px;
    box-shadow:0 8px 30px rgba(20,30,70,0.08);
  }
  .panel{
    width:320px;
    background:var(--board);
    border-radius:12px;
    padding:18px;
    box-shadow:0 8px 30px rgba(53, 87, 222, 0.06);
  }
  h2{ margin-bottom:8px; font-size:18px }
  label{display:block;margin-top:8px;font-size:13px;color:#555}
  select, button{
    width:100%;
    padding:10px 12px;
    border-radius:8px;
    border:1px solid #071223;
    background:rgb(155, 176, 12);
    margin-top:8px;
    font-weight:600;
    cursor:pointer;
  }
  .info{
    margin-top:12px;
    background:#3f63cc;
    padding:10px;
    border-radius:8px;
    font-size:14px;
  }
  .dice{
    display:flex;
    justify-content:space-between;
    gap:8px;
    margin-top:12px;
  }
  .dice-result{
    font-size:28px;
    font-weight:700;
    color:#0b3b8c;
    flex:1;
    text-align:center;
    background:linear-gradient(180deg,#2057af,#981616);
    padding:10px;border-radius:8px;border:1px solid #0e319c;
  }
  .token-buttons{
    display:flex;flex-wrap:wrap;gap:8px;margin-top:12px;
  }
  .token-btn{
    flex:1;
    padding:8px;
    border-radius:8px;
    font-weight:700;
    border:1px solid #1d59b9;
    background:rgb(152, 37, 37);
  }
  .small{font-size:13px;color:#444;margin-top:8px}
  .score-list{margin-top:12px}
  .player-row{display:flex;justify-content:space-between;padding:8px;border-radius:6px;margin-bottom:6px}
  .player-row span{font-weight:700}
  .badge{background:#123aaa;padding:6px 8px;border-radius:8px;font-weight:700}
  footer{margin-top:12px;font-size:12px;color:#777}
  .hint{margin-top:8px;font-size:13px;color:#555}
</style>
</head>
<body>

<div class="container">
  <canvas id="board" width="540" height="540"></canvas>

  <div class="panel">
    <h2>Simple Ludo — Playable Demo</h2>

    <label>Number of Players</label>
    <select id="playerCount">
      <option value="2">2 Players (Blue, Red)</option>
      <option value="3">3 Players (Blue, Red, Green)</option>
      <option value="4" selected>4 Players (Blue, Red, Green, Yellow)</option>
    </select>

    <label>Start Game</label>
    <button id="startBtn">Start / Restart</button>

    <div class="info">
      <div><strong>Current:</strong> <span id="currentPlayer">—</span></div>
      <div style="margin-top:6px"><strong>Dice:</strong> <span class="badge" id="diceVal">—</span></div>
    </div>

    <div class="dice">
      <button id="rollBtn">Roll Dice</button>
      <div class="dice-result" id="diceDisplay">—</div>
    </div>

    <div class="hint">Click a token button to move it after a valid roll. Need a 6 to bring a token out of home.</div>

    <label style="margin-top:12px">Select Token to Move</label>
    <div id="tokenBtns" class="token-buttons"></div>

    <div class="score-list" id="scoreList"></div>

    <footer>
      Rules (simplified): Need 6 to release from home. Capture opponent on landing (unless square is safe). Exact roll to finish one token. First player to finish all tokens wins.
    </footer>
  </div>
</div>

<script>
/* ---------- Simple Ludo Logic (works offline in one file) ----------

Implementation notes:
- Track is 52 squares (0..51). Each player has a start index: 0(blue),13(red),26(green),39(yellow).
- Each token stores stepsMoved:
    - -1 : at home (not on board)
    - 0..51 : on board, steps moved since entering
    - 52 : finished (completed full 52 steps)
- Board index for a token = (playerStartIndex + stepsMoved) % 52
- To release from home, roll must be 6, token stepsMoved becomes 0.
- To finish, token.stepsMoved + roll must equal 52 exactly.
- Safe squares are the 4 start positions of players (indices above).
- On landing, if opponent tokens exist on that board index and it's not a safe square, they are sent home.
- Players take turns; rolling a 6 gives another roll.
------------------------------------------------------------------*/

const canvas = document.getElementById('board');
const ctx = canvas.getContext('2d');

const W = canvas.width;
const H = canvas.height;

const playerCountSelect = document.getElementById('playerCount');
const startBtn = document.getElementById('startBtn');
const rollBtn = document.getElementById('rollBtn');
const diceDisplay = document.getElementById('diceDisplay');
const diceValBadge = document.getElementById('diceVal');
const currentPlayerLabel = document.getElementById('currentPlayer');
const tokenBtnsDiv = document.getElementById('tokenBtns');
const scoreList = document.getElementById('scoreList');

let NUM_PLAYERS = 4;
const COLORS = ['#0b61ff','#e22b2b','#179a2b','#f1c40f']; // Blue, Red, Green, Yellow
const NAMES = ['Blue','Red','Green','Yellow'];
const START_IDX = [0,13,26,39];
const SAFE_INDICES = new Set(START_IDX); // basic safe squares

let game = null;

function newGame(numPlayers){
  NUM_PLAYERS = numPlayers;
  // initialize players and tokens
  const players = [];
  for(let p=0;p<NUM_PLAYERS;p++){
    const tokens = [];
    for(let t=0;t<4;t++){
      tokens.push({ id:t, steps:-1 }); // -1 at home
    }
    players.push({ id:p, name:NAMES[p], color:COLORS[p], tokens, finished:0 });
  }

  game = {
    players,
    cur: 0,
    dice: null,
    extraTurn: false,
    winnerOrder: []
  };

  updateUI();
  drawBoard();
}

function rollDice(){
  if(!game) return;
  const r = Math.floor(Math.random()*6)+1;
  game.dice = r;
  diceDisplay.textContent = r;
  diceValBadge.textContent = r;
  updateTokenButtons();
  // clicking a token (or if there is only a forced move) will apply move
}

function updateUI(){
  currentPlayerLabel.textContent = game ? `${game.players[game.cur].name}` : '—';
  diceDisplay.textContent = game && game.dice !== null ? game.dice : '—';
  diceValBadge.textContent = game && game.dice !== null ? game.dice : '—';
  renderScores();
  updateTokenButtons();
  drawBoard();
}

function renderScores(){
  scoreList.innerHTML = '';
  game.players.forEach(p=>{
    const div = document.createElement('div');
    div.className = 'player-row';
    div.style.background = p.color + '15';
    div.innerHTML = `<span style="color:${p.color}">${p.name}</span><span>${p.finished}/4</span>`;
    scoreList.appendChild(div);
  });
}

function updateTokenButtons(){
  tokenBtnsDiv.innerHTML = '';
  if(!game) return;
  const player = game.players[game.cur];
  const dice = game.dice;
  // Create one button per token showing state
  player.tokens.forEach(token=>{
    const btn = document.createElement('button');
    btn.className = 'token-btn';
    const stateText = token.steps === -1 ? 'Home' : (token.steps>=52 ? 'Finished' : `On ${boardIndex(player.id, token.steps)}`);
    btn.textContent = `T${token.id+1}: ${stateText}`;
    if(dice === null){
      btn.disabled = true;
      btn.title = 'Roll dice first';
    } else {
      const can = canMoveToken(player.id, token.id, dice);
      btn.disabled = !can;
      btn.title = can ? 'Click to move' : 'Cannot move this token';
      if(can) btn.style.border = `2px solid ${player.color}`;
    }
    btn.addEventListener('click', ()=> handleMove(player.id, token.id));
    tokenBtnsDiv.appendChild(btn);
  });
}

function canMoveToken(playerId, tokenId, dice){
  const player = game.players[playerId];
  const token = player.tokens[tokenId];
  if(token.steps >= 52) return false; // already finished
  // if token at home
  if(token.steps === -1){
    return dice === 6; // need a 6 to come out
  } else {
    // must not overshoot 52
    return (token.steps + dice) <= 52;
  }
}

// compute board index for a player's token given steps moved (0..51)
function boardIndex(playerId, steps){
  if(steps < 0) return null;
  if(steps >= 52) return null;
  return (START_IDX[playerId] + steps) % 52;
}

// handle move of a token (after valid roll)
function handleMove(playerId, tokenId){
  const dice = game.dice;
  if(dice === null) return;
  const player = game.players[playerId];
  const token = player.tokens[tokenId];
  // validate
  if(!canMoveToken(playerId, tokenId, dice)) return;

  // moving from home
  if(token.steps === -1){
    token.steps = 0;
  } else {
    token.steps += dice;
  }

  // if token reached exactly 52 -> finished
  if(token.steps === 52){
    player.finished++;
    // mark finished; no board index
  }

  // if token is on board and not finished, check capture
  if(token.steps >=0 && token.steps < 52){
    const idx = boardIndex(playerId, token.steps);
    // capture opponents on this index unless safe square
    if(!SAFE_INDICES.has(idx)){
      game.players.forEach((op, opId)=>{
        if(opId === playerId) return;
        op.tokens.forEach(opT=>{
          if(opT.steps >=0 && opT < 52){
            const opIdx = boardIndex(opId, opT.steps);
            if(opIdx === idx){
              // send opponent token home
              opT.steps = -1;
            }
          }
        });
      });
    }
  }

  // clear dice (unless 6 -> extra turn)
  if(dice === 6){
    game.extraTurn = true;
    // allow another roll for same player
    game.dice = null;
    updateUI();
    // keep same current player
    checkWinAndMaybeContinue(playerId);
    return;
  } else {
    game.extraTurn = false;
    game.dice = null;
    // advance to next alive player
    advanceTurn();
  }
  updateUI();
  checkWinAndMaybeContinue(playerId);
}

function advanceTurn(){
  // find next player that is still in game (not finished all tokens)
  let next = game.cur;
  for(let i=1;i<=NUM_PLAYERS;i++){
    const cand = (game.cur + i) % NUM_PLAYERS;
    // allow even if finished: they can have tokens but if they finished all tokens we may skip them
    if(game.players[cand].finished < 4){
      next = cand;
      break;
    }
  }
  game.cur = next;
  game.dice = null;
  updateUI();
}

function checkWinAndMaybeContinue(playerId){
  const player = game.players[playerId];
  if(player.finished === 4 && !game.winnerOrder.includes(playerId)){
    game.winnerOrder.push(playerId);
    alert(`${player.name} finished all tokens! Place: ${game.winnerOrder.length}`);
    // check if only one player left => game over
    const alive = game.players.filter(p=>p.finished < 4).length;
    if(alive <= 1){
      // show final order
      const order = game.winnerOrder.map((id,i)=> `${i+1}. ${game.players[id].name}`).join('\n');
      const remaining = game.players.filter(p=>p.finished <4).map(p=>p.name);
      let msg = 'Game Over!\n\nFinal order:\n' + order;
      if(remaining.length) msg += '\nRemaining players: ' + remaining.join(', ');
      alert(msg);
    } else {
      // continue game skipping finished players
      if(game.players[game.cur].finished === 4) advanceTurn();
    }
  }
}

/* -------------------- Board drawing -------------------- */

const GRID = 15; // used for positioning (visual)
const MARGIN = 18;
const TILE = (W - 2*MARGIN) / GRID;

function buildTrackCoords(){
  // We'll create the 52 track positions roughly matching a classic Ludo path visually.
  const coords = [];

  // We'll trace along outer path: top row left->right (6), right column top->down (6), bottom row right->left (6), left column bottom->up (6)
  // But to get 52 cells we will follow a more ludo-like path: manual sequence of coordinates across grid cells
  // We will push coordinates (x,y) in pixel center positions.

  // Hardcode an order (approx) using grid cells indices (col,row) 0..14 forming ludo layout
  // This layout forms a cross-shaped track:
  const pathCells = [
    // top left area moving right across top track
    [6,0],[7,0],[8,0],[9,0],[10,0],[11,0],
    [11,1],[11,2],[11,3],[11,4],[11,5],
    [12,5],[13,5],[14,5],
    [14,6],[14,7],[14,8],[14,9],[14,10],
    [13,10],[12,10],[11,10],
    [11,11],[11,12],[11,13],[11,14],
    [10,14],[9,14],[8,14],[7,14],[6,14],
    [6,13],[6,12],[6,11],
    [5,11],[4,11],[3,11],[2,11],[1,11],
    [1,10],[0,10],[0,9],[0,8],[0,7],
    [1,7],[2,7],[3,7],[4,7],[5,7],
    [5,6],[5,5],[6,5],[7,5]
  ];
  // But above is more than 52, we'll trim to 52 and try to make it cyclical. We'll pick 52 first unique positions.
  const unique = [];
  const seen = new Set();
  for(const c of pathCells){
    const key = c[0]+','+c[1];
    if(!seen.has(key)){
      unique.push(c);
      seen.add(key);
    }
    if(unique.length>=52) break;
  }
  // if still less than 52, pad by repeating center row cells
  while(unique.length < 52){
    unique.push([6 + (unique.length%3), 6 + Math.floor((unique.length%9)/3)]);
  }

  // convert to pixel centers
  for(const cell of unique){
    const cx = MARGIN + cell[0]*TILE + TILE/2;
    const cy = MARGIN + cell[1]*TILE + TILE/2;
    coords.push({x:cx,y:cy});
  }
  return coords;
}

const TRACK = buildTrackCoords();

function drawBoard(){
  // clear
  ctx.clearRect(0,0,W,H);
  // background
  ctx.fillStyle = '#fff';
  roundRect(ctx,8,8,W-16,H-16,12,true,false);

  // draw track tiles
  for(let i=0;i<TRACK.length;i++){
    const t = TRACK[i];
    // tile background
    ctx.fillStyle = (SAFE_INDICES.has(i) ? '#fffbe6' : '#f2f7ff');
    ctx.beginPath();
    ctx.arc(t.x,t.y,TILE*0.42,0,Math.PI*2);
    ctx.fill();

    // index small
    ctx.fillStyle = '#abc';
    ctx.font = '10px Poppins';
    ctx.textAlign = 'center';
    ctx.fillText(i, t.x, t.y+4);
  }

  // draw center home area (decorative)
  ctx.fillStyle = '#f8fbff';
  roundRect(ctx, W/2 - 60, H/2 - 60, 120,120,10,true,false);

  // draw tokens
  if(!game) return;
  game.players.forEach(p=>{
    p.tokens.forEach(tkn=>{
      if(tkn.steps >=0 && tkn.steps < 52){
        const idx = boardIndex(p.id, tkn.steps);
        const pos = TRACK[idx];
        drawToken(pos.x, pos.y, p.color, `${p.id}-${tkn.id}`);
      } else if(tkn.steps === -1){
        // draw at player's home corner
        const home = playerHomeCoord(p.id, tkn.id);
        drawToken(home.x, home.y, p.color, `${p.id}-${tkn.id}`, true);
      } else {
        // finished -> draw stacked near score list area area (off-board)
        const fin = finishedCoord(p.id, tkn.id);
        drawToken(fin.x, fin.y, p.color, `${p.id}-${tkn.id}`, true);
      }
    });
  });
}

function drawToken(x,y,color,label,small=false){
  ctx.beginPath();
  ctx.fillStyle = color;
  ctx.shadowColor = 'rgba(0,0,0,0.15)';
  ctx.shadowBlur = 8;
  ctx.arc(x,y, small ? 12 : 16,0,Math.PI*2);
  ctx.fill();
  ctx.shadowBlur = 0;
  // border
  ctx.lineWidth = 2;
  ctx.strokeStyle = '#fff';
  ctx.stroke();

  // label small
  ctx.fillStyle = '#fff';
  ctx.font = small ? '10px Poppins' : '12px Poppins';
  ctx.textAlign = 'center';
  ctx.fillText(label.split('-')[1], x, y+3);
}

function playerHomeCoord(playerId, tokenId){
  // place 4 tokens per player in a corner home region
  const pad = 30;
  if(playerId === 0){ // top center-ish
    return { x: W/2 - TILE*3 + (tokenId%2)*30, y: 20 + Math.floor(tokenId/2)*30 };
  } else if(playerId === 1){ // right top
    return { x: W - 60 - Math.floor(tokenId/2)*30, y: H/2 - TILE*3 + (tokenId%2)*30 };
  } else if(playerId === 2){ // bottom center
    return { x: W/2 - TILE*3 + (tokenId%2)*30, y: H - 40 - Math.floor(tokenId/2)*30 };
  } else { // left
    return { x: 20 + Math.floor(tokenId/2)*30, y: H/2 - TILE*3 + (tokenId%2)*30 };
  }
}

function finishedCoord(playerId, tokenId){
  // draw finished tokens near right side stacked
  const baseX = W - 28;
  const baseY = 28 + playerId*80 + tokenId*18;
  return { x: baseX, y: baseY };
}

function roundRect(ctx,x,y,w,h,r,fill,stroke){
  if (typeof stroke === 'undefined') stroke = true;
  if (typeof r === 'undefined') r = 5;
  ctx.beginPath();
  ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  ctx.closePath();
  if(fill) ctx.fill();
  if(stroke) ctx.stroke();
}

/* ------------------ UI hooks ------------------ */

startBtn.addEventListener('click', ()=>{
  const n = parseInt(playerCountSelect.value,10);
  newGame(n);
});

rollBtn.addEventListener('click', ()=>{
  if(!game) return alert('Start a game first!');
  rollDice();
});

playerCountSelect.addEventListener('change', ()=>{
  // no-op until Start clicked
});

// initialize default
newGame(4);

/* Helpful: allow clicking on canvas token to select the nearest token and attempt to move it (if dice rolled) */
canvas.addEventListener('click', (e)=>{
  if(!game || game.dice === null) return;
  const rect = canvas.getBoundingClientRect();
  const mx = e.clientX - rect.left;
  const my = e.clientY - rect.top;

  // find nearest token of current player within threshold
  const player = game.players[game.cur];
  let found = null;
  for(const t of player.tokens){
    let pos = null;
    if(t.steps >=0 && t.steps < 52){
      pos = TRACK[boardIndex(player.id, t.steps)];
    } else if(t.steps === -1){
      pos = playerHomeCoord(player.id, t.id);
    } else {
      pos = finishedCoord(player.id, t.id);
    }
    const d = Math.hypot(mx-pos.x, my-pos.y);
    if(d < 20){
      found = t.id;
      break;
    }
  }
  if(found !== null){
    handleMove(player.id, found);
  }
});

/* show initial hint */
diceDisplay.textContent = 'Roll';
diceValBadge.textContent = '—';
updateUI();

</script>
</body>
</html>
