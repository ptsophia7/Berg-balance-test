<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BBS 臨床快速評估系統</title>
    <style>
        :root { --primary: #2c3e50; --secondary: #3498db; --low: #27ae60; --mid: #f39c12; --high: #e74c3c; }
        body { font-family: -apple-system, sans-serif; background: #f0f2f5; margin: 0; padding: 10px; display: flex; justify-content: center; }
        .app-container { width: 100%; max-width: 550px; }
        .section { display: none; background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        .section.active { display: block; }
        h2 { color: var(--primary); font-size: 1.1rem; margin: 0 0 15px 0; border-bottom: 2px solid #eee; padding-bottom: 8px; }
        .eval-ui { margin-bottom: 15px; text-align: center; }
        .timer-display { font-size: 2.2rem; font-weight: bold; color: var(--primary); }
        .score-group { display: flex; flex-direction: column; gap: 8px; }
        .score-btn { width: 100%; padding: 12px; border: 1px solid #ddd; background: #fff; border-radius: 8px; text-align: left; cursor: pointer; }
        .score-btn.active { background: var(--secondary); color: white; border-color: var(--secondary); }
        .nav-btns { display: flex; gap: 10px; margin-top: 20px; flex-wrap: wrap; }
        .btn { flex: 1; padding: 12px; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; }
        .btn-email { background: #8e44ad; color: white; }
        .btn-next { background: #2c3e50; color: white; }
        .compare-box { background: #ebf5fb; padding: 12px; border-radius: 8px; margin: 10px 0; font-size: 0.9rem; border-left: 5px solid var(--secondary); }
        .total-score { font-size: 3.5rem; font-weight: bold; text-align: center; margin: 5px 0; }
        .risk-banner { font-size: 1.2rem; padding: 10px; border-radius: 8px; color: white; text-align: center; font-weight: bold; }
        .side-btn { padding: 8px; border: 1px solid #ddd; border-radius: 5px; background: #f8f9fa; cursor: pointer; flex: 1; }
    </style>
</head>
<body>

<div class="app-container">
    <div class="section active" id="sec-1">
        <h2>BBS 評估開始</h2>
        <div style="margin-bottom:15px;"><label>病歷號</label><input type="text" id="p_id" style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px;"></div>
        <div style="margin-bottom:15px;"><label>接收 Email</label><input type="email" id="target_email" style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px;"></div>
        <div class="nav-btns"><button class="btn btn-next" onclick="go(2)">開始評估</button></div>
    </div>

    <div id="dynamic-items"></div>

    <div class="section" id="sec-16">
        <h2>評估結果對比</h2>
        <div id="compare_res" class="compare-box">無歷史紀錄</div>
        <div class="total-score" id="res_total">0</div>
        <div id="res_risk" class="risk-banner"></div>
        <div class="nav-btns">
            <button class="btn btn-email" onclick="sendEmail()">發送專業格式報告</button>
            <button class="btn btn-next" onclick="saveAndReset()">存檔重置</button>
        </div>
    </div>
</div>

<script>
    const bbsData = [
        { id: 1, name: "坐到站", type: "click", s: ["獨立", "手撐", "多次嘗試", "少量協助", "大量協助"] },
        { id: 2, name: "獨立站", type: "timer", s: ["2min", "監督2min", "30s", "嘗試30s", "無法30s"] },
        { id: 3, name: "獨立坐", type: "timer", s: ["2min", "監督2min", "30s", "10s", "無法10s"] },
        { id: 4, name: "站到坐", type: "click", s: ["流暢", "手控下降", "腿貼椅", "獨立不穩", "協助"] },
        { id: 5, name: "床椅轉移", type: "click", s: ["安全獨立", "手輔助", "監督下", "一人協助", "二人協助"] },
        { id: 6, name: "閉眼站", type: "timer", s: ["10s", "監督10s", "3s", "<3s", "跌倒"] },
        { id: 7, name: "併攏站", type: "timer", s: ["1min", "監督1min", "30s", "15s", "無法15s"] },
        { id: 8, name: "站立前伸", type: "click", s: [">25cm", ">12.5cm", ">5cm", "監督", "平衡失控"] },
        { id: 9, name: "地面拾物", type: "click", s: ["輕鬆", "監督下", "2-5cm", "無法", "失平衡"] },
        { id: 10, name: "回頭向後看", type: "click", s: ["雙側", "單側", "僅側面", "監督", "需協助"] },
        { id: 11, name: "360度轉圈", type: "timer", s: ["雙向4s", "單向4s", "安全慢", "監督", "協助"] },
        { id: 12, name: "階梯踏步", type: "timer", s: ["20s 8步", ">20s 8步", "4步", ">2步", "協助"] },
        { id: 13, name: "直線站立", type: "timer", s: ["30s", "跨前30s", "小步30s", "15s", "失衡"] },
        { id: 14, name: "單腳站立", type: "timer", compare: true, s: [">10s", "5-10s", "3s+", "<3s", "協助"] }
    ];

    let currentS = {}; let currentT = {}; let runT = {}; let legD = { L: 0, R: 0 };

    function init() {
        const wrap = document.getElementById('dynamic-items');
        bbsData.forEach((item, idx) => {
            const secNum = idx + 2;
            const div = document.createElement('div');
            div.className = 'section'; div.id = `sec-${secNum}`;
            let specialUI = item.compare ? `<div style="display:flex; gap:5px; margin-bottom:10px;"><button class="side-btn" id="l-leg" onclick="markL('L')">記左腳</button><button class="side-btn" id="r-leg" onclick="markL('R')">記右腳</button></div>` : "";
            
            div.innerHTML = `
                <h2>${idx+1}. ${item.name}</h2>
                <div class="eval-ui">
                    ${item.type === 'timer' || item.compare ? `<div class="timer-display" id="disp-${item.id}">0.0s</div><button class="btn" onclick="startC(${item.id})">計時</button> <button class="btn" onclick="stopC(${item.id})">停止</button>` : ''}
                    ${specialUI}
                </div>
                <div class="score-group">
                    ${[4,3,2,1,0].map(val => `<button class="score-btn" id="btn-${item.id}-${val}" onclick="recS(${item.id}, ${val})">${val}分：${item.s[4-val]}</button>`).join('')}
                </div>
                <div class="nav-btns">
                    <button class="btn" onclick="go(${secNum-1})">上一步</button>
                    <button class="btn btn-next" onclick="${secNum === 15 ? 'showR()' : 'go('+(secNum+1)+')'}">下一步</button>
                </div>`;
            wrap.appendChild(div);
        });
    }

    function go(n) { document.querySelectorAll('.section').forEach(s => s.classList.remove('active')); document.getElementById(`sec-${n}`).classList.add('active'); }
    function startC(id) { const s = Date.now(); if(runT[id]) clearInterval(runT[id]); runT[id] = setInterval(() => { const t = ((Date.now()-s)/1000).toFixed(1); document.getElementById(`disp-${id}`).innerText = t+'s'; currentT[id] = t+'s'; }, 100); }
    function stopC(id) { clearInterval(runT[id]); }
    function markL(s) { const t = parseFloat(document.getElementById('disp-14').innerText); legD[s] = t; document.getElementById(s === 'L' ? 'l-leg' : 'r-leg').innerText = (s==='L'?'左':'右')+t+"s"; currentT[14] = `L${legD.L}s/R${legD.R}s`; }
    function recS(id, v) { currentS[id] = v; document.querySelectorAll(`#sec-${id+1} .score-btn`).forEach(b => b.classList.remove('active')); document.getElementById(`btn-${id}-${v}`).classList.add('active'); }

    function showR() {
        let tot = 0; const pId = document.getElementById('p_id').value;
        bbsData.forEach(item => tot += (currentS[item.id] || 0));
        document.getElementById('res_total').innerText = tot;
        const b = document.getElementById('res_risk');
        b.innerText = tot >= 41 ? "低風險" : (tot >= 21 ? "中風險" : "高風險");
        b.style.background = tot >= 41 ? 'var(--low)' : (tot >= 21 ? 'var(--mid)' : 'var(--high)');
        
        const history = JSON.parse(localStorage.getItem('bbs_v4') || '[]');
        const last = history.reverse().find(r => r.id === pId);
        document.getElementById('compare_res').innerHTML = last ? `上次紀錄 (${last.date}): ${last.score}分<br>對比本次：${tot >= last.score ? '進步' : '退步'} ${Math.abs(tot - last.score)}分` : "無歷史紀錄";
        go(16);
    }

    function sendEmail() {
        const email = document.getElementById('target_email').value;
        const tot = document.getElementById('res_total').innerText;
        let detail = "";
        bbsData.forEach(item => {
            let t = currentT[item.id] ? `/` + currentT[item.id] : "";
            detail += `${item.id}.${currentS[item.id] || 0}${t}, `;
        });
        const body = `BBS結果: ${tot} (${detail.slice(0, -2)})`;
        window.location.href = `mailto:${email}?subject=BBS報告_${document.getElementById('p_id').value}&body=${encodeURIComponent(body)}`;
    }

    function saveAndReset() {
        const pId = document.getElementById('p_id').value || "Unknown";
        let history = JSON.parse(localStorage.getItem('bbs_v4') || '[]');
        history.push({ date: new Date().toLocaleDateString(), id: pId, score: parseInt(document.getElementById('res_total').innerText) });
        localStorage.setItem('bbs_v4', JSON.stringify(history));
        location.reload();
    }

    init();
</script>
</body>
</html>
