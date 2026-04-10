<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <title>ERa Robot Control</title>
    <script src="https://www.unpkg.com/@eohjsc/era-widget@1.1.3/src/index.js"></script>
    <style>
        body { font-family: sans-serif; text-align: center; background: #0a0a0a; color: #fff; margin: 0; overflow: hidden; touch-action: none; }
        .header { background: #1a1a1a; padding: 15px; font-size: 20px; color: #00e5ff; border-bottom: 2px solid #00e5ff; text-transform: uppercase; }
        #joy-outer { position: relative; width: 250px; height: 250px; margin: 20px auto; background: #111; border-radius: 50%; border: 4px solid #333; display: flex; align-items: center; justify-content: center; box-shadow: 0 0 15px rgba(0,229,255,0.1); }
        .ctrl-group { margin: 15px; display: flex; justify-content: center; gap: 15px; }
        .btn { padding: 12px 20px; font-size: 14px; font-weight: bold; border-radius: 25px; border: none; cursor: pointer; color: white; transition: 0.2s; }
        .btn:active { transform: scale(0.9); }
        .btn-auto { background: #00c853; } 
        .btn-man { background: #ff1744; }
        .stat-box { background: #161616; padding: 10px 20px; border-radius: 10px; border: 1px solid #333; display: inline-block; font-size: 14px; }
        b { color: #00e5ff; }
    </style>
</head>
<body>

    <div class="header">ERa Robot Control</div>

    <div id="joy-outer">
        <canvas id="joystick"></canvas>
    </div>

    <div class="ctrl-group">
        <button id="btnAuto" class="btn btn-auto">Dò Line</button>
        <button id="btnManual" class="btn btn-man">Dừng Tay</button>
    </div>

    <div class="stat-box">
        Mode: <b id="m_t">--</b> | Sensor: <b id="d_t">--</b> cm
    </div>

    <script>
        // 1. Khởi tạo các biến ERa
        const eraWidget = new EraWidget();
        let configDist = null, configMode = null, actions = [];
        
        const btnAuto = document.getElementById('btnAuto');
        const btnManual = document.getElementById('btnManual');
        const distLabel = document.getElementById('d_t');
        const modeLabel = document.getElementById('m_t');

        eraWidget.init({
            needRealtimeConfigs: true,
            needActions: true,
            maxRealtimeConfigsCount: 2, // 1 cho Sensor, 1 cho Mode
            maxActionsCount: 4,         // Auto, Manual, Joystick, Stop
            onConfiguration: (configuration) => {
                configDist = configuration.realtime_configs[0];
                configMode = configuration.realtime_configs[1];
                actions = configuration.actions;
            },
            onValues: (values) => {
                // Cập nhật khoảng cách từ cảm biến
                if (configDist && values[configDist.id]) {
                    distLabel.innerText = values[configDist.id].value;
                }
                // Cập nhật chế độ hiện tại
                if (configMode && values[configMode.id]) {
                    modeLabel.innerText = values[configMode.id].value == 1 ? "AUTO" : "MANUAL";
                }
            }
        });

        // 2. Xử lý nút nhấn Gửi lệnh về ERa
        btnAuto.addEventListener('click', () => {
            // Giả sử Action 0 là lệnh AUTO
            if (actions[0]) eraWidget.triggerAction(actions[0].action, null);
        });

        btnManual.addEventListener('click', () => {
            // Giả sử Action 1 là lệnh MANUAL
            if (actions[1]) eraWidget.triggerAction(actions[1].action, null);
        });

        // 3. Logic Joystick
        var canvas = document.getElementById('joystick');
        var ctx = canvas.getContext('2d');
        var s = 250; canvas.width = s; canvas.height = s;
        var cx = s/2, cy = s/2, ro = 80, ri = 40;
        var x = cx, y = cy, active = false;

        function draw() {
            ctx.clearRect(0,0,s,s);
            ctx.beginPath(); ctx.arc(cx, cy, ro, 0, Math.PI*2); ctx.strokeStyle="#333"; ctx.lineWidth=3; ctx.stroke();
            ctx.beginPath(); ctx.arc(x, y, ri, 0, Math.PI*2); ctx.fillStyle="#00e5ff"; ctx.fill();
        }
        draw();

        function handle(e) {
            if(!active) return;
            var r = canvas.getBoundingClientRect();
            var t = e.touches ? e.touches[0] : e;
            var dx = (t.clientX - r.left) - cx, dy = (t.clientY - r.top) - cy;
            var d = Math.sqrt(dx*dx + dy*dy);
            if(d > ro) { dx *= ro/d; dy *= ro/d; }
            x = cx + dx; y = cy + dy; draw();

            let angle = Math.round(Math.atan2(-dy, dx) * 180 / Math.PI);
            if(angle < 0) angle += 360;
            let speed = Math.round(Math.min(d, ro) * 2.6);

            // Gửi dữ liệu Joystick qua Action (Giả sử Action 2 xử lý di chuyển)
            // Truyền giá trị speed và angle dưới dạng param nếu ERa hỗ trợ hoặc qua Virtual Pin
            // Ở đây ví dụ gửi lệnh di chuyển đơn giản:
            if(actions[2]) eraWidget.triggerAction(actions[2].action, speed); 
        }

        canvas.onmousedown = () => active = true;
        window.onmouseup = () => { 
            active = false; x=cx; y=cy; draw(); 
            if(actions[3]) eraWidget.triggerAction(actions[3].action, null); // Lệnh STOP
        };
        canvas.ontouchstart = (e) => { active = true; e.preventDefault(); };
        window.ontouchend = () => { active = false; x=cx; y=cy; draw(); if(actions[3]) eraWidget.triggerAction(actions[3].action, null); };
        window.onmousemove = handle; window.ontouchmove = handle;
    </script>
</body>
</html>
