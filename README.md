<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BBS 平衡評估助手</title>
    <style>
        :root {
            --primary: #2c3e50;
            --low-risk: #27ae60;
            --mid-risk: #f39c12;
            --high-risk: #e74c3c;
        }
        body { font-family: -apple-system, sans-serif; background: #f4f7f6; margin: 0; padding: 20px; }
        .card { background: white; padding: 15px; border-radius: 12px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin-bottom: 20px; }
        h2 { color: var(--primary); border-left: 5px solid var(--primary); padding-left: 10px; }
        .input-group { margin-bottom: 15px; }
        label { display: block; margin-bottom: 5px; font-weight: bold; }
        input, select { width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 6px; box-sizing: border-box; }
        
        .test-item { border-bottom: 1px solid #eee; padding: 15px 0; }
        .timer-display { font-size: 24px; font-weight: bold; color: var(--primary); margin: 10px 0; }
        .btn { padding: 10px 20px; border: none; border-radius: 6px; cursor: pointer; font-weight: bold; }
        .btn-start { background: #3498db; color: white; }
        .btn-stop { background: #e67e22; color: white; margin-left: 10px; }
        .btn-submit { background: var(--primary); color: white; width: 100%; font-size: 18px; margin-top: 20px; }
        
        .score-select { margin-top: 10px; display: flex; gap: 5px; }
        .score-opt { flex: 1; text-align: center; padding: 8px; border: 1px solid #ddd; border-radius: 4px; font-size: 12px; }
        .score-opt.active { background: var(--primary); color: white; border-color: var(--primary); }
        
        #resultArea { display: none; text-align: center; }
        .risk-tag { font-size: 24px; font-weight: bold; padding: 10px; border-radius: 8px; color: white; display: inline-block; margin-top: 10px; }
        
        table { width: 100%; border-collapse: collapse; margin-top: 20px; background: white; font-size: 14px; }
        th, td { border: 1px solid #ddd; padding: 10px; text-align: left; }
        th { background: #eee; }
    </style>
</head>
<body>

    <div class="card">
        <h2>病人資訊</h2>
        <div class="input-group">
            <label>病歷號</label>
            <input type="text" id="patientId" placeholder="請輸入病歷號">
        </div>
        <div class="input-group">
            <label>病人類型</label>
            <select id="patientType">
                <option value="一般高齡">一般高齡</option>
                <option value="中風">中風</option>
                <option value="巴金森氏症">巴金森氏症</option>
                <option value="其他">其他</option>
            </select>
        </div>
    </div>

    <div class="card">
        <h2>計時評估項 (6,7,11,12,13,14)</h2>
        <div id="timerContainer"></div>
        
        <h2>其他項目總分 (1-5, 8-10)</h2>
        <div class="input-group">
            <label>這 8 項的點選分數總和 (0-32)</label>
            <input type="number" id="otherScore" min="0" max="32" value="0">
        </div>

        <button class="btn btn-submit" onclick="calculateResult()">計算結果並存檔</button>
    </div>

    <div id="resultArea" class="card">
        <h2>評估結果</h2>
        <div id="scoreDisplay" style="font-size: 20px;"></div>
        <div id="riskTag" class="risk-tag"></div>
        <div id="timeDetails" style="margin-top: 15px; text-align: left; font-size: 14px; color: #666;"></div>
    </div>

    <div class="card">
        <h2>歷史紀錄 (本機儲存)</h2>
        <div style="overflow-x: auto;">
            <table id="historyTable">
                <thead>
                    <tr>
                        <th>日期</th>
                        <th>病歷號</th>
                        <th>類型</th>
                        <th>總分</th>
                        <th>風險</th>
                    </tr>
                </thead>
                <tbody></tbody>
            </table>
        </div>
        <button class="btn" style="margin-top:10px; background:#95a5a6; color:white;" onclick="clearHistory()">清除所有紀錄</button>
    </div>

<script>
    const items = [
        { id: 6, name: "閉眼站立 (目標10s)", thresholds: [10, 3, 0], scores: [4, 2, 1, 0] },
        { id: 7, name: "雙腳併攏站立 (目標60s)", thresholds: [60, 30, 15], scores: [4, 2, 1, 0] },
        { id: 11, name: "360度轉圈 (目標4s)", thresholds: [4, 10, 20], scores: [4, 2, 1, 0] },
        { id: 12, name: "階梯踏步 (目標20s)", thresholds: [20, 40, 60], scores: [4, 3, 2, 0] },
        { id: 13, name: "串聯站立 (目標30s)", thresholds: [30, 15, 5], scores: [4, 2, 1, 0] },
        { id: 14, name: "單腳站立 (目標10s)", thresholds: [10, 5, 3], scores: [4, 3, 2, 1] }
    ];

    let timers = {};
    let results = {};

    // 初始化計時項目 UI
    const container = document.getElementById('timerContainer');
    items.forEach(item => {
        results[item.id] = { time: 0, score: 0 };
        const div = document.createElement('div');
        div.className = 'test-item';
        div.innerHTML = `
            <label>${item.id}. ${item.name}</label>
            <div class="timer-display" id="display-${item.id}">0.0s</div>
            <button class="btn btn-start" onclick="startTimer(${item.id})">開始</button>
            <button class="btn btn-stop" onclick="stopTimer(${item.id})">停止</button>
            <div class="score-select" id="score-row-${item.id}">
                ${item.scores.map(s => `<div class="score-opt" id="opt-${item.id}-${s}">${s}分</div>`).join('')}
            </div>
        `;
        container.appendChild(div);
    });

    function startTimer(id) {
        if (timers[id]) clearInterval(timers[id].interval);
        const startTime = Date.now();
        timers[id] = {
            interval: setInterval(() => {
                const elapsed = ((Date.now() - startTime) / 1000).toFixed(1);
                document.getElementById(`display-${id}`).innerText = elapsed + 's';
                results[id].time = parseFloat(elapsed);
            }, 100)
        };
    }

    function stopTimer(id) {
        if (timers[id]) {
            clearInterval(timers[id].interval);
            autoScore(id, results[id].time);
        }
    }

    function autoScore(id, time) {
        const config = items.find(i => i.id === id);
        let score = 0;
        // 簡易自動評分邏輯 (可依臨床需求精確調整)
        if (time >= config.thresholds[0]) score = config.scores[0];
        else if (time >= config.thresholds[1]) score = config.scores[1];
        else if (time >= config.thresholds[2]) score = config.scores[2];
        else score = 0;

        results[id].score = score;
        
        // 更新 UI 狀態
        document.querySelectorAll(`#score-row-${id} .score-opt`).forEach(el => el.classList.remove('active'));
        const activeOpt = document.getElementById(`opt-${id}-${score}`);
        if(activeOpt) activeOpt.classList.add('active');
    }

    function calculateResult() {
        const pId = document.getElementById('patientId').value;
        if (!pId) { alert("請輸入病歷號"); return; }

        let timerTotal = 0;
        let timeLog = "";
        items.forEach(item => {
            timerTotal += results[item.id].score;
            timeLog += `項目${item.id}: ${results[item.id].time}s (${results[item.id].score}分) | `;
        });

        const otherTotal = parseInt(document.getElementById('otherScore').value) || 0;
        const finalScore = timerTotal + otherTotal;

        let risk = "";
        let color = "";
        if (finalScore >= 41) { risk = "低跌倒風險"; color = 'var(--low-risk)'; }
        else if (finalScore >= 21) { risk = "中度跌倒風險"; color = 'var(--mid-risk)'; }
        else { risk = "高跌倒風險"; color = 'var(--high-risk)'; }

        // 顯示結果
        document.getElementById('resultArea').style.display = 'block';
        document.getElementById('scoreDisplay').innerText = `總分：${finalScore} / 56`;
        const rt = document.getElementById('riskTag');
        rt.innerText = risk;
        rt.style.backgroundColor = color;
        document.getElementById('timeDetails').innerText = "計時詳情：" + timeLog;

        // 存入 LocalStorage
        const record = {
            date: new Date().toLocaleString(),
            id: pId,
            type: document.getElementById('patientType').value,
            score: finalScore,
            risk: risk
        };
        saveRecord(record);
    }

    function saveRecord(record) {
        let history = JSON.parse(localStorage.getItem('bbs_history') || '[]');
        history.unshift(record);
        localStorage.setItem('bbs_history', JSON.stringify(history));
        renderHistory();
    }

    function renderHistory() {
        const history = JSON.parse(localStorage.getItem('bbs_history') || '[]');
        const tbody = document.querySelector('#historyTable tbody');
        tbody.innerHTML = history.map(r => `
            <tr>
                <td>${r.date}</td>
                <td>${r.id}</td>
                <td>${r.type}</td>
                <td>${r.score}</td>
                <td>${r.risk}</td>
            </tr>
        `).join('');
    }

    function clearHistory() {
        if(confirm("確定要刪除所有紀錄嗎？")) {
            localStorage.removeItem('bbs_history');
            renderHistory();
        }
    }

    // 初始載入紀錄
    renderHistory();
</script>

</body>
</html>
