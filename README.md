<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>贝塞尔曲线生成器</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #0c0e1a 0%, #1a1c2e 100%);
            color: #e0e0e0;
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 40px 20px;
        }
        h1 {
            font-size: 2rem;
            margin-bottom: 10px;
            background: linear-gradient(90deg, #00d2ff, #7a5cff);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            background-clip: text;
        }
        .subtitle { color: #888; margin-bottom: 30px; font-size: 0.95rem; }
        .canvas-container {
            position: relative;
            width: 600px;
            height: 400px;
            background: rgba(255, 255, 255, 0.03);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 16px;
            overflow: hidden;
            box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
        }
        canvas { display: block; cursor: crosshair; }
        .point {
            position: absolute;
            width: 16px;
            height: 16px;
            border-radius: 50%;
            transform: translate(-50%, -50%);
            cursor: grab;
            transition: box-shadow 0.2s;
        }
        .point:hover { box-shadow: 0 0 12px rgba(255, 255, 255, 0.5); }
        .point:active { cursor: grabbing; }
        .point.start { background: #00d2ff; }
        .point.control1 { background: #ff6b6b; }
        .point.control2 { background: #4ecdc4; }
        .point.end { background: #ffe66d; }
        .label {
            position: absolute;
            top: -22px; left: 50%;
            transform: translateX(-50%);
            font-size: 11px; white-space: nowrap; color: #aaa;
        }
        .controls {
            margin-top: 24px;
            display: flex; gap: 16px;
            flex-wrap: wrap; justify-content: center;
        }
        .btn {
            padding: 10px 20px; border: none; border-radius: 8px;
            font-size: 0.9rem; cursor: pointer;
            transition: all 0.2s; font-weight: 500;
        }
        .btn-primary {
            background: linear-gradient(90deg, #00d2ff, #7a5cff); color: #fff;
        }
        .btn-primary:hover {
            transform: translateY(-2px);
            box-shadow: 0 4px 16px rgba(122, 92, 255, 0.4);
        }
        .btn-secondary {
            background: rgba(255, 255, 255, 0.08); color: #e0e0e0;
            border: 1px solid rgba(255, 255, 255, 0.15);
        }
        .btn-secondary:hover { background: rgba(255, 255, 255, 0.12); }
        .code-output {
            margin-top: 20px; width: 600px; max-width: 100%;
            background: rgba(0, 0, 0, 0.3);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 12px; padding: 16px;
            font-family: 'Courier New', monospace;
            font-size: 0.85rem; color: #a8e6cf;
            word-break: break-all; min-height: 50px; position: relative;
        }
        .code-label {
            position: absolute; top: -10px; left: 16px;
            background: #1a1c2e; padding: 0 8px;
            font-size: 0.75rem; color: #7a5cff;
        }
        .toast {
            position: fixed; bottom: 30px; left: 50%;
            transform: translateX(-50%) translateY(100px);
            background: rgba(0, 210, 255, 0.9); color: #000;
            padding: 10px 24px; border-radius: 8px;
            font-weight: 600; opacity: 0;
            transition: all 0.3s; pointer-events: none;
        }
        .toast.show {
            transform: translateX(-50%) translateY(0); opacity: 1;
        }
        @media (max-width: 640px) {
            .canvas-container, .code-output { width: 100%; }
            h1 { font-size: 1.5rem; }
        }
    </style>
</head>
<body>
    <h1>贝塞尔曲线生成器</h1>
    <p class="subtitle">拖拽控制点，实时生成三次贝塞尔曲线</p>
    <div class="canvas-container" id="canvasContainer">
        <canvas id="bezierCanvas"></canvas>
        <div class="point start" id="p0"><span class="label">起点 P0</span></div>
        <div class="point control1" id="p1"><span class="label">控制点 P1</span></div>
        <div class="point control2" id="p2"><span class="label">控制点 P2</span></div>
        <div class="point end" id="p3"><span class="label">终点 P3</span></div>
    </div>
    <div class="controls">
        <button class="btn btn-primary" id="copyBtn">复制 SVG 路径</button>
        <button class="btn btn-secondary" id="resetBtn">重置</button>
        <button class="btn btn-secondary" id="animateBtn">播放动画</button>
    </div>
    <div class="code-output" id="codeOutput">
        <span class="code-label">SVG Path</span>
        <span id="pathText"></span>
    </div>
    <div class="toast" id="toast">已复制到剪贴板！</div>
    <script>
        const canvas = document.getElementById('bezierCanvas');
        const ctx = canvas.getContext('2d');
        const container = document.getElementById('canvasContainer');
        const pathText = document.getElementById('pathText');
        const W = 600;
        const H = 400;
        canvas.width = W;
        canvas.height = H;
        let points = [
            { x: 80, y: 320, color: '#00d2ff' },
            { x: 160, y: 60, color: '#ff6b6b' },
            { x: 440, y: 340, color: '#4ecdc4' },
            { x: 520, y: 80, color: '#ffe66d' }
        ];
        let dragging = null;
        let animating = false;
        function updateDOMPoints() {
            const ids = ['p0', 'p1', 'p2', 'p3'];
            points.forEach((p, i) => {
                const el = document.getElementById(ids[i]);
                el.style.left = p.x + 'px';
                el.style.top = p.y + 'px';
            });
        }
        function drawGrid() {
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.04)';
            ctx.lineWidth = 1;
            for (let x = 0; x <= W; x += 40) {
                ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke();
            }
            for (let y = 0; y <= H; y += 40) {
                ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke();
            }
        }
        function bezierPoint(t, p0, p1, p2, p3) {
            const u = 1 - t;
            return u * u * u * p0 + 3 * u * u * t * p1 + 3 * u * t * t * p2 + t * t * t * p3;
        }
        function drawCurve(progress = 1) {
            const [p0, p1, p2, p3] = points;
            ctx.setLineDash([6, 4]);
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.15)';
            ctx.lineWidth = 1;
            ctx.beginPath();
            ctx.moveTo(p0.x, p0.y);
            ctx.lineTo(p1.x, p1.y);
            ctx.lineTo(p2.
