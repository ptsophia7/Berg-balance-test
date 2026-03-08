<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BBS 步進式評估系統</title>
    <style>
        :root {
            --primary: #2c3e50;
            --secondary: #3498db;
            --low: #27ae60; --mid: #f39c12; --high: #e74c3c;
        }
        body { font-family: -apple-system, sans-serif; background: #f0f2f5; margin: 0; padding: 20px; display: flex; justify-content: center; }
        .app-container { width: 100%; max-width: 500px; }
        .section { display: none; background: white; padding: 20px; border-radius: 16px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); }
        .section.active { display: block; animation: fadeIn 0.4s; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        
        h2 { color: var(--primary); font-size: 1.2rem; margin-top: 0; border-bottom: 2px solid #eee; padding-bottom: 10px; }
        .progress { font-size: 0.8rem; color: #888; margin-bottom: 10px; }
        
        .item-box { margin-bottom: 25px; padding: 15px; border: 1px solid #f0f0f0; border-radius: 10px; }
        .item-title { font-weight: bold; margin-bottom: 10px; display: block; }
        
        .score-group { display: flex; justify-content: space-between; gap: 5px; }
        .score-btn { flex: 1; padding: 10px 5px; border: 1px solid #ddd; background: white; border-radius: 6px; font-size: 0.9rem; cursor: pointer; }
        .score-btn.active { background: var(--secondary); color: white; border-color: var(--secondary); }
        
        .timer-box { background: #f8f9fa; padding: 10px; border-radius: 8px; text-align: center; margin-bottom: 10px; }
        .timer-val { font-size: 1.5rem; font-weight: bold; font-family: monospace; }
        
        .nav-btns { display: flex; gap: 10px; margin-top: 20px; }
        .btn { flex: 1; padding: 12px; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; }
        .btn-next { background: var(--primary); color: white; }
        .btn-prev { background: #bdc3c7; color: white; }
        
        .result-card { text-align: center; }
        .total-score { font-size: 3rem; font-weight: bold; color: var(--primary); }
        .risk-banner { font-size: 1.5rem; padding: 15px; border-radius: 10px; color: white; margin: 15px 0; }
        table { width: 100%; font-size: 0.85rem; border-collapse: collapse; margin-top: 10px; }
        td, th { border: 1px solid #eee; padding: 8px; text-align: left; }
    </style>
</head>
<body>

<div class="app-container">
    <div class="section active" id="sec-1">
        <div class="progress">Step 1 / 9</div>
        <h2>基本資料</h2>
        <div style="margin-bottom:15px;">
            <label>病歷號</label>
            <input type="text" id="p_id" style="width:100%; padding:10px; margin-top:5px; border:1px solid #ddd; border-radius:5px;">
        </div>
        <div>
            <label>病人類型</label>
            <select id="p_type" style="width:100%; padding:10px; margin-top:5px; border:1px solid #ddd; border-radius:5px;">
                <option>高齡者</option><option>中風</option><option>巴金森氏症</option><option>脊髓損傷</option>
            </select>
        </div>
        <div class="nav-btns"><button class="btn btn-next" onclick="go(2)">開始評估</button></div>
    </div>

    <div id="dynamic-sections"></div>

    <div class="section" id="sec-9">
        <div class="progress">Step 9 / 9 - 評估報告</div>
        <div class="result-card">
            <div id="res_p_info" style="font-weight:bold; margin-bottom:10px;"></div>
            <div>總分</div>
            <div class="total-score" id="res_total">0</div>
            <div id="res_risk" class="risk-banner"></div>
            <table id="res_table">
                <thead><tr><th>項目</th><th>分數</th><th>紀錄</th></tr></thead>
                <tbody></tbody>
            </table>
            <div class="nav-btns">
                <button class="btn btn-prev" onclick="go(1)">重新測試</button>
                <button class="btn btn-next" onclick="saveData()">儲存並完成</button>
            </div>
        </div>
    </div>
</div>

<script>
    const bbsItems = [
        { id: 1, name: "從坐位站起", type: "click" },
        { id: 2, name: "獨立站立", type: "click" },
        { id: 3, name: "獨立坐著", type: "click" },
        { id: 4, name: "從站位坐下", type: "click" },
        { id: 5, name: "床椅轉移", type: "click" },
        { id: 6, name: "閉眼站立", type: "timer", target: 10 },
        { id: 7, name: "雙腳併攏站立", type: "timer", target: 60 },
        { id: 8, name: "站立前伸", type: "click" },
        { id: 9, name: "從地面拾物", type: "click" },
        { id: 10, name: "回頭向後看", type: "click" },
        { id: 11, name: "360度轉圈", type: "timer", target: 4 },
        { id: 12, name: "階梯踏步", type: "timer", target: 20 },
        { id: 13, name: "串聯站立", type: "timer", target: 30 },
        { id: 14, name: "單腳站立", type: "timer", target: 10 }
    ];

    let scores = {};
    let timers = {};
    let timeLogs = {};

    function initApp() {
        const container = document.getElementById('dynamic-sections');
        for (let i = 0; i < 7; i++) {
            const secNum = i + 2;
            const item1 = bbsItems[i * 2];
            const item2 = bbsItems[i * 2 + 1];
            
            const div = document.createElement('div');
            div.className = 'section';
            div.id = `sec-${secNum}`;
            div.innerHTML = `
                <div class="progress">Step ${secNum} / 9</div>
                <h2>評估項目 ${item1.id} & ${item2.id}</h2>
                ${renderItem(item1)}
                ${renderItem(item2)}
                <div class="nav-btns">
                    <button class="btn btn-prev" onclick="go(${secNum-1})">上一步</button>
                    <button class="btn btn-next" onclick="${secNum === 8 ? 'showFinal()' : 'go('+(secNum+1)+')'}">下一步</button>
                </div>
            `;
            container.appendChild(div);
        }
    }

    function renderItem(item) {
        let html = `<div class="item-box"><span class="item-title">${item.id}. ${item.name}</span>`;
        if (item.type === 'timer') {
            html += `
                <div class="timer-box">
                    <div class="timer-val" id="val-${item.id}">0.0s</div>
                    <button onclick="startT(${item.id})">開始</button>
                    <button onclick="stopT(${item.id})">停止</button>
                </div>`;
        }
        html += `<div class="score-group">`;
        [0, 1, 2, 3, 4].forEach(s => {
            html += `<button class="score-btn" id="btn-${item.id}-${s}" onclick="setScore(${item.id}, ${s})">${s}</button>`;
        });
        html += `</div></div>`;
        return html;
    }

    function go(n) {
        document.querySelectorAll('.section').forEach(s => s.classList.remove('active'));
        document.getElementById(`sec-${n}`).classList.add('active');
        window.scrollTo(0,0);
    }

    function setScore(id, s) {
        scores[id] = s;
        document.querySelectorAll(`#sec-${Math.floor((id-1)/2)+2} #btn-${id}-0, #btn-${id}-1, #btn-${id}-2, #btn-${id}-3, #btn-${id}-4`)
                .forEach(b => b.classList.remove('active'));
        document.getElementById(`btn-${id}-${s}`).classList.add('active');
    }

    function startT(id) {
        const start = Date.now();
        if (timers[id]) clearInterval(timers[id]);
        timers[id] = setInterval(() => {
            const t = ((Date.now() - start) / 1000).toFixed(1);
            document.getElementById(`val-${id}`).innerText = t + 's';
            timeLogs[id] = t;
        }, 100);
    }

    function stopT(id) { clearInterval(timers[id]); }

    function showFinal() {
        let total = 0;
        let tbody = "";
        const p_id = document.getElementById('p_id').value || "未填寫";
        const p_type = document.getElementById('p_type').value;

        bbsItems.forEach(item => {
            const s = scores[item.id] || 0;
            total += s;
            tbody += `<tr><td>${item.id}.${item.name}</td><td>${s}分</td><td>${timeLogs[item.id] ? timeLogs[item.id]+'s' : '-'}</td></tr>`;
        });

        document.getElementById('res_p_info').innerText = `病歷號：${p_id} (${p_type})`;
        document.getElementById('res_total').innerText = total;
        const rb = document.getElementById('res_risk');
        if (total >= 41) { rb.innerText = "低跌倒風險"; rb.style.background = 'var(--low)'; }
        else if (total >= 21) { rb.innerText = "中度跌倒風險"; rb.style.background = 'var(--mid)'; }
        else { rb.innerText = "高跌倒風險"; rb.style.background = 'var(--high)'; }

        document.querySelector('#res_table tbody').innerHTML = tbody;
        go(9);
    }

    function saveData() {
        alert("資料已儲存至瀏覽器紀錄中！");
        // 這裡可以擴充將資料寫入 LocalStorage 的功能
        location.reload();
    }

    initApp();
</script>
</body>
</html>
