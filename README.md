<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BBS 專業臨床評估系統</title>
    <style>
        :root {
            --primary: #2c3e50; --secondary: #3498db;
            --low: #27ae60; --mid: #f39c12; --high: #e74c3c;
        }
        body { font-family: -apple-system, "Microsoft JhengHei", sans-serif; background: #f0f2f5; margin: 0; padding: 15px; display: flex; justify-content: center; }
        .app-container { width: 100%; max-width: 600px; }
        .section { display: none; background: white; padding: 20px; border-radius: 16px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); }
        .section.active { display: block; animation: fadeIn 0.3s; }
        @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
        
        h2 { color: var(--primary); font-size: 1.3rem; margin: 0 0 15px 0; padding-bottom: 10px; border-bottom: 2px solid #eee; }
        .progress { font-size: 0.9rem; color: #7f8c8d; margin-bottom: 10px; font-weight: bold; }
        
        .criteria-list { background: #f8f9fa; padding: 12px; border-radius: 8px; margin-bottom: 20px; font-size: 0.95rem; line-height: 1.6; color: #34495e; }
        .criteria-item { margin-bottom: 5px; display: block; }
        .criteria-item b { color: var(--secondary); }

        .eval-ui { margin-bottom: 20px; text-align: center; }
        .timer-display { font-size: 2.5rem; font-weight: bold; font-family: monospace; color: var(--primary); margin: 10px 0; }
        
        .score-group { display: flex; flex-direction: column; gap: 8px; }
        .score-btn { width: 100%; padding: 12px; border: 1px solid #ddd; background: white; border-radius: 10px; text-align: left; cursor: pointer; font-size: 1rem; }
        .score-btn.active { background: var(--secondary); color: white; border-color: var(--secondary); }

        .side-selector { display: flex; gap: 10px; margin-bottom: 15px; }
        .side-btn { flex: 1; padding: 8px; border: 1px solid #ddd; border-radius: 5px; background: #eee; cursor: pointer; }
        .side-btn.active { background: var(--primary); color: white; }

        .nav-btns { display: flex; gap: 10px; margin-top: 25px; }
        .btn { flex: 1; padding: 15px; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; font-size: 1rem; }
        .btn-next { background: var(--primary); color: white; }
        .btn-prev { background: #bdc3c7; color: white; }
        
        .total-score { font-size: 4rem; font-weight: bold; margin: 10px 0; }
        .risk-banner { font-size: 1.5rem; padding: 15px; border-radius: 10px; color: white; margin: 15px 0; font-weight: bold; text-align: center; }
        table { width: 100%; font-size: 0.85rem; border-collapse: collapse; margin-top: 15px; }
        td, th { border: 1px solid #eee; padding: 10px; text-align: left; }
    </style>
</head>
<body>

<div class="app-container">
    <div class="section active" id="sec-1">
        <div class="progress">病人資料登錄</div>
        <h2>基本資料</h2>
        <div style="margin-bottom:20px;">
            <label>病歷號</label>
            <input type="text" id="p_id" style="width:100%; padding:12px; margin-top:8px; border:1px solid #ddd; border-radius:8px;" placeholder="請輸入編號">
        </div>
        <div style="margin-bottom:20px;">
            <label>病人類型</label>
            <select id="p_type" style="width:100%; padding:12px; margin-top:8px; border:1px solid #ddd; border-radius:8px;">
                <option>高齡者</option><option>中風</option><option>巴金森氏症</option><option>前庭功能障礙</option>
            </select>
        </div>
        <div class="nav-btns"><button class="btn btn-next" onclick="go(2)">進入第一項評估</button></div>
    </div>

    <div id="dynamic-items"></div>

    <div class="section" id="sec-16">
        <div class="progress">評估完成 - 報告生成</div>
        <div id="res_p_info" style="font-weight:bold; margin-bottom:10px;"></div>
        <div style="text-align:center;">
            <div>BBS 總積分</div>
            <div class="total-score" id="res_total">0</div>
            <div id="res_risk" class="risk-banner"></div>
        </div>
        <table id="res_table">
            <thead><tr><th>項目</th><th>得分</th><th>備註</th></tr></thead>
            <tbody></tbody>
        </table>
        <div class="nav-btns">
            <button class="btn btn-prev" onclick="go(1)">修改資料</button>
            <button class="btn btn-next" style="background:#27ae60" onclick="location.reload()">存檔並重置</button>
        </div>
    </div>
</div>

<script>
    const bbsData = [
        { id: 1, name: "從坐位站起 (Sitting to standing)", type: "click", criteria: ["4分：不需用手支撐，獨立站起","3分：需用手支撐，獨立站起","2分：多次嘗試後能站起","1分：需他人少量協助","0分：需他人大量協助"] },
        { id: 2, name: "獨立站立 (Standing unsupported)", type: "timer", target: 120, criteria: ["4分：穩定站立 2 分鐘","3分：監督下站立 2 分鐘","2分：不需支撐站立 30 秒","1分：需多次嘗試，不支撐站立 30 秒","0分：無法在不支撐下站立 30 秒"] },
        { id: 3, name: "獨立坐著 (Sitting unsupported)", type: "timer", target: 120, criteria: ["4分：雙腳著地穩定坐 2 分鐘","3分：監督下坐 2 分鐘","2分：坐 30 秒","1分：坐 10 秒","0分：無法不支撐坐 10 秒"] },
        { id: 4, name: "從站位坐下 (Standing to sitting)", type: "click", criteria: ["4分：動作流暢，少用手支撐","3分：需用手控制下降","2分：腿後部緊貼椅子以控制下降","1分：獨立完成但下降速度不穩","0分：需協助"] },
        { id: 5, name: "床椅轉移 (Transfers)", type: "click", criteria: ["4分：不需用手即可安全轉移","3分：需用手輔助安全轉移","2分：需口頭指導或監督下完成","1分：需一人協助","0分：需二人協助"] },
        { id: 6, name: "閉眼站立 (Standing eyes closed)", type: "timer", target: 10, criteria: ["4分：穩定站立 10 秒","3分：穩定站立 10 秒 (需監督)","2分：站立 3 秒","1分：無法閉眼 3 秒但能保持站立","0分：閉眼即跌倒"] },
        { id: 7, name: "雙腳併攏站立 (Standing feet together)", type: "timer", target: 60, criteria: ["4分：獨立併攏站立 1 分鐘","3分：監督下併攏站立 1 分鐘","2分：獨立併攏站立 30 秒","1分：需協助才能併攏，但能站立 15 秒","0分：需協助才能併攏且無法站立 15 秒"] },
        { id: 8, name: "站立前伸 (Reaching forward)", type: "click", criteria: ["4分：前伸 > 25 公分","3分：前伸 > 12.5 公分","2分：前伸 > 5 公分","1分：前伸需監督","0分：嘗試時失去平衡或跌倒"] },
        { id: 9, name: "從地面拾物 (Pick up object)", type: "click", criteria: ["4分：輕鬆安全拾起","3分：獨立拾起，需監督","2分：無法拾起，距離物體 2-5 公分並保持平衡","1分：無法拾起，嘗試時需監督","0分：嘗試時失去平衡或需協助"] },
        { id: 10, name: "回頭向後看 (Turning behind)", type: "click", criteria: ["4分：雙側均能轉向後方並轉移重心","3分：僅一側能看向後方，另一側較差","2分：僅轉向側面但能保持平衡","1分：轉頭時需監督","0分：需協助以防跌倒"] },
        { id: 11, name: "360度轉圈 (Turning 360°)", type: "timer", target: 4, special: "side", criteria: ["4分：雙向轉圈各需 4 秒內","3分：單向轉圈 4 秒內","2分：安全轉完但速度緩慢","1分：需近距離監督或口頭指導","0分：轉動時需協助"] },
        { id: 12, name: "階梯踏步 (Stool stepping)", type: "timer", target: 20, criteria: ["4分：獨立安全在 20 秒內完成 8 步","3分：獨立安全完成 8 步 (> 20 秒)","2分：在監督下完成 4 步","1分：需少量協助完成 > 2 步","0分：需協助以防跌倒"] },
        { id: 13, name: "串聯站立 (Tandem stance)", type: "timer", target: 30, special: "foot", criteria: ["4分：腳跟對腳尖獨立站立 30 秒","3分：腳稍跨前且能獨立站立 30 秒","2分：獨立踏出一小步並站立 30 秒","1分：踏步需協助，但能站立 15 秒","0分：踏步或站立時失去平衡"] },
        { id: 14, name: "單腳站立 (One leg stand)", type: "timer", target: 10, special: "compare", criteria: ["4分：獨立抬腳站立 > 10 秒","3分：獨立抬腳站立 5-10 秒","2分：獨立抬腳站立 3 秒以上","1分：嘗試抬腳維持 < 3 秒但能獨立站立","0分：無法嘗試或需協助以防跌倒"] }
    ];

    let currentScores = {};
    let currentTimes = {};
    let runTimers = {};

    function init() {
        const wrapper = document.getElementById('dynamic-items');
        bbsData.forEach((item, index) => {
            const secNum = index + 2;
            const div = document.createElement('div');
            div.className = 'section';
            div.id = `sec-${secNum}`;
            
            let specialUI = "";
            if(item.special === "side") specialUI = `<div class="side-selector"><button class="side-btn active" onclick="setSide(this)">向左轉</button><button class="side-btn" onclick="setSide(this)">向右轉</button></div>`;
            if(item.special === "foot") specialUI = `<div class="side-selector"><button class="side-btn active" onclick="setSide(this)">左腳在後</button><button class="side-btn" onclick="setSide(this)">右腳在後</button></div>`;
            if(item.special === "compare") specialUI = `<div class="side-selector"><span>測試雙腳，系統將自動記錄較差側。</span></div><div class="timer-box" style="margin-bottom:10px;"><button class="side-btn" id="l-leg" onclick="markLeg('L')">記錄左腳秒數</button> <button class="side-btn" id="r-leg" onclick="markLeg('R')">記錄右腳秒數</button></div>`;

            div.innerHTML = `
                <div class="progress">項目評估 ${index + 1} / 14</div>
                <h2>${item.name}</h2>
                <div class="criteria-list">${item.criteria.map(c => `<span class="criteria-item">${c}</span>`).join('')}</div>
                <div class="eval-ui">
                    ${item.type === 'timer' ? `<div class="timer-display" id="disp-${item.id}">0.0s</div><button class="btn" style="background:#3498db; color:white; margin-bottom:10px;" onclick="startClock(${item.id})">開始計時</button> <button class="btn" style="background:#e67e22; color:white;" onclick="stopClock(${item.id})">停止</button>` : ''}
                    ${specialUI}
                </div>
                <div class="score-group">
                    ${[4,3,2,1,0].map(s => `<button class="score-btn" id="btn-${item.id}-${s}" onclick="recordScore(${item.id}, ${s})">${s} 分 - ${item.criteria[4-s].split('：')[1]}</button>`).join('')}
                </div>
                <div class="nav-btns">
                    <button class="btn btn-prev" onclick="go(${secNum-1})">上一步</button>
                    <button class="btn btn-next" onclick="${secNum === 15 ? 'showReport()' : 'go('+(secNum+1)+')'}">下一步</button>
                </div>
            `;
            wrapper.appendChild(div);
        });
    }

    function go(n) {
        document.querySelectorAll('.section').forEach(s => s.classList.remove('active'));
        document.getElementById(`sec-${n}`).classList.add('active');
        window.scrollTo(0,0);
    }

    function setSide(btn) {
        btn.parentElement.querySelectorAll('.side-btn').forEach(b => b.classList.remove('active'));
        btn.classList.add('active');
    }

    let legData = { L: 0, R: 0 };
    function markLeg(side) {
        const time = parseFloat(document.getElementById('disp-14').innerText);
        legData[side] = time;
        document.getElementById(side === 'L' ? 'l-leg' : 'r-leg').innerText = (side==='L'?'左':'右') + "腳: " + time + "s";
        // 自動比較
        const minTime = Math.min(legData.L || 999, legData.R || 999);
        currentTimes[14] = `左${legData.L}s/右${legData.R}s (取小者: ${minTime}s)`;
    }

    function startClock(id) {
        const start = Date.now();
        if(runTimers[id]) clearInterval(runTimers[id]);
        runTimers[id] = setInterval(() => {
            const t = ((Date.now() - start) / 1000).toFixed(1);
            document.getElementById(`disp-${id}`).innerText = t + 's';
            currentTimes[id] = t + 's';
        }, 100);
    }

    function stopClock(id) { clearInterval(runTimers[id]); }

    function recordScore(id, s) {
        currentScores[id] = s;
        document.querySelectorAll(`#sec-${id+1} .score-btn`).forEach(b => b.classList.remove('active'));
        document.getElementById(`btn-${id}-${s}`).classList.add('active');
    }

    function showReport() {
        let total = 0;
        let tbody = "";
        bbsData.forEach(item => {
            const s = currentScores[item.id] || 0;
            total += s;
            tbody += `<tr><td>${item.id}</td><td>${s}</td><td>${currentTimes[item.id] || '-'}</td></tr>`;
        });
        document.getElementById('res_p_info').innerText = `病歷號：${document.getElementById('p_id').value || '未填'} | 類型：${document.getElementById('p_type').value}`;
        document.getElementById('res_total').innerText = total;
        const banner = document.getElementById('res_risk');
        if (total >= 41) { banner.innerText = "低跌倒風險 (Low Risk)"; banner.style.background = 'var(--low)'; }
        else if (total >= 21) { banner.innerText = "中度跌倒風險 (Medium Risk)"; banner.style.background = 'var(--mid)'; }
        else { banner.innerText = "高跌倒風險 (High Risk)"; banner.style.background = 'var(--high)'; }
        document.querySelector('#res_table tbody').innerHTML = tbody;
        go(16);
    }

    init();
</script>
</body>
</html>
