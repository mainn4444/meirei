<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CPU命令シミュレーター (昼白色テーマ)</title>
    <style>
        /* --- 全体のスタイルと昼白色テーマ --- */
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
            display: flex;
            justify-content: center;
            padding: 20px;
            background-color: #f8f9fa; /* 明るいグレー (昼白色の背景) */
            color: #212529; /* ダークグレーのテキスト */
            gap: 25px;
            flex-wrap: wrap;
        }
        .container {
            background-color: #ffffff; /* コンテンツエリアは白 */
            padding: 25px;
            border-radius: 8px;
            border: 1px solid #dee2e6; /* やさしい境界線 */
            box-shadow: 0 4px 6px rgba(0,0,0,0.05);
            min-width: 320px;
        }
        h2 {
            margin-top: 0;
            border-bottom: 2px solid #0d6efd;
            padding-bottom: 10px;
            color: #343a40;
            font-size: 1.5em;
        }
        h3 {
            margin-top: 20px;
            margin-bottom: 10px;
            font-size: 1.1em;
            color: #495057;
        }

        /* --- 各コンポーネントのスタイル --- */
        #program-code, #memory-view, #output {
            border: 1px solid #ced4da;
            border-radius: 4px;
            padding: 10px;
            height: 220px;
            overflow-y: auto;
            background-color: #f8f9fa;
            font-family: "SF Mono", "Consolas", monospace;
            font-size: 1.1em;
        }
        #program-code div.highlight-pc {
            background-color: #cfe2ff; /* 実行行のハイライト (ソフトな青) */
            font-weight: bold;
        }
        #memory-view span, #cpu-registers span {
            display: inline-block;
            width: 90px;
            border: 1px solid #ced4da;
            padding: 8px 5px;
            margin: 3px;
            text-align: center;
            border-radius: 4px;
            background-color: #fff;
            transition: all 0.3s ease;
        }
        /* メモリ変更時のハイライト */
        #memory-view span.highlight-mem {
            background-color: #fd7e14; /* オレンジ色で変更を強調 */
            color: white;
            transform: scale(1.05);
        }
        button {
            padding: 12px 18px;
            border: none;
            border-radius: 5px;
            background-color: #0d6efd;
            color: white;
            font-size: 16px;
            cursor: pointer;
            transition: background-color 0.2s;
            width: 100%;
        }
        button:hover {
            background-color: #0b5ed7;
        }
        button:disabled {
            background-color: #adb5bd;
            cursor: not-allowed;
        }
        input[type="number"] {
            padding: 10px;
            border-radius: 4px;
            border: 1px solid #ced4da;
            width: calc(100% - 22px); /* paddingを考慮 */
            font-size: 1em;
        }
        .info-box {
            margin-top: 15px;
            padding: 12px;
            background-color: #e9ecef;
            border-left: 5px solid #6c757d;
            font-size: 0.9em;
            color: #495057;
        }
    </style>
</head>
<body>

    <div class="container">
        <h2>📜 プログラムコード</h2>
        <div id="program-code"></div>
        <div class="info-box">
            <p><strong>PC (プログラムカウンタ)</strong>が指す行の命令が次に実行されます。</p>
        </div>
    </div>

    <div class="container">
        <h2>🧠 CPU & メモリ</h2>
        <h3>CPU レジスタ (CPU内の小さな記憶装置)</h3>
        <div id="cpu-registers"></div>
        <h3>📝 メインメモリ (データを保存する机)</h3>
        <div id="memory-view"></div>
    </div>

    <div class="container">
        <h2>🎮 操作パネル</h2>
        <h3>📥 入力 (READ命令用)</h3>
        <input type="number" id="input-value" placeholder="数字を入力してください">
        
        <h3>📤 出力 (WRITE命令の結果)</h3>
        <div id="output"></div>
        
        <div style="margin-top: 25px; display: grid; gap: 10px;">
            <button id="next-btn">次の命令を実行 →</button>
            <button id="reset-btn" style="background-color: #6c757d;">リセット</button>
        </div>
        <div class="info-box">
            <h4>使い方</h4>
            <ol>
                <li><strong>READ命令</strong>の時は、入力欄に数字を入れてから「次へ」を押します。</li>
                <li>命令が1行ずつ実行され、メモリや出力が変化する様子を観察します。</li>
                <li>「リセット」で最初からやり直せます。</li>
            </ol>
        </div>
    </div>

<script>
    // --- プログラム ---
    // 足し算の結果を別の場所に保存するように変更
    const program = [
        "READ 0",   // 入力値をメモリ[0]へ
        "READ 1",   // 入力値をメモリ[1]へ
        "ADD 0 1 2",// メモリ[0]と[1]を足して、結果をメモリ[2]へ
        "WRITE 2",  // メモリ[2]の値を出力
        "STOP"      // 停止
    ];

    // --- グローバル変数 (シミュレーターの状態) ---
    let memory;
    let pc; // プログラムカウンタ

    // --- HTML要素のキャッシュ ---
    const programCodeDiv = document.getElementById('program-code');
    const memoryViewDiv = document.getElementById('memory-view');
    const cpuRegistersDiv = document.getElementById('cpu-registers');
    const outputDiv = document.getElementById('output');
    const inputValueEl = document.getElementById('input-value');
    const nextBtn = document.getElementById('next-btn');
    const resetBtn = document.getElementById('reset-btn');

    // --- 表示を更新するメイン関数 ---
    function updateDisplay() {
        // 1. プログラムコードの表示を更新
        programCodeDiv.innerHTML = '';
        program.forEach((line, index) => {
            const div = document.createElement('div');
            div.textContent = `${index.toString().padStart(2, '0')}: ${line}`;
            if (index === pc) {
                div.classList.add('highlight-pc');
                div.scrollIntoView({ behavior: 'smooth', block: 'center' });
            }
            programCodeDiv.appendChild(div);
        });

        // 2. CPUレジスタの表示を更新
        cpuRegistersDiv.innerHTML = `<span>PC: ${pc}</span>`;

        // 3. メモリの表示を更新
        memoryViewDiv.innerHTML = '';
        memory.forEach((value, index) => {
            const span = document.createElement('span');
            span.id = `mem-${index}`;
            span.textContent = `[${index}]: ${value}`;
            memoryViewDiv.appendChild(span);
        });
    }

    // --- 初期化・リセット関数 ---
    function initialize() {
        memory = new Array(10).fill(0); // 10個のメモリ空間
        pc = 0; // プログラムカウンタを0に
        outputDiv.innerHTML = '';
        inputValueEl.value = '';
        nextBtn.disabled = false;
        document.title = "CPU命令シミュレーター";
        updateDisplay();
    }

    // --- 「次の命令を実行」ボタンの処理 ---
    function executeStep() {
        if (pc >= program.length) return;

        const instructionLine = program[pc];
        const parts = instructionLine.split(' ');
        const command = parts[0];
        
        let shouldIncrementPC = true;

        switch (command) {
            case 'READ': {
                const memAddr = parseInt(parts[1]);
                const numValue = parseInt(inputValueEl.value);
                if (isNaN(numValue)) {
                    alert("入力エリアに半角数字を入れてから実行してください。");
                    return; // PCを進めずに処理を中断
                }
                memory[memAddr] = numValue;
                inputValueEl.value = ''; // 入力欄をクリア
                highlightMemoryCell(memAddr);
                break;
            }
            case 'WRITE': {
                const memAddr = parseInt(parts[1]);
                outputDiv.innerHTML += `> ${memory[memAddr]}<br>`;
                outputDiv.scrollTop = outputDiv.scrollHeight;
                highlightMemoryCell(memAddr, false);
                break;
            }
            case 'ADD': {
                const addr1 = parseInt(parts[1]);
                const addr2 = parseInt(parts[2]);
                const destAddr = parseInt(parts[3]);
                memory[destAddr] = memory[addr1] + memory[addr2];
                highlightMemoryCell(addr1, false);
                highlightMemoryCell(addr2, false);
                highlightMemoryCell(destAddr, true);
                break;
            }
            case 'STOP':
                nextBtn.disabled = true;
                document.title = "プログラム終了";
                shouldIncrementPC = false;
                break;
            default:
                console.error(`不明な命令です: ${command}`);
        }

        if (shouldIncrementPC) {
            pc++;
        }
        updateDisplay();
    }
    
    // メモリセルを一時的にハイライトする関数
    function highlightMemoryCell(index, isDestination = true) {
        const el = document.getElementById(`mem-${index}`);
        if (el) {
            el.classList.add('highlight-mem');
            setTimeout(() => {
                el.classList.remove('highlight-mem');
            }, 1200); // 1.2秒後にハイライトを消す
        }
    }

    // --- イベントリスナーを接続 ---
    nextBtn.addEventListener('click', executeStep);
    resetBtn.addEventListener('click', initialize);

    // --- 最初の読み込み時に初期化 ---
    initialize();
</script>

</body>
</html>
