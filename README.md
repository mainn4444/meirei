# meirei
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CPU命令シミュレーター</title>
    <style>
        body {
            font-family: 'Helvetica Neue', Arial, 'Hiragino Kaku Gothic ProN', 'Hiragino Sans', Meiryo, sans-serif;
            display: flex;
            justify-content: center;
            align-items: flex-start;
            padding: 20px;
            background-color: #f0f2f5;
            gap: 20px;
            flex-wrap: wrap;
        }
        .container {
            background-color: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0,0,0,0.1);
            min-width: 300px;
        }
        h2 {
            margin-top: 0;
            border-bottom: 2px solid #007bff;
            padding-bottom: 10px;
            color: #333;
        }
        h3 {
            margin-top: 15px;
            margin-bottom: 5px;
            font-size: 1em;
            color: #555;
        }
        #program-code, #memory-view, #output {
            border: 1px solid #ccc;
            border-radius: 4px;
            padding: 10px;
            height: 200px;
            overflow-y: auto;
            background-color: #f9f9f9;
        }
        #program-code div.highlight {
            background-color: #fff3cd;
            font-weight: bold;
        }
        #memory-view span, #cpu-registers span {
            display: inline-block;
            width: 80px;
            border: 1px solid #ddd;
            padding: 5px;
            margin: 2px;
            text-align: center;
            border-radius: 4px;
        }
        #memory-view span.highlight, #cpu-registers span.highlight {
            background-color: #d1ecf1;
            transition: background-color 0.3s;
        }
        #controls, #io-area {
            display: flex;
            flex-direction: column;
            gap: 10px;
        }
        button {
            padding: 10px 15px;
            border: none;
            border-radius: 5px;
            background-color: #007bff;
            color: white;
            font-size: 16px;
            cursor: pointer;
            transition: background-color 0.3s;
        }
        button:hover {
            background-color: #0056b3;
        }
        button:disabled {
            background-color: #ccc;
            cursor: not-allowed;
        }
        input[type="number"] {
            padding: 8px;
            border-radius: 4px;
            border: 1px solid #ccc;
            width: calc(100% - 18px);
        }
        .explanation {
            margin-top: 10px;
            padding: 10px;
            background-color: #e9ecef;
            border-left: 5px solid #007bff;
            font-size: 0.9em;
        }
    </style>
</head>
<body>

    <div class="container">
        <h2>📜 プログラムコード</h2>
        <div id="program-code"></div>
        <div class="explanation">
            コンピュータに実行させたい命令の一覧です。上から順番に実行されます。
        </div>
    </div>

    <div class="container">
        <h2>🧠 CPUとメモリ</h2>
        <h3>CPU レジスタ (一時的な作業場所)</h3>
        <div id="cpu-registers">
            <span>PC: 0</span>
        </div>
        <h3 style="margin-top: 20px;">📝 メインメモリ (データを保存する場所)</h3>
        <div id="memory-view"></div>
        <div class="explanation">
           CPUが計算や処理に使うデータを一時的に記憶する場所です。
        </div>
    </div>

    <div class="container">
        <h2>↔️ 入出力と操作</h2>
        <div id="io-area">
            <h3>📥 入力 (INPUT)</h3>
            <input type="number" id="input-value" placeholder="数字を入力してREAD命令を実行">
        </div>
        <div id="output-area" style="margin-top:20px;">
            <h3>📤 出力 (OUTPUT)</h3>
            <div id="output"></div>
        </div>
        <div id="controls" style="margin-top: 20px;">
            <button id="reset-btn">リセット</button>
            <button id="next-btn">次の命令を実行 →</button>
        </div>
    </div>

<script>
    // --- シミュレーターの初期設定 ---
    const program = [
        "READ 0",   // 入力された値をメモリの0番地に読み込む
        "READ 1",   // 入力された値をメモリの1番地に読み込む
        "ADD 0 1",  // メモリの0番地と1番地の値を足し算する
        "WRITE 0",  // メモリの0番地の値を出力する
        "STOP"      // プログラムを停止する
    ];

    let memory = new Array(10).fill(0); // 10個のメモリ空間を0で初期化
    let pc = 0; // プログラムカウンタ: 次に実行する命令の場所

    // --- HTML要素の取得 ---
    const programCodeDiv = document.getElementById('program-code');
    const memoryViewDiv = document.getElementById('memory-view');
    const cpuRegistersDiv = document.getElementById('cpu-registers');
    const outputDiv = document.getElementById('output');
    const inputValue = document.getElementById('input-value');
    const nextBtn = document.getElementById('next-btn');
    const resetBtn = document.getElementById('reset-btn');

    // --- 画面表示を更新する関数 ---
    function updateDisplay() {
        // プログラムコードの表示
        programCodeDiv.innerHTML = '';
        program.forEach((line, index) => {
            const div = document.createElement('div');
            div.textContent = `${index}: ${line}`;
            if (index === pc) {
                div.classList.add('highlight'); // 実行中の行をハイライト
            }
            programCodeDiv.appendChild(div);
        });

        // CPUレジスタの表示
        cpuRegistersDiv.innerHTML = `<span>PC: ${pc}</span>`;

        // メモリの表示
        memoryViewDiv.innerHTML = '';
        memory.forEach((value, index) => {
            const span = document.createElement('span');
            span.id = `mem-${index}`;
            span.textContent = `[${index}]: ${value}`;
            memoryViewDiv.appendChild(span);
        });
    }

    // --- リセット処理 ---
    function reset() {
        memory.fill(0);
        pc = 0;
        outputDiv.innerHTML = '';
        inputValue.value = '';
        nextBtn.disabled = false;
        updateDisplay();
    }

    // --- 1ステップ実行処理 ---
    function step() {
        if (pc >= program.length) return;

        // 前回ハイライトを消す
        document.querySelectorAll('.highlight').forEach(el => el.classList.remove('highlight'));

        const instruction = program[pc].split(' ');
        const command = instruction[0];

        switch (command) {
            case 'READ': {
                const memAddr = parseInt(instruction[1]);
                const val = parseInt(inputValue.value);
                if (isNaN(val)) {
                    alert("入力エリアに有効な数字を入力してください。");
                    return;
                }
                memory[memAddr] = val;
                inputValue.value = ''; // 入力欄をクリア
                highlightElement(`mem-${memAddr}`);
                break;
            }
            case 'WRITE': {
                const memAddr = parseInt(instruction[1]);
                outputDiv.innerHTML += `> ${memory[memAddr]}\n`;
                highlightElement(`mem-${memAddr}`);
                break;
            }
            case 'ADD': {
                const addr1 = parseInt(instruction[1]);
                const addr2 = parseInt(instruction[2]);
                memory[addr1] = memory[addr1] + memory[addr2];
                highlightElement(`mem-${addr1}`);
                highlightElement(`mem-${addr2}`);
                break;
            }
            case 'STOP':
                nextBtn.disabled = true;
                alert("プログラムが終了しました。");
                break;
        }

        if (command !== 'STOP') {
            pc++;
        }
        updateDisplay();
    }
    
    // 特定の要素を一時的にハイライトする関数
    function highlightElement(id) {
        const el = document.getElementById(id);
        if (el) {
            el.classList.add('highlight');
            setTimeout(() => {
                el.classList.remove('highlight');
            }, 1000);
        }
    }

    // --- イベントリスナーの設定 ---
    nextBtn.addEventListener('click', step);
    resetBtn.addEventListener('click', reset);

    // --- 初期表示 ---
    reset();
</script>

</body>
</html>
