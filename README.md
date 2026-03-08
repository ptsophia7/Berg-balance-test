<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BBS 專業評估 (含導出功能)</title>
    <style>
        :root { --primary: #2c3e50; --secondary: #3498db; --low: #27ae60; --mid: #f39c12; --high: #e74c3c; }
        body { font-family: -apple-system, "Microsoft JhengHei", sans-serif; background: #f0f2f5; margin: 0; padding: 10px; display: flex; justify-content: center; }
        .app-container { width: 100%; max-width: 550px; }
        .section { display: none; background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        .section.active { display: block; }
        h2 { color: var(--primary); font-size: 1.2rem; margin: 0 0 15px 0; border-bottom: 2px solid #eee; padding-bottom: 8px; }
        .progress { font-size: 0.8rem; color: #7f8c8d; margin-bottom: 5px; }
        .eval-ui { margin-bottom: 15px; text-align: center; }
        .timer-display { font-size: 2.2rem; font-weight: bold; color: var(--primary); margin: 5px 0; }
        .score-group { display: flex; flex-direction: column; gap: 8px; }
        .score-btn { width: 100%; padding: 12px; border: 1px solid #ddd; background: #fff; border-radius: 8px; text-align: left; cursor: pointer; font-size: 1rem; }
        .score-btn.active { background: var(--secondary); color: white; border-color: var(--secondary); }
        .side-selector { display: flex; gap: 5px; margin-bottom: 10px; }
        .side-btn { flex: 1; padding: 8px; border: 1px solid #ddd; border-radius: 5px; background: #f8f9fa; cursor: pointer; font-size: 0.9rem; }
        .side-btn.active { background: var(--primary); color: white; }
        .nav-btns { display: flex; gap: 10px; margin-top: 20px; flex-wrap: wrap; }
        .btn { flex: 1; padding: 12px; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; min-width: 120px; }
        .btn-next { background: var(--primary); color: white; }
        .btn-prev { background: #bdc3c7; color: white; }
        .btn-export { background: #27ae60; color: white; }
        .btn-email { background: #8e44ad; color: white; }
        .total-score { font-size: 3rem; font-weight: bold; text-align: center; margin: 5px 0; }
        .risk-banner { font-size: 1.3rem; padding: 12px; border-radius: 8px; color: white; text-align: center; font-weight: bold; margin-bottom: 15px; }
        table { width: 100%; font-size: 0.85rem; border-collapse: collapse; margin-bottom: 20px; }
        td, th { border: 1px solid #eee; padding: 8px; text-align: left; }
        th { background: #f8f9fa; }
    </style>
</head>
<body>

<div class="app-container">
    <div class="section active" id="sec-1">
        <h2>基本資料</h2>
        <div style="margin-bottom:15px;">
            <label>預設接收 Email</label>
            <input type="email" id="target_email" value="your-email@example.com" style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px;">
        </div>
        <div style="margin-bottom:15px;">
            <label>病歷號</label>
            <input type="text" id="p_id" style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px;">
        </div>
        <div>
            <label>病人類型</label>
            <select id="p_type" style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px;">
                <option>高齡者</option><option>中風</option><option>巴金森氏症</option>
            </select>
        </div>
        <div class="nav-btns"><button class="btn btn-next" onclick="go(2)">開始評估</button></div>
    </div>

    <div id="dynamic-items"></div>

    <div class="section" id="sec-16">
        <h2>評估結果報告</h2>
        <div id="res_p_info" style="font-size:0.9rem; margin-bottom:10px;"></div>
        <div id="res_risk" class="risk-banner"></div>
        <div class="total-score" id="res_total">0</div>
        <div style="text-align:center; color:#7f8c8d; margin-bottom:15px;">BBS 總積分</div>
        
        <table id="res_table">
            <thead><tr><th>項目</th><th>分</th><th>備註</th></tr></thead>
            <tbody></tbody>
        </table>

        <div class="nav-btns">
            <button class="btn btn-export" onclick="downloadCSV()">下載所有歷史 Excel (CSV)</button>
            <button class="btn btn-email" onclick="sendEmail()">發送此報告至 Email</button>
            <button class="btn btn-prev" onclick="go(15)">回上項修改</button>
            <button class="btn btn-next" style="background:#27ae60" onclick="saveAndReset()">存檔並重置</button>
        </div>
    </div>
</div>

<script>
    const bbsData = [
        { id: 1, name: "1. 從坐位站起", type: "click", s: ["獨立站起", "手撐獨立站起", "多次嘗試後站起", "少量協助", "大量協助"] },
        { id: 2, name: "2. 獨立站立", type: "timer", target: 120, s: ["穩定 2min", "監督下 2min", "獨立 30s", "嘗試後站 30s", "無法站立 30s"] },
        { id: 3, name: "3. 獨立坐著", type: "timer", target: 120, s: ["穩定坐 2min", "監督下 2min", "坐 30s", "坐 10s", "無法坐 10s"] },
        { id: 4, name: "4. 從站位坐下", type: "click", s: ["流暢少手撐", "需手控制下降", "腿貼椅下降", "獨立但不穩", "需協助"] },
        { id: 5, name: "5. 床椅轉移", type: "click", s: ["不需手安全轉移", "手輔助安全轉移", "需指導或監督", "一人協助", "二人協助"] },
        { id: 6, name: "6. 閉眼站立", type: "timer", target: 10, s: ["穩定 10s", "監督下 10s", "站立 3s", "閉眼無法維持 3s", "閉眼即跌倒"] },
        { id: 7, name: "7. 併攏站立", type: "timer", target: 60, s: ["獨立 1min", "監督 1min", "獨立 30s", "協助下站 15s", "無法站 15s"] },
        { id: 8, name: "8. 站立前伸", type: "click", s: ["> 25cm", "> 12.5cm", "> 5cm", "需監督", "失去平衡/跌倒"] },
        { id: 9, name: "9. 地面拾物", type: "click", s: ["輕鬆拾起", "獨立拾起(需監督)", "離物2-5cm並平衡", "無法拾起(需監督)", "失去平衡"] },
        { id: 10, name: "10. 回頭向後看", type: "click", s: ["雙側均可回頭", "僅單側可回頭", "僅轉向側面", "回頭需監督", "需協助"] },
        { id: 11, name: "11. 360度轉圈", type: "timer", target: 4, side: "LR", s: ["雙向 4s 內", "單向 4s 內", "安全但緩慢", "需監督/指導", "需協助"] },
        { id: 12, name: "12. 階梯踏步", type: "timer", target: 20, s: ["20s 內 8 步", "> 20s 8 步", "監督下 4 步", "協助下 > 2 步", "需協助"] },
        { id: 13, name: "13. 串聯站立", type: "timer", target: 30, foot: "LR", s: ["獨立 30s", "腳稍跨前 30s", "小踏步 30s", "協助下站 15s", "失去平衡"] },
        { id: 14, name: "14. 單腳站立", type: "timer", target: 10, compare: true, s: ["獨立 > 10s", "5-10s", "3s 以上", "< 3s 但可獨立", "需協助"] }
    ];

    let currentS = {}; let currentT = {}; let runT = {}; let legD = { L: 0, R: 0 };

    function init() {
        const wrap = document.getElementById('dynamic-items');
        bbsData.forEach((item, idx) => {
            const secNum = idx + 2;
            const div = document.createElement('div');
            div.className = 'section'; div.id = `sec-${secNum}`;
            let specialUI = "";
            if(item.side) specialUI = `<div class="side-selector"><button class="side-btn active">向左轉</button><button class="side-btn">向右轉</button></div>`;
            if(item.foot) specialUI = `<div class="side-selector"><button class="side-btn active">左腳在後</button><button class="side-btn">右腳在後</button></div>`;
            if(item.compare) specialUI = `<div class="side-selector"><button class="side-btn" id="l-leg" onclick="markL('L')">記錄左腳</button><button class="side-btn" id="r-leg" onclick="markL('R')">記錄右腳</button></div>`;
            
            div.innerHTML = `
                <div class="progress">項目評估 ${idx + 1} / 14</div>
                <h2>${item.name}</h2>
                <div class="eval-ui">
                    ${item.type === 'timer' ? `<div class="timer-display" id="disp-${item.id}">0.0s</div><button class="btn" style="background:#3498db; color:white;" onclick="startC(${item.id})">計時</button> <button class="btn" style="background:#e67e22; color:white;" onclick="stopC(${item.id})">停止</button>` : ''}
                    ${specialUI}
                </div>
                <div class="score-group">
                    ${[4,3,2,1,0].map(val => `<button class="score-btn" id="btn-${item.id}-${val}" onclick="recS(${item.id}, ${val})">${val}分：${item.s[4-val]}</button>`).join('')}
                </div>
                <div class="nav-btns">
                    <button class="btn btn-prev" onclick="go(${secNum-1})">上一步</button>
                    <button class="btn btn-next" onclick="${secNum === 15 ? 'showR()' : 'go('+(secNum+1)+')'}">下一步</button>
                </div>`;
            wrap.appendChild(div);
        });
    }

    function go(n) { document.querySelectorAll('.section').forEach(s => s.classList.remove('active')); document.getElementById(`sec-${n}`).classList.add('active'); window.scrollTo(0,0); }
    function markL(s) { const t = parseFloat(document.getElementById('disp-14').innerText); legD[s] = t; document.getElementById(s === 'L' ? 'l-leg' : 'r-leg').innerText = (s==='L'?'左':'右')+t+"s"; currentT[14] = `左${legD.L}s/右${legD.R}s`; }
    function startC(id) { const start = Date.now(); if(runT[id]) clearInterval(runT[id]); runT[id] = setInterval(() => { const t = ((Date.now() - start) / 1000).toFixed(1); document.getElementById(`disp-${id}`).innerText = t + 's'; currentT[id] = t + 's'; }, 100); }
    function stopC(id) { clearInterval(runT[id]); }
    function recS(id, v) { currentS[id] = v; document.querySelectorAll(`#sec-${id+1} .score-btn`).forEach(b => b.classList.remove('active')); document.getElementById(`btn-${id}-${v}`).classList.add('active'); }

    function showR() {
        let tot = 0; let tb = "";
        bbsData.forEach(item => { const v = currentS[item.id] || 0; tot += v; tb += `<tr><td>${item.id}. ${item.name.split('. ')[1]}</td><td>${v}</td><td>${currentT[item.id] || '-'}</td></tr>`; });
        document.getElementById('res_p_info').innerText = `病歷號：${document.getElementById('p_id').value || '未填'} | 類型：${document.getElementById('p_type').value}`;
        document.getElementById('res_total').innerText = tot;
        const b = document.getElementById('res_risk');
        if (tot >= 41) { b.innerText = "低跌倒風險"; b.style.background = 'var(--low)'; }
        else if (tot >= 21) { b.innerText = "中度跌倒風險"; b.style.background = 'var(--mid)'; }
        else { b.innerText = "高跌倒風險"; b.style.background = 'var(--high)'; }
        document.querySelector('#res_table tbody').innerHTML = tb;
        go(16);
    }

    // 儲存至本地並導出 CSV
    function saveAndReset() {
        const pId = document.getElementById('p_id').value || "Unknown";
        const tot = document.getElementById('res_total').innerText;
        const risk = document.getElementById('res_risk').innerText;
        const record = { date: new Date().toLocaleString(), id: pId, score: tot, risk: risk };
        
        let history = JSON.parse(localStorage.getItem('bbs_v2_history') || '[]');
        history.push(record);
        localStorage.setItem('bbs_v2_history', JSON.stringify(history));
        
        alert("評估已存檔！");
        location.reload();
    }

    function downloadCSV() {
        const history = JSON.parse(localStorage.getItem('bbs_v2_history') || '[]');
        if (history.length === 0) { alert("目前沒有歷史紀錄可匯出"); return; }
        
        let csvContent = "\uFEFF日期,病歷號,總分,風險\n";
        history.forEach(r => { csvContent += `${r.date},${r.id},${r.score},${r.risk}\n`; });
        
        const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
        const link = document.createElement("a");
        link.href = URL.createObjectURL(blob);
        link.setAttribute("download", `BBS_History_${new Date().toLocaleDateString()}.csv`);
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    }

    function sendEmail() {
        const email = document.getElementById('target_email').value;
        const pId = document.getElementById('p_id').value;
        const tot = document.getElementById('res_total').innerText;
        const risk = document.getElementById('res_risk').innerText;
        
        let body = `BBS 評估報告\n`;
        body += `病歷號: ${pId}\n`;
        body += `總分: ${tot} / 56\n`;
        body += `風險等級: ${risk}\n\n`;
        body += `詳細記錄:\n`;
        bbsData.forEach(item => {
            body += `${item.id}. ${item.name.split('. ')[1]}: ${currentS[item.id] || 0}分 (${currentT[item.id] || '-'})\n`;
        });

        const mailtoLink = `mailto:${email}?subject=BBS評估報告_${pId}&body=${encodeURIComponent(body)}`;
        window.location.href = mailtoLink;
    }

    init();
</script>
</body>
</html>
