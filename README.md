<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>heat • pulse</title>
  <style>
    * { margin: 0; padding: 0; }
    body {
      background: #0d0f1a;
      min-height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      font-family: system-ui, sans-serif;
    }
    .card {
      background: rgba(20, 28, 50, 0.7);
      backdrop-filter: blur(4px);
      padding: 1.8rem;
      border-radius: 2.8rem;
      box-shadow: 0 20px 40px -10px #00000080, 0 0 0 1px #ffb07c30;
    }
    .heat-wrap {
      position: relative;
      width: 460px;
      max-width: 82vw;
      aspect-ratio: 1/1;
      border-radius: 2.2rem;
      overflow: hidden;
      background: #0b0e17;
      box-shadow: inset 0 0 80px #ff6a1a20;
    }
    canvas {
      display: block;
      width: 100%;
      height: 100%;
      cursor: crosshair;
    }
    .footer {
      display: flex;
      justify-content: space-between;
      margin-top: 0.9rem;
      color: #ffc8a0aa;
      font-size: 0.75rem;
      letter-spacing: 0.5px;
      text-transform: uppercase;
      padding: 0 0.3rem;
    }
    .footer span {
      background: #ffb07c18;
      padding: 0.25rem 1.2rem;
      border-radius: 40px;
      border: 1px solid #ffb07c22;
      backdrop-filter: blur(4px);
      animation: softPulse 2.4s infinite alternate;
    }
    @keyframes softPulse {
      0% { opacity: 0.6; text-shadow: 0 0 4px #ff9f4a; }
      100% { opacity: 1; text-shadow: 0 0 16px #ff6a1a, 0 0 40px #ff3d00; }
    }
  </style>
</head>
<body>
<div class="card">
  <div class="heat-wrap">
    <canvas id="heatCanvas" width="600" height="600"></canvas>
  </div>
  <div class="footer">
    <span>🔥 heat·grid</span>
    <span>✦ animated ✦</span>
  </div>
</div>

<script>
  (function() {
    const canvas = document.getElementById('heatCanvas');
    const ctx = canvas.getContext('2d');
    const W = 600, H = 600;

    // ---------- generate random heat data (12x12 grid) ----------
    const COLS = 14, ROWS = 14;
    const cellW = W / COLS, cellH = H / ROWS;
    let heatData = [];

    function generateData() {
      heatData = [];
      for (let r = 0; r < ROWS; r++) {
        const row = [];
        for (let c = 0; c < COLS; c++) {
          // random heat 0..1, with a hot center bias
          const dist = Math.hypot(c - COLS/2, r - ROWS/2) / (Math.min(COLS, ROWS)/2);
          const base = 0.2 + 0.8 * (1 - Math.min(1, dist));
          const variation = 0.25 * (Math.random() - 0.5);
          row.push(Math.min(1, Math.max(0, base + variation)));
        }
        heatData.push(row);
      }
    }
    generateData();

    // ---------- color mapping (cool → hot) ----------
    function heatColor(t) {
      t = Math.min(1, Math.max(0, t));
      const stops = [
        [0,  20,  30,  80],   // deep blue
        [0.3, 70,  40, 130],  // violet
        [0.55, 200, 60,  40], // orange-red
        [0.8, 255, 160, 40],  // gold
        [1,  255, 240, 200]   // white-hot
      ];
      for (let i = 0; i < stops.length - 1; i++) {
        const [p1, r1, g1, b1] = stops[i];
        const [p2, r2, g2, b2] = stops[i+1];
        if (t >= p1 && t <= p2) {
          const f = (t - p1) / (p2 - p1);
          return [
            Math.round(r1 + (r2 - r1) * f),
            Math.round(g1 + (g2 - g1) * f),
            Math.round(b1 + (b2 - b1) * f)
          ];
        }
      }
      return [255, 240, 200];
    }

    // ---------- animation state ----------
    let time = 0;
    let mouseX = W/2, mouseY = H/2;
    let mouseInside = false;

    // mouse / touch interaction
    function updateMouse(e) {
      const rect = canvas.getBoundingClientRect();
      const scaleX = canvas.width / rect.width;
      const scaleY = canvas.height / rect.height;
      const cx = (e.clientX - rect.left) * scaleX;
      const cy = (e.clientY - rect.top) * scaleY;
      mouseX = Math.min(W-1, Math.max(0, cx));
      mouseY = Math.min(H-1, Math.max(0, cy));
      mouseInside = true;
    }
    canvas.addEventListener('mousemove', updateMouse);
    canvas.addEventListener('mouseleave', () => mouseInside = false);
    canvas.addEventListener('touchmove', (e) => {
      e.preventDefault();
      const t = e.touches[0];
      if (!t) return;
      const rect = canvas.getBoundingClientRect();
      const scaleX = canvas.width / rect.width;
      const scaleY = canvas.height / rect.height;
      mouseX = Math.min(W-1, Math.max(0, (t.clientX - rect.left) * scaleX));
      mouseY = Math.min(W-1, Math.max(0, (t.clientY - rect.top) * scaleY));
      mouseInside = true;
    }, { passive: false });
    canvas.addEventListener('touchend', () => mouseInside = false);

    // ---------- draw loop ----------
    function draw() {
      time += 0.025;
      ctx.clearRect(0, 0, W, H);

      // ---- 1. background glow ----
      const bgGrad = ctx.createRadialGradient(W/2, H/2, 20, W/2, H/2, 350);
      bgGrad.addColorStop(0, '#1f1733');
      bgGrad.addColorStop(0.6, '#0f121f');
      bgGrad.addColorStop(1, '#06080d');
      ctx.fillStyle = bgGrad;
      ctx.fillRect(0, 0, W, H);

      // ---- 2. draw heat cells ----
      for (let r = 0; r < ROWS; r++) {
        for (let c = 0; c < COLS; c++) {
          let heat = heatData[r][c];

          // mouse influence: increase heat near cursor
          if (mouseInside) {
            const cx = c * cellW + cellW/2;
            const cy = r * cellH + cellH/2;
            const dist = Math.hypot(mouseX - cx, mouseY - cy);
            if (dist < 120) {
              heat = Math.min(1, heat + 0.25 * (1 - dist/120));
            }
          }

          // animated pulse (wave)
          const pulse = 0.9 + 0.1 * Math.sin(time * 0.9 + r * 0.8 + c * 0.6);
          const finalHeat = Math.min(1, heat * pulse);

          const [rCol, gCol, bCol] = heatColor(finalHeat);

          // cell rectangle with rounded corners
          const x = c * cellW, y = r * cellH;
          const pad = 2;
          const radius = 6;
          ctx.beginPath();
          ctx.roundRect(x + pad, y + pad, cellW - pad*2, cellH - pad*2, radius);
          ctx.fillStyle = `rgba(${rCol}, ${gCol}, ${bCol}, ${0.6 + 0.4 * finalHeat})`;
          ctx.fill();

          // glow overlay
          if (finalHeat > 0.4) {
            ctx.shadowColor = `rgba(${rCol}, ${gCol}, ${bCol}, 0.3)`;
            ctx.shadowBlur = 20 * finalHeat;
            ctx.fill();
            ctx.shadowBlur = 0;
          }
        }
      }

      // ---- 3. additional animated glow ring ----
      const gradRing = ctx.createRadialGradient(W/2, H/2, 30, W/2, H/2, 280);
      gradRing.addColorStop(0, 'rgba(255,120,40,0)');
      gradRing.addColorStop(0.7, `rgba(255,120,40,${0.06 + 0.04 * Math.sin(time*0.3)})`);
      gradRing.addColorStop(1, 'rgba(255,60,20,0)');
      ctx.fillStyle = gradRing;
      ctx.fillRect(0, 0, W, H);

      // ---- 4. vignette ----
      const vig = ctx.createRadialGradient(W/2, H/2, W*0.2, W/2, H/2, W*0.75);
      vig.addColorStop(0, 'rgba(0,0,0,0)');
      vig.addColorStop(1, 'rgba(0,0,0,0.45)');
      ctx.fillStyle = vig;
      ctx.fillRect(0, 0, W, H);

      requestAnimationFrame(draw);
    }

    // helper: roundRect
    CanvasRenderingContext2D.prototype.roundRect = function(x, y, w, h, r) {
      if (w < 2 * r) r = w / 2;
      if (h < 2 * r) r = h / 2;
      this.moveTo(x + r, y);
      this.lineTo(x + w - r, y);
      this.quadraticCurveTo(x + w, y, x + w, y + r);
      this.lineTo(x + w, y + h - r);
      this.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
      this.lineTo(x + r, y + h);
      this.quadraticCurveTo(x, y + h, x, y + h - r);
      this.lineTo(x, y + r);
      this.quadraticCurveTo(x, y, x + r, y);
      this.closePath();
      return this;
    };

    draw();
  })();
</script>
</body>
</html>
