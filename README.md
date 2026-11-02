<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Stockfish 16 Pro Chess App</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/chess.js/0.10.3/chess.min.js"></script>
    <style>
        .cell {
            position: relative;
            display: flex;
            align-items: center;
            justify-content: center;
            user-select: none;
            cursor: pointer;
            overflow: hidden;
        }
        .coord {
            position: absolute;
            font-size: 10px;
            font-weight: bold;
            pointer-events: none;
        }
        .coord-rank { top: 2px; left: 3px; }
        .coord-file { bottom: 2px; right: 4px; }
        .light-coord { color: #b58863; }
        .dark-coord { color: #f0d9b5; }

        .piece-img {
            width: 85%;
            height: 85%;
            object-fit: contain;
            filter: drop-shadow(0px 2px 3px rgba(0,0,0,0.4));
            transition: transform 0.18s cubic-bezier(0.4, 0, 0.2, 1), opacity 0.15s ease;
        }
        .cell:hover .piece-img {
            transform: scale(1.08);
        }
        .piece-selected {
            transform: scale(1.18) translateY(-3px) !important;
            filter: drop-shadow(0px 6px 8px rgba(0,0,0,0.6));
        }
        .premove-highlight {
            background-color: rgba(239, 68, 68, 0.45) !important;
        }
        .hint-from-highlight {
            background-color: rgba(59, 130, 246, 0.6) !important;
        }
        .hint-to-highlight {
            background-color: rgba(20, 184, 166, 0.7) !important;
        }
    </style>
</head>
<body class="bg-slate-950 text-white min-h-screen p-4 flex flex-col items-center font-sans select-none">
    <div class="text-xl font-bold mb-1 text-teal-400">Stockfish 16 Pro Chess</div>
    <div class="text-xs text-slate-400 mb-3">모바일 최적화 웹 체스 앱</div>

    <!-- 설정 컨트롤 바 -->
    <div class="w-full max-w-[420px] bg-slate-900 p-3 rounded-t-lg border border-slate-800 border-b-0 space-y-2">
        <div class="flex items-center justify-between space-x-2">
            <!-- 모드 선택 -->
            <div class="flex items-center space-x-1">
                <label for="mode-select" class="text-xs font-bold text-slate-300">🎮 모드:</label>
                <select id="mode-select" onchange="changeMode(this.value)" class="bg-slate-800 text-teal-300 text-xs font-bold px-2 py-1 rounded border border-slate-700 outline-none cursor-pointer">
                    <option value="ai" selected>🤖 AI 대국</option>
                    <option value="pvp">👥 2인용 대국</option>
                </select>
            </div>
            <!-- AI 난이도 선택 -->
            <div id="difficulty-box" class="flex items-center space-x-1">
                <label for="difficulty-select" class="text-xs font-bold text-slate-300">🤖 난이도:</label>
                <select id="difficulty-select" onchange="changeDifficulty(this.value)" class="bg-slate-800 text-teal-300 text-xs font-bold px-2 py-1 rounded border border-slate-700 outline-none cursor-pointer">
                    <option value="2">🟢 완전 초보</option>
                    <option value="8" selected>🟡 아마추어</option>
                    <option value="15">🟠 상급 마스터</option>
                    <option value="20">🔴 Stockfish Max</option>
                </select>
            </div>
        </div>

        <div class="flex items-center justify-between pt-1 border-t border-slate-800/60">
            <!-- 진영 선택 -->
            <div id="color-box" class="flex items-center space-x-1">
                <label for="color-select" class="text-xs font-bold text-slate-300">👑 진영:</label>
                <select id="color-select" onchange="changeColor(this.value)" class="bg-slate-800 text-amber-300 text-xs font-bold px-2 py-1 rounded border border-slate-700 outline-none cursor-pointer">
                    <option value="white" selected>⚪ 백색 (선공)</option>
                    <option value="black">⚫ 흑색 (후공)</option>
                    <option value="random">🎲 랜덤</option>
                </select>
            </div>
            <span class="text-[11px] text-slate-500">진영 변경 시 새 게임</span>
        </div>
    </div>

    <!-- 평가치 및 상태 바 -->
    <div class="w-full max-w-[420px] flex items-center justify-between bg-slate-900 px-3 py-2 border border-slate-800 mb-2 rounded-b-lg">
        <div id="status" class="text-sm font-bold text-emerald-400">당신의 턴 (백색)</div>
        <div class="flex items-center space-x-2">
            <span class="text-xs text-slate-400 font-semibold">평가치:</span>
            <span id="eval-score" class="text-sm font-extrabold text-teal-300 bg-slate-800 px-2 py-0.5 rounded border border-teal-500/30">+0.00</span>
        </div>
    </div>

    <!-- 체스판 -->
    <div id="board" class="w-full max-w-[420px] aspect-square grid grid-cols-8 grid-rows-8 border-4 border-slate-700 rounded-lg overflow-hidden shadow-2xl"></div>

    <!-- 컨트롤 버튼 -->
    <div class="mt-3 w-full max-w-[420px] grid grid-cols-4 gap-2">
        <button onclick="getHint()" class="py-2 bg-amber-600 hover:bg-amber-500 text-white rounded-lg text-xs font-bold transition flex items-center justify-center space-x-1">
            <span>💡 AI 힌트</span>
        </button>
        <button onclick="resetGame()" class="py-2 bg-indigo-600 hover:bg-indigo-500 text-white rounded-lg text-xs font-bold transition">새 게임</button>
        <button onclick="undoMove()" class="py-2 bg-slate-800 hover:bg-slate-700 text-white rounded-lg text-xs font-bold transition">무르기</button>
        <button onclick="resignGame()" class="py-2 bg-rose-700 hover:bg-rose-600 text-white rounded-lg text-xs font-bold transition">기권</button>
    </div>

    <!-- 기보 기록 출력 창 -->
    <div class="mt-3 w-full max-w-[420px] bg-slate-900 border border-slate-800 rounded-lg p-3">
        <div class="text-xs font-bold text-slate-400 mb-1 flex justify-between">
            <span>📜 착수 기보 (Move Log)</span>
            <span id="premove-status" class="text-rose-400 hidden">🔴 미리두기 예약됨</span>
        </div>
        <div id="pgn-log" class="h-16 overflow-y-auto text-xs font-mono text-slate-300 bg-slate-950 p-2 rounded border border-slate-800/80 leading-relaxed whitespace-pre-wrap">게임 시작</div>
    </div>

    <script>
        const CDN_BASE = 'https://raw.githubusercontent.com/lichess-org/lila/master/public/piece/cburnett/';
        const PIECE_URLS = {
            'P': CDN_BASE + 'wP.svg', 'R': CDN_BASE + 'wR.svg', 'N': CDN_BASE + 'wN.svg',
            'B': CDN_BASE + 'wB.svg', 'Q': CDN_BASE + 'wQ.svg', 'K': CDN_BASE + 'wK.svg',
            'p': CDN_BASE + 'bP.svg', 'r': CDN_BASE + 'bR.svg', 'n': CDN_BASE + 'bN.svg',
            'b': CDN_BASE + 'bB.svg', 'q': CDN_BASE + 'bQ.svg', 'k': CDN_BASE + 'bK.svg'
        };

        let game = new Chess();
        let selected = null;
        let validMoves = [];
        let isThinking = false;
        let currentMode = "ai";
        let playerColorStr = "w";
        let premove = null;
        let currentHint = null;

        function changeMode(mode) {
            currentMode = mode;
            const diffBox = document.getElementById('difficulty-box');
            const colorBox = document.getElementById('color-box');
            if(mode === 'pvp') {
                diffBox.classList.add('opacity-40', 'pointer-events-none');
                colorBox.classList.add('opacity-40', 'pointer-events-none');
            } else {
                diffBox.classList.remove('opacity-40', 'pointer-events-none');
                colorBox.classList.remove('opacity-40', 'pointer-events-none');
            }
            resetGame();
        }

        function changeDifficulty(level) {
            // 난이도 변경 로직
        }

        function changeColor(color) {
            playerColorStr = color === 'black' ? 'b' : 'w';
            resetGame();
        }

        function resetGame() {
            game.reset();
            selected = null;
            validMoves = [];
            premove = null;
            currentHint = null;
            isThinking = false;
            document.getElementById('premove-status').classList.add('hidden');
            updateUI();
        }

        function updateUI(customMsg) {
            const statusEl = document.getElementById('status');
            const pgnEl = document.getElementById('pgn-log');
            
            if (customMsg) {
                statusEl.textContent = customMsg;
            } else {
                statusEl.textContent = game.turn() === 'w' ? "백색(White) 턴" : "흑색(Black) 턴";
            }
            
            pgnEl.textContent = game.pgn() || '게임 시작';
            pgnEl.scrollTop = pgnEl.scrollHeight;
            render();
        }

        function render() {
            const boardEl = document.getElementById('board');
            boardEl.innerHTML = '';
            const state = game.board();
            const isFlipped = (currentMode === 'ai' && playerColorStr === 'b');

            for(let i=0; i<8; i++) {
                for(let j=0; j<8; j++) {
                    const r = isFlipped ? 7 - i : i;
                    const c = isFlipped ? 7 - j : j;
                    const sq = String.fromCharCode(97+c)+(8-r);
                    const isLight = (r+c)%2===0;
                    
                    const cell = document.createElement('div');
                    cell.className = `cell ${isLight?'bg-[#f0d9b5]':'bg-[#b58863]'}`;

                    if (j === 0) {
                        const rankLabel = document.createElement('span');
                        rankLabel.className = `coord coord-rank ${isLight?'light-coord':'dark-coord'}`;
                        rankLabel.textContent = 8 - r;
                        cell.appendChild(rankLabel);
                    }
                    if (i === 7) {
                        const fileLabel = document.createElement('span');
                        fileLabel.className = `coord coord-file ${isLight?'light-coord':'dark-coord'}`;
                        fileLabel.textContent = String.fromCharCode(97 + c);
                        cell.appendChild(fileLabel);
                    }

                    if(selected === sq) {
                        const overlay = document.createElement('div');
                        overlay.className = 'absolute inset-0 bg-yellow-400/50 pointer-events-none z-10';
                        cell.appendChild(overlay);
                    }

                    if(validMoves.some(m => m.to === sq)) {
                        const hint = document.createElement('div');
                        hint.className = 'w-4 h-4 bg-emerald-700/60 rounded-full z-20 pointer-events-none';
                        cell.appendChild(hint);
                    }

                    const p = state[r][c];
                    if(p) {
                        const key = p.color === 'w' ? p.type.toUpperCase() : p.type;
                        const img = document.createElement('img');
                        img.src = PIECE_URLS[key];
                        img.className = 'piece-img z-10';
                        if(selected === sq) img.classList.add('piece-selected');
                        cell.appendChild(img);
                    }

                    cell.onclick = () => handleSquareClick(sq, p);
                    boardEl.appendChild(cell);
                }
            }
        }

        function handleSquareClick(sq, p) {
            if(game.game_over()) return;

            if(selected) {
                const move = game.move({from: selected, to: sq, promotion: 'q'});
                if(move) {
                    selected = null;
                    validMoves = [];
                    updateUI();
                    return;
                }
            }

            if(p && p.color === game.turn()) {
                selected = sq;
                validMoves = game.moves({square: sq, verbose: true});
            } else {
                selected = null;
                validMoves = [];
            }
            render();
        }

        function undoMove() {
            game.undo();
            if(currentMode === 'ai') game.undo(); // AI 모드에서는 내 수와 AI 수 동시 취소
            selected = null;
            validMoves = [];
            updateUI();
        }

        function resignGame() {
            updateUI("🏳️ 기권하였습니다.");
        }

        function getHint() {
            alert("💡 클라이언트 모드에서는 기본 가이드라인이 표시됩니다.");
        }

        // 초기 실행
        resetGame();
    </script>
</body>
</html># -
