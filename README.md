<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Heat • pulse</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    body {
      background: #0b0e1a;
      min-height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      font-family: system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
    }
    .card {
      background: rgba(18, 25, 45, 0.65);
      backdrop-filter: blur(6px);
      border-radius: 3rem;
      padding: 1.8rem 2rem 2.2rem;
      box-shadow: 0 25px 45px -10px rgba(0, 0, 0, 0.7), 0 0 0 1px rgba(255, 215, 100, 0.15);
      transition: 0.3s ease;
    }
    .heat-container {
      position: relative;
      width: 420px;
      max-width: 80vw;
      aspect-ratio: 1 / 1;
      border-radius: 2.5rem;
      overflow: hidden;
      box-shadow: inset 0 0 60px rgba(255, 120, 20, 0.2);
      background: #0f1422;
    }
    canvas {
      display: block;
      width: 100%;
      height: 100%;
      cursor: crosshair;
      transition: filter 0.2s;
    }
    canvas:active {
      filter: drop-shadow(0 0 12px #ff7f2a);
    }
    .badge {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-top: 1rem;
      color: rgba(255, 215, 180, 0.7);
      font-weight: 400;
      letter-spacing: 0.3px;
      font-size: 0.8rem;
      text-transform: uppercase;
      padding: 0 0.2rem;
    }
    .badge span {
      background: rgba(255, 180, 80, 0.12);
      padding: 0.35rem 1rem;
      border-radius: 40px;
      border: 1px solid rgba(255, 180, 80, 0.2);
      backdrop-filter: blur(4px);
    }
    .badge i {
      font-style: normal;
      display: inline-block;
      animation: pulseGlow 2.2s infinite alternate;
    }
    @keyframes pulseGlow {
      0% { opacity: 0.5; text-shadow: 0 0 2px #ff9f4a; }
      100% { opacity: 1; text-shadow: 0 0 12px #ff7f2a, 0 0 30px #ff4f1a; }
    }
    /* tiny responsive */
    @media (max-width: 500px) {
      .card { padding: 1rem; border-radius: 2rem; }
      .heat-container { border-radius: 1.8rem; }
    }
  </style>
</head>
<body>
<div class="card">
  <div class="heat-container">
    <canvas id="heatCanvas" width="600" height="600"></canvas>
  </div>
  <div class="badge">
    <span>🔥 <i>heat·pulse</i></span>
    <span>✦ animated ✦</span>
  </div>
</div>
<script>
  (function() {
    const canvas = document.getElementById('heatCanvas');
    const ctx = canvas.getContext('2d');

    // dimensions
    const w = 600, h = 600;
    canvas.width = w; canvas.height = h;

    // ---------- heat particles ----------
    const COUNT = 220;
    const particles = [];

    // color stops for heat (cold → hot)
    const hotColors = [
      { pos: 0.0, r: 20, g: 30, b: 70 },     // deep cool
      { pos: 0.25, r: 80, g: 40, b: 120 },    // violet
      { pos: 0.5, r: 220, g: 70, b: 40 },     // orange-red
      { pos: 0.75, r: 255, g: 180, b: 40 },   // gold
      { pos: 1.0, r: 255, g: 240, b: 180 }    // white-hot
    ];

    function lerpColor(t) {
      t = Math.min(1, Math.max(0, t));
      for (let i = 0; i < hotColors.length - 1; i++) {
        const a = hotColors[i];
        const b = hotColors[i+1];
        if (t >= a.pos && t <= b.pos) {
          const range = b.pos - a.pos;
          const frac = range === 0 ? 0 : (t - a.pos) / range;
          return {
            r: Math.floor(a.r + (b.r - a.r) * frac),
            g: Math.floor(a.g + (b.g - a.g) * frac),
            b: Math.floor(a.b + (b.b - a.b) * frac)
          };
        }
      }
      return { r: 255, g: 240, b: 180 };
    }

    // particle factory
    function createParticle(index) {
      const angle = Math.random() * 2 * Math.PI;
      const radius = 40 + Math.random() * 220;  // spread
      return {
        // polar-ish coords, but we map to x,y
        baseX: w/2 + Math.cos(angle) * radius,
        baseY: h/2 + Math.sin(angle) * radius * 0.8, // slight oval
        offsetX: (Math.random() - 0.5) * 70,
        offsetY: (Math.random() - 0.5) * 70,
        size: 6 + Math.random() * 26,
        speed: 0.008 + Math.random() * 0.025,
        phase: Math.random() * 100,
        drift: 0.2 + Math.random() * 0.5,
        // heat intensity (0..1) with variation
        heat: 0.3 + Math.random() * 0.7,
        // for extra shimmer
        twinkle: Math.random() * 0.5 + 0.5,
      };
    }

    for (let i = 0; i < COUNT; i++) {
      particles.push(createParticle(i));
    }

    // mouse / pointer interaction – adds a "hot spot" 
    let mouseX = w/2, mouseY = h/2;
    let mouseIn = false;

    function handleMove(e) {
      const rect = canvas.getBoundingClientRect();
      const scaleX = canvas.width / rect.width;
      const scaleY = canvas.height / rect.height;
      const rawX = (e.clientX - rect.left) * scaleX;
      const rawY = (e.clientY - rect.top) * scaleY;
      mouseX = Math.min(w-1, Math.max(0, rawX));
      mouseY = Math.min(h-1, Math.max(0, rawY));
      mouseIn = true;
    }

    function handleLeave() {
      mouseIn = false;
    }

    canvas.addEventListener('mousemove', handleMove);
    canvas.addEventListener('mouseleave', handleLeave);
    // touch support
    canvas.addEventListener('touchmove', (e) => {
      e.preventDefault();
      const touch = e.touches[0];
      if (!touch) return;
      const rect = canvas.getBoundingClientRect();
      const scaleX = canvas.width / rect.width;
      const scaleY = canvas.height / rect.height;
      mouseX = Math.min(w-1, Math.max(0, (touch.clientX - rect.left) * scaleX));
      mouseY = Math.min(h-1, Math.max(0, (touch.clientY - rect.top) * scaleY));
      mouseIn = true;
    }, { passive: false });
    canvas.addEventListener('touchend', () => { mouseIn = false; });

    // animation loop
    let time = 0;

    function drawHeatmap() {
      time += 0.02;
      ctx.clearRect(0, 0, w, h);

      // ---- 1. draw soft radial background (ambient heat glow) ----
      const grad = ctx.createRadialGradient(w/2, h/2, 10, w/2, h/2, 320);
      grad.addColorStop(0, '#2b1a3a');
      grad.addColorStop(0.3, '#1f132e');
      grad.addColorStop(0.8, '#0b0e1a');
      ctx.fillStyle = grad;
      ctx.fillRect(0, 0, w, h);

      // ---- 2. heat particles (with animation + mouse influence) ----
      for (let p of particles) {
        // animated offset (orbit + wave)
        const waveX = Math.sin(time * p.speed + p.phase) * 18;
        const waveY = Math.cos(time * p.speed * 0.7 + p.phase * 1.2) * 18;

        // base position + animation
        let x = p.baseX + p.offsetX + waveX;
        let y = p.baseY + p.offsetY + waveY;

        // mouse attraction / repulsion (hot spot)
        if (mouseIn) {
          const dx = mouseX - x;
          const dy = mouseY - y;
          const dist = Math.hypot(dx, dy);
          if (dist < 150 && dist > 1) {
            const force = 0.08 * (1 - dist / 150);
            x += dx * force * 1.2;
            y += dy * force * 1.2;
            // also increase heat near mouse
            p.heat = Math.min(1, p.heat + 0.03 * (1 - dist / 150));
          } else {
            // slowly cool back
            p.heat = Math.max(0.15, p.heat - 0.0015);
          }
        } else {
          // natural drift back to base heat
          p.heat = Math.max(0.2, p.heat - 0.0008);
        }

        // clamp heat
        p.heat = Math.min(1, Math.max(0.05, p.heat));

        // size variation with heat + pulse
        const pulseSize = 1 + 0.2 * Math.sin(time * 1.7 + p.phase);
        const size = p.size * (0.6 + 0.4 * p.heat) * pulseSize;

        // ---- color from heat ----
        const col = lerpColor(p.heat);

        // glow / alpha based on heat and twinkle
        const alpha = 0.45 + 0.55 * p.heat * (0.85 + 0.15 * Math.sin(time * 2.1 + p.phase));
        const glow = 0.3 + 0.7 * p.heat;

        // draw glow circle (heat map style)
        ctx.beginPath();
        ctx.arc(x, y, size * 0.8, 0, Math.PI * 2);
        ctx.fillStyle = `rgba(${col.r}, ${col.g}, ${col.b}, ${alpha * 0.35})`;
        ctx.fill();

        // main bright core
        ctx.beginPath();
        ctx.arc(x, y, size * 0.5, 0, Math.PI * 2);
        ctx.fillStyle = `rgba(${col.r}, ${col.g}, ${col.b}, ${alpha * 0.7})`;
        ctx.fill();

        // hot highlight (white-ish)
        if (p.heat > 0.5) {
          ctx.beginPath();
          ctx.arc(x - size*0.08, y - size*0.08, size * 0.15, 0, Math.PI * 2);
          ctx.fillStyle = `rgba(255, 240, 200, ${0.2 * p.heat})`;
          ctx.fill();
        }

        // extra spark / glint (animation)
        if (p.heat > 0.6 && Math.sin(time * 3.3 + p.phase * 2) > 0.6) {
          ctx.beginPath();
          ctx.arc(x + size*0.2, y - size*0.2, size * 0.08, 0, Math.PI * 2);
          ctx.fillStyle = `rgba(255, 255, 220, 0.5)`;
          ctx.fill();
        }
      }

      // ---- 3. extra radial "heat haze" (animated) ----
      const pulse = 0.6 + 0.4 * Math.sin(time * 0.5);
      const grad2 = ctx.createRadialGradient(
        w/2 + 20 * Math.sin(time*0.2), h/2 + 20 * Math.cos(time*0.3), 30,
        w/2, h/2, 280 * pulse
      );
      grad2.addColorStop(0, 'rgba(255, 140, 40, 0.05)');
      grad2.addColorStop(0.5, 'rgba(200, 70, 20, 0.04)');
      grad2.addColorStop(1, 'rgba(0,0,0,0)');
      ctx.fillStyle = grad2;
      ctx.fillRect(0, 0, w, h);

      // ---- 4. subtle vignette ----
      const vig = ctx.createRadialGradient(w/2, h/2, w*0.3, w/2, h/2, w*0.75);
      vig.addColorStop(0, 'rgba(0,0,0,0)');
      vig.addColorStop(1, 'rgba(0,0,0,0.4)');
      ctx.fillStyle = vig;
      ctx.fillRect(0, 0, w, h);

      requestAnimationFrame(drawHeatmap);
    }

    drawHeatmap();

    // resize handler: keep aspect ratio but canvas is fixed 600x600, we just scale via CSS
    // (already handled by container)
  })();
</script>
</body>
</html>
