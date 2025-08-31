# Shashka-offline-
<!DOCTYPE html>
<html lang="uz">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1.0"/>
  <title>Klassik Xalqaro Shashka</title>
  <style>
    :root{
      --gold:#facc15;
      --sky:#38bdf8;
      --dark:#0f172a;
      --light:#f1f5f9;
      --brown:#8b5e3c;   /* jigar rang (qorong'i katak) */
    }
    *{box-sizing:border-box}
    body{
      display:flex;flex-direction:column;align-items:center;
      background:linear-gradient(135deg,#1e293b,#0f172a);
      color:#fff;font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif;
      margin:0;padding:12px;min-height:100vh;
      -webkit-tap-highlight-color: transparent;
      user-select:none;
    }
    h1{
      margin:10px;font-size:1.8rem;color:var(--gold);
      text-shadow:1px 1px 4px #000;
    }
    .controls{
      margin:8px;display:flex;gap:8px;flex-wrap:wrap;justify-content:center;align-items:center
    }
    select,button{
      padding:8px 12px;border-radius:10px;border:none;font-size:14px;cursor:pointer;transition:.15s;
    }
    button{background:var(--gold);color:#000;font-weight:700}
    button:hover{filter:brightness(1.05)}
    #status{margin:6px;font-size:1.05rem;color:var(--sky)}
    #winner{
      margin-top:8px;font-size:1.2rem;font-weight:700;color:var(--gold);
      text-shadow:1px 1px 6px #000;animation:pulse 1.6s infinite
    }
    @keyframes pulse{0%{transform:scale(1)}50%{transform:scale(1.06)}100%{transform:scale(1)}}

    /* Taxta */
    #board{
      display:grid;grid-template-columns:repeat(10,1fr);
      width:min(95vmin,520px);aspect-ratio:1;border:4px solid var(--gold);
      margin:12px 0;touch-action:manipulation;border-radius:12px;overflow:hidden;
      box-shadow:0 8px 22px rgba(0,0,0,.35);
      background:#0001;
    }
    .cell{display:flex;align-items:center;justify-content:center;position:relative}
    .light{background:var(--light)}
    .dark{background:var(--brown)}

    /* Qo’nish nuqtalari ko‘rsatkichi */
    .cell.move-target::after{
      content:"";position:absolute;inset:10%;border:3px dashed rgba(255,255,0,.95);border-radius:10px;
      animation:blink .8s infinite alternate;
    }
    @keyframes blink{from{opacity:.55}to{opacity:1}}

    /* Donalar */
    .piece{
      width:70%;height:70%;border-radius:50%;
      box-shadow:0 3px 8px rgba(0,0,0,.55);
      cursor:pointer;position:relative;transition:transform .1s ease, box-shadow .1s ease;
    }
    .piece.white{background:#fff}
    .piece.black{background:#111}
    .piece:active{transform:scale(.92)}
    .highlight{
      box-shadow:0 0 14px 3px var(--sky), 0 0 20px 6px var(--sky) inset !important;
      outline:3px solid rgba(56,189,248,.95);border-radius:50%;
    }
    /* Damka – oltin halqa */
    .piece.king::after{
      content:"";position:absolute;left:50%;top:50%;
      width:76%;height:76%;border:3px solid gold;border-radius:50%;
      transform:translate(-50%,-50%);box-shadow:0 0 10px rgba(255,215,0,.55);
    }

    /* Uchar (ghost) animatsiya – juda tez */
    .ghost{
      position:fixed;pointer-events:none;z-index:10;width:48px;height:48px;border-radius:50%;
      box-shadow:0 3px 8px rgba(0,0,0,.45);transition:transform .12s ease;
    }

    @media (max-width:420px){
      #board{width:94vw}
      select,button{font-size:13px}
    }
  </style>
</head>
<body>
  <h1>🌟 Klassik Xalqaro Shashka</h1>

  <div class="controls">
    <select id="mode">
      <option value="2p">👥 2 o‘yinchi</option>
      <option value="ai">🤖 Kompyuterga qarshi</option>
    </select>
    <select id="difficulty">
      <option value="easy">Oson</option>
      <option value="medium">O‘rtacha</option>
      <option value="hard">Qiyin</option>
    </select>
    <select id="playerColor">
      <option value="white">Oq bilan o‘ynash</option>
      <option value="black">Qora bilan o‘ynash</option>
    </select>
    <button id="newgame">♻️ Yangi o‘yin</button>
  </div>

  <div id="status"></div>
  <div id="board"></div>
  <div id="winner"></div>

  <script>
    /************ Asosiy holat ************/
    const SIZE = 10;
    const boardEl = document.getElementById('board');
    const statusEl = document.getElementById('status');
    const winnerEl = document.getElementById('winner');
    const modeEl = document.getElementById('mode');
    const diffEl = document.getElementById('difficulty');
    const colorEl = document.getElementById('playerColor');
    const newBtn = document.getElementById('newgame');

    let board = [];
    let currentPlayer = 'white';
    let gameOver = false;
    let gameMode = '2p';
    let difficulty = 'easy';
    let playerColor = 'white';

    let selected = null;                 // {r,c}
    let legalTargets = [];               // [{r,c,captures,path,becameKing}]
    let mustCaptureGlobally = false;
    let globalMaxCapture = 0;            // maksimal urish uzunligi (xalqaro qoida)
    let forceChain = false;              // ketma-ket urishda aynan shu dona bilan davom ettirish majburiy

    const inBounds = (r,c)=> r>=0 && r<SIZE && c>=0 && c<SIZE;
    const cloneBoard = src => src.map(row => row.map(cell => cell ? {...cell} : null));
    const opposite = col => col==='white' ? 'black' : 'white';

    const ALL_DIRS = [[1,1],[1,-1],[-1,1],[-1,-1]];
    const MAN_DIRS_MOVE = { white:[[-1,-1],[-1,1]], black:[[1,-1],[1,1]] };

    function createPiece(color, king=false){ return {color, king}; }
    function getCell(r,c){ return boardEl.querySelector(`.cell[data-row="${r}"][data-col="${c}"]`); }

    /************ O‘yin start ************/
    newBtn.addEventListener('click', startGame);

    function startGame(){
      board = [];
      boardEl.innerHTML = '';
      winnerEl.textContent = '';
      statusEl.textContent = '';
      selected = null;
      legalTargets = [];
      forceChain = false;
      gameOver = false;

      gameMode = modeEl.value;
      difficulty = diffEl.value;
      playerColor = colorEl.value;
      currentPlayer = 'white'; // oq boshlaydi

      // Doskani yaratish
      for(let r=0;r<SIZE;r++){
        board[r]=[];
        for(let c=0;c<SIZE;c++){
          const cell = document.createElement('div');
          cell.className = 'cell ' + (((r+c)&1) ? 'dark' : 'light');
          cell.dataset.row = r; cell.dataset.col = c;
          boardEl.appendChild(cell);

          let piece = null;
          if(((r+c)&1)===1 && r<4) piece = createPiece('black', false);
          if(((r+c)&1)===1 && r>=SIZE-4) piece = createPiece('white', false);
          board[r][c] = piece;
        }
      }
      draw();
      updateStatus();

      if(gameMode==='ai' && playerColor!==currentPlayer) aiTurn();
    }

    /************ Chizish ************/
    function draw(){
      // default cell click
      [...boardEl.children].forEach(cell=>{
        cell.innerHTML='';
        cell.classList.remove('move-target');
        cell.onclick = ()=> onCellClick(+cell.dataset.row, +cell.dataset.col);
      });

      // global majburiy urish va maksimal uzunlikni hisoblash
      mustCaptureGlobally = hasAnyCapture(currentPlayer, board);
      globalMaxCapture = mustCaptureGlobally ? getGlobalMaxCapture(board, currentPlayer) : 0;

      // donalarni chizish
      for(let r=0;r<SIZE;r++){
        for(let c=0;c<SIZE;c++){
          const p = board[r][c];
          if(!p) continue;
          const cell = getCell(r,c);
          const el = document.createElement('div');
          el.className = 'piece ' + p.color + (p.king?' king':'');
          el.onclick = (e)=>{ e.stopPropagation(); onPieceClick(r,c); };
          // majburiy urishda faqat maksimal urish bera oladigan donalar yaltiraydi
          if(mustCaptureGlobally && !forceChain){
            const seqs = captureSequences(r,c,board);
            if(seqs.some(s => s.captures.length === globalMaxCapture)) el.classList.add('highlight');
          }
          // ketma-ket urishda faqat o‘sha dona yaltiraydi
          if(forceChain && selected && selected.r===r && selected.c===c){
            el.classList.add('highlight');
          }
          cell.appendChild(el);
          cell.onclick = null; // band katakka qo‘nishni bloklash
        }
      }

      // tanlangan donaning qo‘nish nuqtalarini ko‘rsatish
      legalTargets.forEach(t => getCell(t.r,t.c).classList.add('move-target'));
    }

    function updateStatus(){
      statusEl.textContent = currentPlayer==='white' ? '♟️ Oq navbatda' : '♞ Qora navbatda';
    }

    /************ Eventlar ************/
    function onPieceClick(r,c){
      if(gameOver) return;
      const p = board[r][c];
      if(!p || p.color!==currentPlayer) return;

      // Agar zanjir davom etayotgan bo‘lsa — faqat tanlangan dona bilan davom etish mumkin
      if(forceChain && selected && (r!==selected.r || c!==selected.c)) return;

      if(mustCaptureGlobally && !forceChain){
        const seqs = captureSequences(r,c,board);
        if(seqs.length===0) return; // bu dona ura olmaydi
        const top = seqs.filter(s => s.captures.length === globalMaxCapture);
        if(top.length===0) return; // maksimalga teng bo‘lmasa, tanlanmaydi
        selected = {r,c};
        legalTargets = top.map(s=>({r:s.landing.r,c:s.landing.c,captures:s.captures,path:s.path,becameKing:s.becameKing}));
      }else{
        selected = {r,c};
        legalTargets = computeMovesForPiece(r,c,board,false);
      }
      draw();
    }

    function onCellClick(r,c){
      if(gameOver || !selected) return;
      if(board[r][c]) return; // band
      const target = legalTargets.find(t => t.r===r && t.c===c);
      if(!target) return;
      animateAndApply(selected.r, selected.c, target);
    }

    /************ Qoidalar ************/
    function hasAnyCapture(color, brd){
      for(let r=0;r<SIZE;r++){
        for(let c=0;c<SIZE;c++){
          const p = brd[r][c];
          if(!p || p.color!==color) continue;
          if(captureSequences(r,c,brd).length>0) return true;
        }
      }
      return false;
    }

    function getGlobalMaxCapture(brd, color){
      let mx = 0;
      for(let r=0;r<SIZE;r++){
        for(let c=0;c<SIZE;c++){
          const p = brd[r][c];
          if(!p || p.color!==color) continue;
          const seqs = captureSequences(r,c,brd);
          for(const s of seqs) if(s.captures.length>mx) mx = s.captures.length;
        }
      }
      return mx;
    }

    function computeMovesForPiece(r,c,brd,onlyCaptures=false){
      const p = brd[r][c]; if(!p) return [];
      const seqs = captureSequences(r,c,brd);
      if(seqs.length>0){
        const mx = mustCaptureGlobally ? globalMaxCapture : Math.max(...seqs.map(s=>s.captures.length));
        return seqs.filter(s=> s.captures.length===mx)
                   .map(s=>({r:s.landing.r,c:s.landing.c,captures:s.captures,path:s.path,becameKing:s.becameKing}));
      }
      if(onlyCaptures) return [];

      // Oddiy yurishlar
      if(p.king){
        const moves=[];
        for(const [dr,dc] of ALL_DIRS){
          let nr=r+dr, nc=c+dc;
          while(inBounds(nr,nc) && !brd[nr][nc]){
            moves.push({r:nr,c:nc,captures:[],path:[{r:nr,c:nc}],becameKing:false});
            nr+=dr; nc+=dc;
          }
        }
        return moves;
      }else{
        const moves=[];
        for(const [dr,dc] of MAN_DIRS_MOVE[p.color]){
          const nr=r+dr, nc=c+dc;
          if(inBounds(nr,nc) && !brd[nr][nc]){
            const become = (p.color==='white' && nr===0) || (p.color==='black' && nr===SIZE-1);
            moves.push({r:nr,c:nc,captures:[],path:[{r:nr,c:nc}],becameKing:become});
          }
        }
        return moves;
      }
    }

    // To‘liq ketma-ket urishlarni topish (DFS)
    function captureSequences(r,c,brd){
      const me = brd[r][c]; if(!me) return [];
      const out = [];

      function dfs(sr,sc,boardNow,captured,path,becameKingFlag){
        const piece = boardNow[sr][sc];
        // Man damkaga aylansa — shu yerda tugaydi (xalqaro qoida)
        if(!piece.king && ((piece.color==='white' && sr===0) || (piece.color==='black' && sr===SIZE-1))){
          out.push({landing:{r:sr,c:sc},captures:[...captured],path:[...path],becameKing:true});
          return;
        }

        let any = false;

        if(piece.king){
          for(const [dr,dc] of ALL_DIRS){
            let nr=sr+dr, nc=sc+dc, enemy=null;
            while(inBounds(nr,nc)){
              if(boardNow[nr][nc]){
                if(boardNow[nr][nc].color===piece.color) break;
                if(enemy) break; // ikkinchi dushman – blok
                enemy = {r:nr,c:nc};
              }else{
                if(enemy){
                  const nb = cloneBoard(boardNow);
                  nb[sr][sc]=null;
                  nb[enemy.r][enemy.c]=null;
                  nb[nr][nc]={...piece}; // king o‘zi
                  dfs(nr,nc,nb,[...captured,enemy],[...path,{r:nr,c:nc}],becameKingFlag);
                  any = true;
                }
              }
              nr+=dr; nc+=dc;
            }
          }
        }else{
          // Man – urishda orqaga ham sakray oladi
          for(const [dr,dc] of ALL_DIRS){
            const nr=sr+dr, nc=sc+dc, jr=sr+2*dr, jc=sc+2*dc;
            if(inBounds(jr,jc) && boardNow[nr][nc] && boardNow[nr][nc].color!==piece.color && !boardNow[jr][jc]){
              const nb = cloneBoard(boardNow);
              nb[sr][sc]=null;
              const become = (piece.color==='white' && jr===0) || (piece.color==='black' && jr===SIZE-1);
              nb[nr][nc]=null;
              nb[jr][jc]={...piece, king: become ? true : piece.king};
              if(become){
                out.push({landing:{r:jr,c:jc},captures:[...captured,{r:nr,c:nc}],path:[...path,{r:jr,c:jc}],becameKing:true});
                any = true;
              }else{
                dfs(jr,jc,nb,[...captured,{r:nr,c:nc}],[...path,{r:jr,c:jc}],becameKingFlag||false);
                any = true;
              }
            }
          }
        }

        if(!any){
          out.push({landing:{r:sr,c:sc},captures:[...captured],path:[...path],becameKing:becameKingFlag||false});
        }
      }

      dfs(r,c,brd,[],[],false);
      return out.filter(s => s.captures.length>0);
    }

    /************ Animatsiya va yurishni qo‘llash ************/
    function animateAndApply(sr,sc,target){
      const srcCell = getCell(sr,sc);
      const dstCell = getCell(target.r,target.c);
      const pieceEl = srcCell?.firstChild;

      if(!pieceEl){ applyMove(sr,sc,target); return; }

      const sb = srcCell.getBoundingClientRect();
      const db = dstCell.getBoundingClientRect();
      const ghost = pieceEl.cloneNode(true);
      ghost.classList.add('ghost');
      const size = Math.min(sb.width*0.7, 56);
      ghost.style.width = size+'px';
      ghost.style.height = size+'px';
      ghost.style.left = (sb.left + (sb.width-size)/2) + 'px';
      ghost.style.top  = (sb.top  + (sb.height-size)/2) + 'px';
      document.body.appendChild(ghost);

      requestAnimationFrame(()=>{
        const dx = (db.left + (db.width-size)/2) - parseFloat(ghost.style.left);
        const dy = (db.top  + (db.height-size)/2) - parseFloat(ghost.style.top);
        ghost.style.transform = `translate(${dx}px, ${dy}px)`;
      });

      setTimeout(()=>{ ghost.remove(); applyMove(sr,sc,target); }, 130); // juda tez
    }

    function applyMove(sr,sc,target){
      const p = board[sr][sc];
      const {r:tr, c:tc, captures, becameKing} = target;

      // maydondan ko'chirish
      board[sr][sc] = null;

      // urilganlarni olib tashlash
      if(captures && captures.length){
        captures.forEach(cp => { board[cp.r][cp.c] = null; });
      }

      // qo‘nish + damka
      const makeKing = p.king ? true : (becameKing || (p.color==='white' && tr===0) || (p.color==='black' && tr===SIZE-1));
      board[tr][tc] = {color:p.color, king: makeKing};

      // zanjir davom ettirish (faqat agar man damkaga aylanmagan bo‘lsa)
      if(captures && captures.length && !(!p.king && makeKing)){
        const more = captureSequences(tr,tc,board);
        if(more.length>0){
          const mx = Math.max(...more.map(s=>s.captures.length));
          const options = more.filter(s=> s.captures.length===mx)
                              .map(s=>({r:s.landing.r,c:s.landing.c,captures:s.captures,path:s.path,becameKing:s.becameKing}));

          selected = {r:tr,c:tc};
          legalTargets = options;
          forceChain = true;         // <<< faqat shu dona bilan davom etish shart
          draw();

          // AI bo‘lsa — avtomatik davom ettiramiz
          if(gameMode==='ai' && currentPlayer!==playerColor){
            const pick = options[Math.random()*options.length|0];
            animateAndApply(tr,tc,pick);
          }
          return;
        }
      }

      // navbatni almashtirish
      selected = null;
      legalTargets = [];
      forceChain = false;
      currentPlayer = opposite(currentPlayer);
      draw();
      updateStatus();
      checkWinner();

      if(!gameOver && gameMode==='ai' && playerColor!==currentPlayer){
        aiTurn();
      }
    }

    /************ G‘olibni tekshirish (aniq) ************/
    function checkWinner(){
      const whiteInfo = allMovesFor('white', board);
      const blackInfo = allMovesFor('black', board);

      const whiteCount = countPieces('white');
      const blackCount = countPieces('black');

      if(whiteCount===0 || whiteInfo.moves.length===0){
        gameOver=true; winnerEl.textContent='🏆 Qora tosh egasi — Sizni tabriklaymiz, g‘olib bo‘ldingiz!';
      }else if(blackCount===0 || blackInfo.moves.length===0){
        gameOver=true; winnerEl.textContent='🏆 Oq tosh egasi — Sizni tabriklaymiz, g‘olib bo‘ldingiz!';
      }
    }
    function countPieces(color){
      let n=0;
      for(let r=0;r<SIZE;r++) for(let c=0;c<SIZE;c++) if(board[r][c]?.color===color) n++;
      return n;
    }

    /************ AI ************/
    function allMovesFor(color, brd){
      const caps=[]; const quiet=[];
      for(let r=0;r<SIZE;r++){
        for(let c=0;c<SIZE;c++){
          const p=brd[r][c]; if(!p || p.color!==color) continue;
          const seqs = captureSequences(r,c,brd);
          if(seqs.length){
            const mx = Math.max(...seqs.map(s=>s.captures.length));
            seqs.filter(s=> s.captures.length===mx)
                .forEach(s=> caps.push({sr:r,sc:c, r:s.landing.r, c:s.landing.c, captures:s.captures, becameKing:s.becameKing}));
          }else{
            // oddiy yurishlar (global flaglarsiz)
            if(p.king){
              for(const [dr,dc] of ALL_DIRS){
                let nr=r+dr, nc=c+dc;
                while(inBounds(nr,nc) && !brd[nr][nc]){
                  quiet.push({sr:r,sc:c,r:nr,c:nc,captures:[],becameKing:false});
                  nr+=dr; nc+=dc;
                }
              }
            }else{
              for(const [dr,dc] of MAN_DIRS_MOVE[p.color]){
                const nr=r+dr, nc=c+dc;
                if(inBounds(nr,nc) && !brd[nr][nc]){
                  const become = (p.color==='white' && nr===0) || (p.color==='black' && nr===SIZE-1);
                  quiet.push({sr:r,sc:c,r:nr,c:nc,captures:[],becameKing:become});
                }
              }
            }
          }
        }
      }
      if(caps.length){
        const mx = Math.max(...caps.map(m=> m.captures.length));
        return {moves: caps.filter(m=> m.captures.length===mx), must:true};
      }
      return {moves:quiet, must:false};
    }

    function simulateMove(brd, mv){
      const p = brd[mv.sr][mv.sc];
      brd[mv.sr][mv.sc]=null;
      if(mv.captures) mv.captures.forEach(cp => brd[cp.r][cp.c]=null);
      const toRow = mv.r;
      const makeKing = p.king ? true : (mv.becameKing || (p.color==='white' && toRow===0) || (p.color==='black' && toRow===SIZE-1));
      brd[mv.r][mv.c] = {color:p.color, king: makeKing};
    }

    function evaluate(brd, asWhite=true){
      let score=0;
      for(let r=0;r<SIZE;r++){
        for(let c=0;c<SIZE;c++){
          const p=brd[r][c]; if(!p) continue;
          let val = p.king ? 5 : 3;
          val += (p.color==='white' ? (9-r)*0.02 : r*0.02); // ilgari yurish rag'bati
          // mobilitet (global flaglarsiz, soddalashtirilgan)
          const seqs = captureSequences(r,c,brd);
          let mob = 0;
          if(seqs.length){ mob += Math.max(...seqs.map(s=>s.captures.length))*0.05; }
          else{
            if(p.king){
              for(const [dr,dc] of ALL_DIRS){
                let nr=r+dr,nc=c+dc; while(inBounds(nr,nc) && !brd[nr][nc]){ mob+=0.02; nr+=dr; nc+=dc; }
              }
            }else{
              for(const [dr,dc] of MAN_DIRS_MOVE[p.color]){
                const nr=r+dr,nc=c+dc; if(inBounds(nr,nc)&&!brd[nr][nc]) mob+=0.05;
              }
            }
          }
          val += mob;
          score += (p.color==='white') ? val : -val;
        }
      }
      return asWhite ? score : -score;
    }

    // AI rejimlari
    function aiRandomMove(color){
      const {moves} = allMovesFor(color, board);
      if(moves.length===0) return null;
      return moves[Math.random()*moves.length|0];
    }

    function aiGreedyMove(color){
      const {moves} = allMovesFor(color, board);
      if(moves.length===0) return null;
      let best=[], bestScore=-1e9;
      for(const m of moves){
        const b2 = cloneBoard(board);
        simulateMove(b2,m);
        let sc = evaluate(b2, color==='white');
        sc += (m.captures?m.captures.length:0)*1.2 + (m.becameKing?1.4:0);
        if(sc>bestScore){bestScore=sc; best=[m]} else if(sc===bestScore) best.push(m);
      }
      return best[Math.random()*best.length|0];
    }

    function aiMinimaxMove(color, depth){
      const isMax = (color==='white');
      const {moves} = allMovesFor(color, board);
      if(moves.length===0) return null;
      let best=null, bestVal=isMax?-1e9:1e9;
      for(const mv of moves){
        const b2 = cloneBoard(board);
        simulateMove(b2,mv);
        const val = minimax(b2, depth-1, opposite(color), -1e9, 1e9);
        if(isMax ? val>bestVal : val<bestVal){ bestVal=val; best=mv; }
      }
      return best;
    }

    function minimax(brd, depth, turn, alpha, beta){
      if(depth===0) return evaluate(brd, true);
      const isMax = (turn==='white');
      const {moves} = allMovesFor(turn, brd);
      if(moves.length===0) return isMax ? -1e6 : 1e6;

      if(isMax){
        let value=-1e9;
        for(const mv of moves){
          const b2 = cloneBoard(brd);
          simulateMove(b2,mv);
          value = Math.max(value, minimax(b2, depth-1, opposite(turn), alpha, beta));
          alpha = Math.max(alpha, value);
          if(alpha>=beta) break;
        }
        return value;
      }else{
        let value=1e9;
        for(const mv of moves){
          const b2 = cloneBoard(brd);
          simulateMove(b2,mv);
          value = Math.min(value, minimax(b2, depth-1, opposite(turn), alpha, beta));
          beta = Math.min(beta, value);
          if(alpha>=beta) break;
        }
        return value;
      }
    }

    function aiTurn(){
      setTimeout(()=>{
        if(gameOver) return;
        const color = currentPlayer; // doim navbatdagini o‘ynaydi
        let move=null;
        if(difficulty==='hard') move = aiMinimaxMove(color, 3);
        else if(difficulty==='medium') move = aiGreedyMove(color);
        else move = aiRandomMove(color);

        if(!move){
          gameOver=true;
          winnerEl.textContent = color==='white'
            ? '🏆 Qora tosh egasi — Sizni tabriklaymiz, g‘olib bo‘ldingiz!'
            : '🏆 Oq tosh egasi — Sizni tabriklaymiz, g‘olib bo‘ldingiz!';
          return;
        }
        animateAndApply(move.sr, move.sc, {r:move.r,c:move.c,captures:move.captures||[],becameKing:move.becameKing||false});
      }, 120);
    }

    /************ Boshlash ************/
    startGame();
  </script>
</body>
</html>
