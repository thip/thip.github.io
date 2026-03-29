---
layout: post
title:  "Simulating a glowing fireplace with an RP2040"
image: /assets/firestring.jpg
description: "Building a custom LED fire simulation for a blocked-up fireplace, from prototype to custom PCB and optimised firmware."
permalink: /posts/fire-string
---

![The Fire String in a fireplace](/assets/firestring.jpg)
>If you want one of these for yourself, I've got a limited number available on my store: [hortus.dev/products/fire-string](https://hortus.dev/products/fire-string).

Like many houses in the UK, mine has an old fireplace that can't be used. The previous owners blocked up the chimney and replaced the fire with an imitation electric one. I imagine they had perfectly understandable reasons for doing this, maybe they struggled with maintaining it or they were worried about safety with small children or pets around. From my perspective though, I always think these heaters are a poor substitute for the real thing. I've yet to come across one that doesn't remind me that it would be much nicer if there was a real fire roaring in the hearth. As a result, the cheap electric heater that came with the house quickly made its way to gumtree so that someone who might appreciate it more could put it to good use. This left my partner and me with a slightly sad looking void in our living room. Over the years we've tried to fill it with various things like pots, plants and most recently a large glass jar. They looked nice, but always slightly out of place and they never did much to restore the coziness that I missed from having a real fire - specifically that relaxed feeling as you let the dancing embers hypnotise you as the fire burns down late in the evening. This felt like something I could capture with a microcontroller, a string of RGB LEDs, and a couple of free evenings.

## The Theory

I love writing little simulations of physical things. There's something satisfying about taking a real physical process, stripping it down to its core ideas, and seeing how far a few simple rules will get you. This project is very much in that spirit - it was inspired by the way that heat diffuses through and radiates away from objects as visible light, though as you'll see there are a lot of big simplifications along the way.

The basic idea is simple: All objects emit light when they get hot. The hotter something is, the more it glows towards the blue end of the spectrum, and the cooler it is the more it glows towards the red end. There is a very specific relationship between temperature and colour. We call it black-body radiation and it is governed by something called Planck's law. We, however, are not going to worry about that too much as for our purposes we don't need to be physically accurate - we just need something that looks nice. We can get away with mapping some imaginary temperatures to a colour gradient that gives a satisfying effect. All you need to know is that we're going to make the LED pixels in our string white when we want them to look really hot, red when we want them to look cooler, and yellow/orange for the temperatures in-between.

<div style="background:#111;border-radius:8px;padding:16px;margin:1.5em 0">
  <div style="color:#fff;font-size:20px;font-weight:bold;margin-bottom:4px">Black-body radiation</div>
  <p style="color:#ccc;font-size:16px;margin:0 0 12px;line-height:1.5"><em>Try adjusting the temperature slider on the right to see how the wavelengths of light that are emitted change. Low temperatures emit light that only covers the red end of the visible spectrum, but as it gets hotter the peak shifts towards blue and we see a bigger range of wavelengths emitted across the visible spectrum which we perceive as white.</em></p>
  <hr style="border:none;border-top:2px solid #ccc;margin:0 0 12px">
  <div style="display:flex;gap:16px;align-items:stretch">
    <div style="flex:1;min-width:0">
      <div style="color:#ccc;font-size:16px;font-weight:bold;margin-bottom:4px">Perceived colour:</div>
      <canvas id="fs-bb-pixels" style="width:100%;display:block;border-radius:4px"></canvas>
      <div id="fs-bb-label" style="color:#bbb;font-size:16px;margin-top:8px;font-family:monospace;white-space:pre-line"></div>
      <div style="color:#ccc;font-size:16px;font-weight:bold;margin-top:12px;margin-bottom:4px">Emitted wavelengths of light:</div>
      <canvas id="fs-bb-chart" style="width:100%;display:block;border-radius:4px"></canvas>
    </div>
    <div style="display:flex;flex-direction:column;align-items:center;min-width:40px">
      <span style="color:#ccc;font-size:16px;font-weight:bold;">Temperature:</span>
      <span style="color:#ccc;font-size:16px">6000 °C</span>
      <input type="range" id="fs-bb-temp" min="773" max="6273" value="3000" orient="vertical" style="writing-mode:vertical-lr;direction:rtl;-webkit-appearance:slider-vertical;flex:1;margin:8px 0">
      <span style="color:#ccc;font-size:16px">500 °C</span>
    </div>
  </div>
</div>
<script>
(function () {
  // CIE 1931 2° colour matching functions, 380–700 nm at 10 nm steps (33 entries)
  var CMF0 = 380, CMFS = 10;
  var xb = [0.001368,0.004243,0.014310,0.043510,0.134380,0.283900,0.348280,0.336200,0.290800,0.195360,0.095640,0.032010,0.004900,0.009300,0.063270,0.165500,0.290400,0.433450,0.594500,0.762100,0.916800,1.026300,1.062200,1.002600,0.854450,0.642400,0.447900,0.283500,0.164900,0.087400,0.046770,0.022700,0.011359];
  var yb = [0.000039,0.000120,0.000396,0.001210,0.004000,0.011600,0.023000,0.038000,0.060000,0.090980,0.139020,0.208020,0.323000,0.503000,0.710000,0.862000,0.954000,0.994950,0.995000,0.952000,0.870000,0.757000,0.631000,0.503000,0.381000,0.265000,0.175000,0.107000,0.061000,0.032000,0.017000,0.008210,0.004102];
  var zb = [0.006450,0.020050,0.067850,0.207400,0.645600,1.385600,1.747060,1.772110,1.669200,1.287640,0.812950,0.465180,0.272000,0.158200,0.078250,0.042160,0.020300,0.008750,0.003900,0.002100,0.001650,0.001100,0.000800,0.000340,0.000190,0.000050,0.000020,0.000000,0.000000,0.000000,0.000000,0.000000,0.000000];

  var H_PLANCK = 6.62607015e-34, C_LIGHT = 2.99792458e8, K_B = 1.380649e-23;
  var LMIN = 200, LMAX = 1500; // chart wavelength range (nm)
  var VIS0 = 380, VIS1 = 700;  // visible range (nm)
  var PCOLS = 10, PROWS = 1;

  // Spectral radiance via Planck's law, λ in nm, T in K
  function planck(lam_nm, T) {
    var l = lam_nm * 1e-9;
    var e = (H_PLANCK * C_LIGHT) / (l * K_B * T);
    if (e > 700) return 0; // prevent overflow for very short λ or low T
    return (2 * H_PLANCK * C_LIGHT * C_LIGHT) / (Math.pow(l, 5) * (Math.exp(e) - 1));
  }

  // Integrate Planck × CIE CMFs → XYZ → linear sRGB → gamma → display colour
  // Returns [r, g, b] in 0–1 with brightness scaled relative to refY (6500 K luminance)
  function blackbodyRgb(T) {
    var X = 0, Y = 0, Z = 0;
    for (var i = 0; i < xb.length; i++) {
      var p = planck(CMF0 + i * CMFS, T);
      X += p * xb[i]; Y += p * yb[i]; Z += p * zb[i];
    }
    if (Y <= 0) return [0, 0, 0];
    // Brightness relative to 6500 K — pow(0.22) gives near-black at low T, smooth ramp to full
    var brightness = Math.pow(Y / refY, 0.22);
    // Normalise to Y = 1 to get chromaticity, then convert to linear sRGB
    var xn = X / Y, zn = Z / Y;
    var lr =  3.2406 * xn - 1.5372 - 0.4986 * zn;
    var lg = -0.9689 * xn + 1.8758 + 0.0415 * zn;
    var lb =  0.0557 * xn - 0.2040 + 1.0570 * zn;
    lr = Math.max(0, lr); lg = Math.max(0, lg); lb = Math.max(0, lb);
    // Normalise chromaticity so max channel = 1, then apply brightness
    var mx = Math.max(lr, lg, lb, 1e-10);
    lr = lr / mx * brightness; lg = lg / mx * brightness; lb = lb / mx * brightness;
    // Clamp (brightness can exceed 1 near refY temperature)
    lr = Math.min(1, lr); lg = Math.min(1, lg); lb = Math.min(1, lb);
    // sRGB gamma encode
    function enc(v) { return v <= 0.0031308 ? 12.92 * v : 1.055 * Math.pow(v, 1/2.4) - 0.055; }
    return [enc(lr), enc(lg), enc(lb)];
  }

  // Monochromatic wavelength → approximate display RGB (for spectrum strip)
  function spectrumRgb(nm) {
    var r = 0, g = 0, b = 0;
    if      (nm >= 380 && nm < 440) { r = (440 - nm) / 60; b = 1; }
    else if (nm >= 440 && nm < 490) { g = (nm - 440) / 50; b = 1; }
    else if (nm >= 490 && nm < 510) { g = 1; b = (510 - nm) / 20; }
    else if (nm >= 510 && nm < 580) { r = (nm - 510) / 70; g = 1; }
    else if (nm >= 580 && nm < 645) { r = 1; g = (645 - nm) / 65; }
    else if (nm >= 645 && nm <= 700) { r = 1; }
    var f = nm < 420 ? 0.3 + 0.7*(nm-380)/40 : nm > 660 ? 0.3 + 0.7*(700-nm)/40 : 1;
    return [Math.round(Math.pow(r*f, 0.8)*190), Math.round(Math.pow(g*f, 0.8)*190), Math.round(Math.pow(b*f, 0.8)*190)];
  }

  // Reference values at 6500 K for relative scaling of curve height and pixel brightness.
  var refPeak = planck(2898000 / 6500, 6500);
  var refY = (function () {
    var Y = 0;
    for (var i = 0; i < yb.length; i++) Y += planck(CMF0 + i * CMFS, 6500) * yb[i];
    return Y;
  })();

  var chart  = document.getElementById('fs-bb-chart');
  var pixels = document.getElementById('fs-bb-pixels');
  var slider = document.getElementById('fs-bb-temp');
  var lbl    = document.getElementById('fs-bb-label');

  // Chart layout constants
  var LABEL_H = 40;

  function drawChart(T) {
    var W = chart.offsetWidth || 600;
    var H = 150 + LABEL_H;
    var CTOP = LABEL_H, CBOT = H, CH = CBOT - CTOP;
    chart.width = W; chart.height = H;
    var ctx = chart.getContext('2d');

    // Label area at top
    ctx.fillStyle = '#111'; ctx.fillRect(0, 0, W, LABEL_H);

    // Spectrum background below labels
    for (var x = 0; x < W; x++) {
      var lam = LMIN + (x / W) * (LMAX - LMIN);
      if (lam >= VIS0 && lam <= VIS1) {
        var sc = spectrumRgb(lam);
        ctx.fillStyle = 'rgb('+Math.round(sc[0]*0.7)+','+Math.round(sc[1]*0.7)+','+Math.round(sc[2]*0.7)+')';
      } else {
        ctx.fillStyle = lam < VIS0 ? '#120a18' : '#0a0e14';
      }
      ctx.fillRect(x, CTOP, 1, CH);
    }
    // Spectrum region labels with bracket bars (top row of label area)
    var vis0x = Math.round((VIS0 - LMIN) / (LMAX - LMIN) * W);
    var vis1x = Math.round((VIS1 - LMIN) / (LMAX - LMIN) * W);
    ctx.font = '12px sans-serif'; ctx.textBaseline = 'bottom'; ctx.textAlign = 'center';
    ctx.fillStyle = '#bbb'; ctx.strokeStyle = '#bbb'; ctx.lineWidth = 1;
    function regionLabel(text, x0, x1) {
      var midX = Math.round((x0 + x1) / 2);
      var barY = 18;
      ctx.beginPath(); ctx.moveTo(x0, barY); ctx.lineTo(x1, barY); ctx.stroke();
      ctx.beginPath(); ctx.moveTo(x0, barY - 3); ctx.lineTo(x0, barY + 3); ctx.stroke();
      ctx.beginPath(); ctx.moveTo(x1, barY - 3); ctx.lineTo(x1, barY + 3); ctx.stroke();
      ctx.fillText(text, midX, barY - 5);
    }
    if (vis0x > 30) regionLabel('UV', 0, vis0x);
    regionLabel('Visible', vis0x, vis1x);
    if (vis1x < W - 30) regionLabel('Infrared', vis1x, W);

    // Visible range boundary marks
    ctx.strokeStyle = 'rgba(255,255,255,0.08)'; ctx.lineWidth = 1;
    [vis0x, vis1x].forEach(function (vx) {
      ctx.beginPath(); ctx.moveTo(vx, CTOP); ctx.lineTo(vx, CBOT); ctx.stroke();
    });

    // Build Planck curve. Normalise shape to its own peak,
    // then scale height by pow(chartMax/refPeak, 0.25) so cooler curves appear shorter.
    var curve = new Float64Array(W), chartMax = 0;
    for (var x = 0; x < W; x++) {
      curve[x] = planck(LMIN + (x / W) * (LMAX - LMIN), T);
      if (curve[x] > chartMax) chartMax = curve[x];
    }
    var scale = chartMax > 0 ? Math.pow(chartMax / refPeak, 0.25) : 0;
    if (chartMax > 0) for (var x = 0; x < W; x++) curve[x] = (curve[x] / chartMax) * scale;

    // Filled area under curve
    ctx.beginPath(); ctx.moveTo(0, CBOT);
    for (var x = 0; x < W; x++) ctx.lineTo(x, CBOT - curve[x] * CH);
    ctx.lineTo(W, CBOT); ctx.closePath();
    ctx.fillStyle = 'rgba(255,160,40,0.18)'; ctx.fill();

    // Curve line
    ctx.beginPath(); ctx.moveTo(0, CBOT - curve[0] * CH);
    for (var x = 1; x < W; x++) ctx.lineTo(x, CBOT - curve[x] * CH);
    ctx.strokeStyle = 'rgba(255,200,80,0.9)'; ctx.lineWidth = 1.5; ctx.stroke();

    // Wien peak dashed line (λ_max = b/T, b = 2.898e6 nm·K)
    var wienLam = 2898000 / T;
    var wienX = (wienLam - LMIN) / (LMAX - LMIN) * W;
    if (wienX > 0 && wienX < W) {
      ctx.save(); ctx.setLineDash([3, 4]);
      ctx.strokeStyle = 'rgba(255,255,255,0.3)'; ctx.lineWidth = 1;
      ctx.beginPath(); ctx.moveTo(wienX, CTOP); ctx.lineTo(wienX, CBOT); ctx.stroke();
      ctx.restore();
    }

  }

  function drawPixels(T) {
    var W = pixels.offsetWidth || 600;
    var ps = Math.floor(W / PCOLS);
    var gap = Math.max(2, Math.floor(ps * 0.08));
    pixels.width = ps * PCOLS; pixels.height = ps * PROWS;
    var ctx = pixels.getContext('2d');
    ctx.fillStyle = '#111'; ctx.fillRect(0, 0, pixels.width, pixels.height);
    var col = blackbodyRgb(T);
    ctx.fillStyle = 'rgb('+Math.round(col[0]*255)+','+Math.round(col[1]*255)+','+Math.round(col[2]*255)+')';
    for (var row = 0; row < PROWS; row++)
      for (var c = 0; c < PCOLS; c++)
        ctx.fillRect(c * ps + gap, row * ps + gap, ps - gap*2, ps - gap*2);
  }

  function update() {
    var T = parseInt(slider.value);
    var wien = Math.round(2898000 / T);
    var desc = T < 798 ? 'black' : T < 1500 ? 'dull red' : T < 2500 ? 'orange' : T < 3000 ? 'yellow' : T < 3500 ? 'warm white' : T < 5500 ? 'white' : T < 8000 ? 'cool white' : 'blue-white';
    lbl.textContent = (T - 273) + ' °C - ' + desc + '\nPeak wavelength at ' + wien + ' nm';
    drawChart(T);
    drawPixels(T);
  }

  slider.addEventListener('input', update);
  window.addEventListener('resize', update);
  update();
})();
</script>

Black-body radiation isn't just about colour, it also represents heat leaving a system. In a real physical system some heat is lost as radiation (the light that gets emitted by hot objects as previously discussed) and some is lost through conduction to whatever substance surrounds the object. We're not going to bother modelling heat loss to radiation, but we can simulate heat being conducted away to the air very easily by just saying that each pixel always loses some small proportion of its heat. This will enable our LEDs to go from white to red and eventually fade to black as they cool.

This by itself wouldn't make for a particularly interesting effect though. We need one last concept: some way for the temperature of pixels to change and for heat to spread through the pixels. To do that we introduce diffusion of thermal energy from hotter areas to cooler areas. This is governed by _the heat equation_, but again we can skip over and simplify this by going with the idea that every pixel will move some proportion of its heat to its neighbours. With all that in place we've got the foundations we need to code this up and start building our simulation.

<div style="background:#111;border-radius:8px;padding:16px;margin:1.5em 0">
  <div style="color:#fff;font-size:20px;font-weight:bold;margin-bottom:4px">Decay and diffusion</div>
  <p style="color:#ccc;font-size:16px;margin:0 0 12px;line-height:1.5"><em>Click or drag on the grid to add heat, then watch it spread and fade. Decay controls how quickly each pixel loses heat. Diffusion controls how much heat spreads to neighbouring pixels each frame. Try setting decay to zero and watch how heat diffuses outward until the whole grid settles to a uniform temperature.</em></p>
  <hr style="border:none;border-top:2px solid #ccc;margin:0 0 12px">
  <div id="fs-dd-wrap" style="position:relative">
    <canvas id="fs-dd-canvas" style="width:100%;display:block;border-radius:4px;cursor:crosshair"></canvas>
    <div id="fs-dd-overlay" style="position:absolute;top:0;left:0;width:100%;height:100%;display:flex;align-items:center;justify-content:center;background:rgba(0,0,0,0.5);border-radius:4px;cursor:pointer">
      <span style="color:#fff;font-size:18px;font-weight:bold;pointer-events:none">Click here to draw on the grid</span>
    </div>
  </div>
  <div style="margin-top:12px;display:grid;grid-template-columns:5em 1fr 3em;align-items:center;gap:6px 10px">
    <span style="color:#ccc;font-size:16px">Decay</span>
    <input type="range" id="fs-dd-decay" min="0" max="40" value="8" style="margin:0">
    <span id="fs-dd-decay-val" style="color:#aaa;font-size:16px;font-family:monospace;text-align:right"></span>
    <span style="color:#ccc;font-size:16px">Diffusion</span>
    <input type="range" id="fs-dd-diff" min="0" max="70" value="30" style="margin:0">
    <span id="fs-dd-diff-val" style="color:#aaa;font-size:16px;font-family:monospace;text-align:right"></span>
  </div>
</div>
<script>
(function () {
  var COLS = 10, ROWS = 3, N = COLS * ROWS;
  var LUT_SIZE = 101, LUT_STEP = 10;

  function lerpCurve(curve, x) {
    if (x <= curve[0][0]) return curve[0][1];
    if (x >= curve[curve.length - 1][0]) return curve[curve.length - 1][1];
    for (var i = 0; i < curve.length - 1; i++) {
      var x0 = curve[i][0], y0 = curve[i][1], x1 = curve[i+1][0], y1 = curve[i+1][1];
      if (x0 <= x && x <= x1) return y0 + (x - x0) / (x1 - x0) * (y1 - y0);
    }
    return curve[curve.length - 1][1];
  }
  var palR = [[0,0],[250,230],[500,250],[750,255],[1000,255]];
  var palG = [[0,0],[250,70],[500,180],[750,240],[1000,255]];
  var palB = [[0,0],[250,0],[500,40],[750,200],[1000,255]];
  var colLUT = [];
  for (var ci = 0; ci < LUT_SIZE; ci++) {
    var cx = ci * LUT_STEP;
    colLUT.push([Math.round(lerpCurve(palR,cx)), Math.round(lerpCurve(palG,cx)), Math.round(lerpCurve(palB,cx))]);
  }

  var cur = new Float64Array(N);
  var lst = new Float64Array(N);

  var canvas    = document.getElementById('fs-dd-canvas');
  var decSlider = document.getElementById('fs-dd-decay');
  var difSlider = document.getElementById('fs-dd-diff');
  var decLbl    = document.getElementById('fs-dd-decay-val');
  var difLbl    = document.getElementById('fs-dd-diff-val');

  function simulateFrame() {
    var decay = parseInt(decSlider.value) / 100;
    var diff  = parseInt(difSlider.value) / 100 / 4; // per-pair rate, scaled for stability
    var i, row, col, heat, flow, j;

    // Snapshot current into last so diffusion uses consistent previous-frame values
    for (i = 0; i < N; i++) lst[i] = cur[i];

    // Pairwise diffusion: transfer heat proportional to temperature difference
    // Process only right and down edges to avoid double-counting
    for (i = 0; i < N; i++) {
      row = Math.floor(i / COLS); col = i % COLS;
      if (col < COLS - 1) {
        j = i + 1;
        flow = diff * (lst[i] - lst[j]);
        cur[i] -= flow; cur[j] += flow;
      }
      if (row < ROWS - 1) {
        j = i + COLS;
        flow = diff * (lst[i] - lst[j]);
        cur[i] -= flow; cur[j] += flow;
      }
    }

    // Clamp then decay
    for (i = 0; i < N; i++) {
      heat = cur[i] < 0 ? 0 : cur[i];
      cur[i] = heat - heat * decay;
    }
  }

  function draw() {
    var cw = canvas.offsetWidth || 400;
    var ps = Math.floor(cw / COLS);
    var gap = Math.max(2, Math.floor(ps * 0.08));
    canvas.width = ps * COLS; canvas.height = ps * ROWS;
    var ctx = canvas.getContext('2d');
    ctx.fillStyle = '#111'; ctx.fillRect(0, 0, canvas.width, canvas.height);
    for (var i = 0; i < N; i++) {
      var col = i % COLS, row = Math.floor(i / COLS);
      var heat = cur[i] < 0 ? 0 : cur[i] > 1000 ? 1000 : cur[i];
      var hi = Math.min(Math.floor(heat / LUT_STEP), LUT_SIZE - 1);
      var c = colLUT[hi];
      ctx.fillStyle = 'rgb('+c[0]+','+c[1]+','+c[2]+')';
      ctx.fillRect(col * ps + gap, row * ps + gap, ps - gap * 2, ps - gap * 2);
    }
    decLbl.textContent = parseInt(decSlider.value) + '%';
    difLbl.textContent = parseInt(difSlider.value) + '%';
  }

  // Click / drag to paint heat
  function pixelAt(clientX, clientY) {
    var rect = canvas.getBoundingClientRect();
    var x = (clientX - rect.left) / rect.width  * canvas.width;
    var y = (clientY - rect.top)  / rect.height * canvas.height;
    var ps = canvas.width / COLS;
    var col = Math.floor(x / ps), row = Math.floor(y / ps);
    return (col >= 0 && col < COLS && row >= 0 && row < ROWS) ? row * COLS + col : -1;
  }
  function addHeat(clientX, clientY) {
    var idx = pixelAt(clientX, clientY);
    if (idx >= 0) cur[idx] = 1000;
  }
  var dragging = false;
  var overlay = document.getElementById('fs-dd-overlay');
  function dismissOverlay() {
    if (overlay) { overlay.style.display = 'none'; overlay = null; }
  }
  canvas.addEventListener('mousedown', function (e) { dismissOverlay(); dragging = true; addHeat(e.clientX, e.clientY); e.preventDefault(); });
  canvas.addEventListener('mousemove', function (e) { if (dragging) addHeat(e.clientX, e.clientY); });
  window.addEventListener('mouseup',   function ()  { dragging = false; });
  canvas.addEventListener('touchstart', function (e) { dismissOverlay(); addHeat(e.touches[0].clientX, e.touches[0].clientY); e.preventDefault(); }, { passive: false });
  canvas.addEventListener('touchmove',  function (e) { addHeat(e.touches[0].clientX, e.touches[0].clientY); e.preventDefault(); }, { passive: false });
  // Also dismiss overlay if clicked directly
  if (overlay) overlay.addEventListener('click', function (e) { dismissOverlay(); addHeat(e.clientX, e.clientY); });

  window.addEventListener('resize', draw);

  var lastRender = 0, lastPhysics = 0;
  var RENDER_MS = 1000 / 30, PHYSICS_MS = 1000 / 15; // render 30fps, simulate 15 ticks/s
  function loop(ts) {
    if (ts - lastPhysics >= PHYSICS_MS) { simulateFrame(); lastPhysics = ts; }
    if (ts - lastRender  >= RENDER_MS)  { draw();          lastRender  = ts; }
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
})();
</script>

## Making it work

Rummaging around in my parts box I picked out a Pimoroni Tiny 2040 board and a nice long string of WS2812B LEDs that I'd been saving for a rainy day project just like this one. Hooking them up was as simple as connecting power, ground and data directly to the Tiny 2040. WS2812B led string like the one I used can draw quite a lot of power so it is worth thinking about how you are going to supply them. In this case however these concerns were mitigated by the fact that most of the LEDs would be lit as a low red with only the occasional bright white ember.


I initially implemented the code using circuit python mostly out of convenience. There's something satisfying about being able to write code directly on the device and getting real time feedback as you make changes. The LED string was represented using an array. I implemented the ember effect by first adding heat to random positions in the array every frame, and then iterating through it applying the rules for decay and diffusion to each element and its neighbours. I then used a lookup table to render the correct colour values for the resulting temperatures in the array to the LED string.

<div style="background:#111;border-radius:8px;padding:16px;margin:1.5em 0">
  <div style="color:#fff;font-size:20px;font-weight:bold;margin-bottom:4px">The ember simulation</div>
  <p style="color:#ccc;font-size:16px;margin:0 0 12px;line-height:1.5"><em>Heat is randomly added to pixels each frame, then decay and diffusion are applied to produce flowing waves of colour. Each row is a continuous strip of LEDs, just like the physical string.</em></p>
  <hr style="border:none;border-top:2px solid #ccc;margin:0 0 12px">
  <canvas id="fs-basic-canvas" style="width:100%;display:block;border-radius:4px"></canvas>
  <div style="text-align:right;margin-top:10px">
    <button id="fs-basic-btn" style="background:#222;color:#ccc;border:1px solid #444;border-radius:4px;padding:4px 20px;cursor:pointer;font-size:16px">Pause simulation</button>
  </div>
</div>
<script>
(function () {
  var LUT_SIZE = 101, LUT_STEP = 10;
  var COLS = 20, ROWS = 6, N = COLS * ROWS;

  function lerpCurve(curve, x) {
    if (x <= curve[0][0]) return curve[0][1];
    if (x >= curve[curve.length - 1][0]) return curve[curve.length - 1][1];
    for (var i = 0; i < curve.length - 1; i++) {
      var x0 = curve[i][0], y0 = curve[i][1], x1 = curve[i+1][0], y1 = curve[i+1][1];
      if (x0 <= x && x <= x1) return y0 + (x - x0) / (x1 - x0) * (y1 - y0);
    }
    return curve[curve.length - 1][1];
  }

  // Same boosted screen palette as the full sim
  var palR = [[0,0],[250,230],[500,250],[750,255],[1000,255]];
  var palG = [[0,0],[250,70],[500,180],[750,240],[1000,255]];
  var palB = [[0,0],[250,0],[500,40],[750,200],[1000,255]];
  var colLUT = [];
  for (var ci = 0; ci < LUT_SIZE; ci++) {
    var cx = ci * LUT_STEP;
    colLUT.push([Math.round(lerpCurve(palR,cx)), Math.round(lerpCurve(palG,cx)), Math.round(lerpCurve(palB,cx))]);
  }

  var cur = new Float64Array(N), lst = new Float64Array(N);
  var DECAY = 0.06, DIFF = 0.2, SPARK_CHANCE = 0.008, SPARK_HEAT = 1000;

  function simulateFrame() {
    var i, j, heat, totalDiff, hpn, ni;

    // Random sparks — heat added directly, no fuel system
    for (i = 0; i < N; i++) {
      if (Math.random() < SPARK_CHANCE) cur[i] = Math.min(cur[i] + SPARK_HEAT, 1000);
    }

    // Snapshot for diffusion
    for (i = 0; i < N; i++) lst[i] = cur[i];

    // Diffuse from neighbours (offset=1, no skipping)
    for (i = 0; i < N; i++) {
      heat = lst[i];
      if (heat <= 0) continue;
      totalDiff = heat * DIFF;
      if (totalDiff > heat) totalDiff = heat;
      // Left and right neighbours only (1D chain wrapped across rows)
      var nCount = 0;
      if (i > 0) nCount++;
      if (i < N - 1) nCount++;
      if (nCount === 0) continue;
      hpn = totalDiff / nCount;
      cur[i] -= totalDiff;
      if (i > 0) cur[i - 1] += hpn;
      if (i < N - 1) cur[i + 1] += hpn;
    }

    // Clamp + decay
    for (i = 0; i < N; i++) {
      heat = cur[i] < 0 ? 0 : cur[i] > 1000 ? 1000 : cur[i];
      cur[i] = heat - heat * DECAY;
    }
  }

  var canvas = document.getElementById('fs-basic-canvas');
  var btn    = document.getElementById('fs-basic-btn');

  function clamp(v, lo, hi) { return v < lo ? lo : v > hi ? hi : v; }

  function draw() {
    var cw = canvas.offsetWidth || 600;
    var ps = Math.floor(cw / COLS);
    var gap = Math.max(2, Math.floor(ps * 0.08));
    canvas.width  = ps * COLS;
    canvas.height = ps * ROWS;
    var ctx = canvas.getContext('2d');
    ctx.fillStyle = '#111'; ctx.fillRect(0, 0, canvas.width, canvas.height);
    for (var i = 0; i < N; i++) {
      var row = Math.floor(i / COLS);
      var col = (row % 2 === 0) ? i % COLS : COLS - 1 - (i % COLS); // snake: odd rows reversed
      var hi = Math.min(Math.floor(clamp(cur[i], 0, 1000) / LUT_STEP), LUT_SIZE - 1);
      var c = colLUT[hi];
      ctx.fillStyle = 'rgb('+c[0]+','+c[1]+','+c[2]+')';
      ctx.fillRect(col * ps + gap, row * ps + gap, ps - gap * 2, ps - gap * 2);
    }
  }

  var paused = false, lastTs = 0, FRAME_MS = 1000 / 15;
  function loop(ts) {
    if (!paused) {
      if (ts - lastTs >= FRAME_MS) { simulateFrame(); draw(); lastTs = ts; }
      requestAnimationFrame(loop);
    }
  }
  btn.addEventListener('click', function () {
    paused = !paused;
    btn.textContent = paused ? 'Resume simulation' : 'Pause simulation';
    if (!paused) { lastTs = 0; requestAnimationFrame(loop); }
  });
  window.addEventListener('resize', draw);
  requestAnimationFrame(loop);
})();
</script>

This produced a good effect, but it wasn't quite what I was after. The string lit up in waves of red and yellow which was very nice, but it was a bit too continuous. I wanted little pinpricks of light that weren't entirely connected, but still ebbed and glowed in roughly the same areas, as I thought this would produce a more convincing ember effect. Luckily this was pretty easy to do - I simply updated my simulation to skip positions in the array, which split and spread the flowing waves of heat across the string more.

The circuit python implementation turned out to be good enough that it's been running in our living room for the last couple of months. The real measure of success though is that my wife has started turning it on unprompted - which tells me it's actually doing its job. There are still some problems though. I was very lazy with my connections for the led string - just twisting and looping the wires through the holes in the Tiny 2040 without any solder. This works fine as long as no one touches it. If the USB cable gets jogged it can make the whole string cut out, flicker, or even burst into a rainbow of colours which isn't really conducive to the warm relaxing vibe I'm going for. With that in mind, and the confidence that we would actually continue to use it I felt it was time to productionise things.

## Making it nice

I wanted to start by designing a custom board around the RP2040. I hadn't worked with it before beyond using random dev boards, and I was intrigued to see what it would be like to integrate into one of my own products. Potentially earning it (and maybe later its bigger brother, the RP2350) a permanent place on my roster of go-to chips. I probably could have got away with continuing to use an off-the-shelf dev board but there's something satisfying about owning the end-to-end design and I'm never one to pass up an opportunity to learn something new.

I followed the reference design from the official raspberry pi [RP2040 hardware design guide](https://pip-assets.raspberrypi.com/categories/814-rp2040/documents/RP-008279-DS-1-hardware-design-with-rp2040.pdf?disposition=inline) pretty closely, adding a 3 pin JST-XH connector for the LED string and a USB-C connector for charging and programming (it annoys me that the official pi pico boards still use usb micro connectors, as it's one more type of cable that I have to keep around). The RP2040 needs more supporting hardware than I am used to. The nature of my projects so far have meant that I tend to get away without an external crystal or flash but both are required for the RP2040. It also has slightly more interesting power supply requirements, needing both a 3.3v supply and a 1.1v supply (which can be generated by an internal LDO, but still requires traces between pins). This is something I somehow missed when designing the first revision of the board, resulting in some very hot, very broken chips when I plugged them in. I had managed to connect the 3.3v supply to the output of the 1.1v LDO. Unfortunately the relevant traces were all hidden under the chip which made a bodge-fix impractical. This was annoying, but mistakes happen and I learned a while ago not to beat myself up too much about it! Whenever I design new hardware I always anticipate and budget for a couple of revisions on the basis that I won't get things perfectly right on the first shot, but thankfully with time and experience silly mistakes of this magnitude are becoming pretty rare for me.

![3D render of the custom Fire String PCB](/assets/firestring_pcb.png)
*3D render of the custom RP2040 board from KiCad*

Happily the second revision of my boards worked flawlessly, which meant it was time to focus on firmware improvements. My original firmware worked well, but I was aware that it wasn't particularly well optimised which was limiting the frame rate to about 10 frames per second. That in turn meant that there wasn't much headroom available to further tune and improve the ember effect, or experiment with new approaches. I broke out Claude Code to help me with this. I'm aware that for some people reading, the use of AI might be a bit of a turn off, but for me it has vastly expanded my ability to execute and finish projects and ideas. I am as concerned as anyone about the rise of AI slop, but I strongly believe that these tools offer huge amounts of value and potential when wielded intentionally. For me, that means delegating implementation (especially when it is tedious or repetitive) but not decision making, and also using it to fill the gaps of my knowledge and experience, but not going beyond it. I still love writing code for its own sake, and I don't think being good at it stops mattering - if anything it matters more, because it's what lets you make good decisions about what to delegate and what to keep. For me the sweet spot is building out a solid skeleton of an idea myself and using AI to help flesh it out without needing to be an expert in everything. It lets me move faster without losing the thread of what I'm actually trying to build.

Claude helped me go from 10 frames a second up to about 80. The biggest win was porting from Circuit Python to Micro Python so that we could take advantage of the `@micropython.viper` annotation which lets us compile our code down to machine code at runtime. This allowed us to write colour values for each LED directly into memory, bypassing all the usual Python bookkeeping. Without it, Python was doing a lot of invisible work behind the scenes for every single LED, on every single frame which adds up quickly over 300 LEDs. This optimisation got us around a 3x speedup. The next biggest win was converting all of the floating point maths to integer maths. Floating point is how computers typically handle decimal numbers, but the RP2040 has no dedicated hardware for it - every calculation has to be emulated in software, which is slow. By scaling everything up to work with whole numbers instead, we cut that overhead entirely and got another 2x speedup.

Having reclaimed a significant amount of processor time I was able to really go to town on the effect and experiment with tuning it. The first thing I did was change the way heat got added to the system. Instead of randomly adding it directly to the pixel array, I introduced the concept of fuel by creating a second array, and slowly transferring heat from this to the pixel array via a burn rate parameter. This meant that hot pixels would ramp up in their brightness rather than instantly lighting up. Next I changed the diffusion, decay and burn rate parameters as well as the probability of fuel being added to use look up tables based on the current temperature of the pixel they were being applied to. This meant that instead of every pixel behaving the same way regardless of how hot it was, each pixel could follow its own curve. A cool pixel might decay slowly and be quick to reignite, while a hot pixel might shed heat rapidly - mimicking the way real embers behave differently at different stages of burning. This gave the effect a much more organic, unpredictable feel, with most pixels settling into a deep red glow but occasional flares punching all the way through to white hot before fading back down.

<div style="background:#111;border-radius:8px;padding:16px;margin:1.5em 0">
  <div style="color:#fff;font-size:20px;font-weight:bold;margin-bottom:4px">The full ember simulation</div>
  <p style="color:#ccc;font-size:16px;margin:0 0 12px;line-height:1.5"><em>This is the actual simulation running in real time using the same algorithm as the firmware. Fuel is randomly added to pixels and slowly converted to heat via a burn rate, which is why embers ramp up gradually rather than appearing instantly. Each parameter varies with temperature - cool pixels decay slowly and reignite easily, while hot pixels shed heat rapidly.</em></p>
  <hr style="border:none;border-top:2px solid #ccc;margin:0 0 12px">
  <canvas id="fs-sim-canvas" style="width:100%;display:block;border-radius:4px"></canvas>
  <div style="display:flex;align-items:center;gap:10px;margin:16px 0">
    <span style="color:#ccc;font-size:16px;font-weight:bold;white-space:nowrap">Fuel amount:</span>
    <span style="color:#aaa;font-size:16px">0%</span>
    <input type="range" id="fs-sim-flare" min="0" max="100" value="50" style="flex:1;margin:0">
    <span style="color:#ccc;font-size:16px">100%</span>
  </div>
  <div style="text-align:right;margin-top:10px">
    <button id="fs-sim-btn" style="background:#222;color:#ccc;border:1px solid #444;border-radius:4px;padding:4px 20px;cursor:pointer;font-size:16px">Pause simulation</button>
  </div>
</div>
<script>
(function () {
  var LUT_SIZE = 101, LUT_STEP = 10;
  var COLS = 20, ROWS = 6, N = COLS * ROWS;  // 120 pixels in a 20×6 grid
  var NEIGHBOURS = 1, OFFSET = 1, N_COUNT = 2;

  function lerpCurve(curve, x) {
    if (x <= curve[0][0]) return curve[0][1];
    if (x >= curve[curve.length - 1][0]) return curve[curve.length - 1][1];
    for (var i = 0; i < curve.length - 1; i++) {
      var x0 = curve[i][0], y0 = curve[i][1], x1 = curve[i+1][0], y1 = curve[i+1][1];
      if (x0 <= x && x <= x1) return y0 + (x - x0) / (x1 - x0) * (y1 - y0);
    }
    return curve[curve.length - 1][1];
  }
  function bakeLUT(curve) {
    var lut = new Float64Array(LUT_SIZE);
    for (var i = 0; i < LUT_SIZE; i++) lut[i] = lerpCurve(curve, i * LUT_STEP);
    return lut;
  }

  // Decay and drain tuned for screen — real firmware values are too aggressive for 120 pixels
  var decayLUT   = bakeLUT([[0,0.02],[333,0.15],[666,0.25],[1000,0.4]]);
  var diffLUT    = bakeLUT([[0,0.43],[333,0.41],[666,0.31],[1000,0.16]]);
  var drainLUT   = bakeLUT([[0,80],[333,500],[666,2000],[1000,4000]]);
  var flareHLUT  = bakeLUT([[0,200],[333,6100],[666,16300],[1000,20900]]);
  var flareChLUT = bakeLUT([[0,0.00171],[333,0.00027],[666,0.00016],[1000,0.00081]]);

  // Palette boosted for screen display — LEDs look vivid at low RGB values but screens don't
  var palR = [[0,0],[250,230],[500,250],[750,255],[1000,255]];
  var palG = [[0,0],[250,70],[500,180],[750,240],[1000,255]];
  var palB = [[0,0],[250,0],[500,40],[750,200],[1000,255]];
  var colLUT = [];
  for (var ci = 0; ci < LUT_SIZE; ci++) {
    var cx = ci * LUT_STEP;
    colLUT.push([Math.round(lerpCurve(palR,cx)), Math.round(lerpCurve(palG,cx)), Math.round(lerpCurve(palB,cx))]);
  }

  var cur = new Float64Array(N), lst = new Float64Array(N), hs = new Float64Array(N);
  // Fixed random offset per pixel (1, 2 or 3) — breaks up waves without being frantic
  var offsets = new Uint8Array(N);
  for (var oi = 0; oi < N; oi++) offsets[oi] = 1 + Math.floor(Math.random() * 3);

  function clamp(v, lo, hi) { return v < lo ? lo : v > hi ? hi : v; }

  function simulateFrame() {
    var i, j, heat, hi, prev, di, fuel, drain, headroom, totalDiff, hpn, ni, decAmt;
    var flareMul = parseInt(flareSlider.value) / 100 * 5; // 0–100% maps to 0–5x multiplier
    for (i = 0; i < N; i++) {
      heat = clamp(cur[i], 0, 1000);
      hi = Math.min(Math.floor(heat / LUT_STEP), LUT_SIZE - 1);
      if (Math.random() < flareChLUT[hi] * flareMul) {
        hs[i] += flareHLUT[Math.floor(Math.random() * LUT_SIZE)];
      }
      fuel = hs[i];
      if (fuel > 0) {
        drain = Math.min(fuel, drainLUT[hi]);
        headroom = 1000 - heat;
        if (drain > headroom) drain = headroom;
        if (drain > 0) { hs[i] -= drain; cur[i] = heat + drain; }
      }
      prev = Math.max(0, lst[i]);
      if (prev > 0) {
        di = Math.min(Math.floor(prev / LUT_STEP), LUT_SIZE - 1);
        totalDiff = prev * diffLUT[di];
        if (totalDiff > prev) totalDiff = prev;
        hpn = totalDiff / N_COUNT;
        var off = offsets[i];
        for (j = -NEIGHBOURS; j <= NEIGHBOURS; j++) {
          if (j === 0) { cur[i] -= totalDiff; }
          else { ni = i + j * off; if (ni >= 0 && ni < N) cur[ni] += hpn; }
        }
      }
    }
    for (i = 0; i < N; i++) cur[i] = clamp(cur[i], 0, 1000);
    for (i = 0; i < N; i++) {
      heat = clamp(cur[i], 0, 1000);
      di = Math.min(Math.floor(heat / LUT_STEP), LUT_SIZE - 1);
      decAmt = heat * decayLUT[di]; if (decAmt > heat) decAmt = heat;
      heat -= decAmt; cur[i] = heat; lst[i] = heat;
    }
  }

  var canvas   = document.getElementById('fs-sim-canvas');
  var btn      = document.getElementById('fs-sim-btn');
  var flareSlider = document.getElementById('fs-sim-flare');

  function draw() {
    var cw = canvas.offsetWidth || 600;
    var ps = Math.floor(cw / COLS);           // pixel size — square
    var gap = Math.max(2, Math.floor(ps * 0.08));
    canvas.width  = ps * COLS;
    canvas.height = ps * ROWS;
    var ctx = canvas.getContext('2d');
    ctx.fillStyle = '#111'; ctx.fillRect(0, 0, canvas.width, canvas.height);
    for (var i = 0; i < N; i++) {
      var row = Math.floor(i / COLS);
      var col = (row % 2 === 0) ? i % COLS : COLS - 1 - (i % COLS); // snake: odd rows reversed
      var hi = Math.min(Math.floor(clamp(cur[i], 0, 1000) / LUT_STEP), LUT_SIZE - 1);
      var c = colLUT[hi];
      ctx.fillStyle = 'rgb('+c[0]+','+c[1]+','+c[2]+')';
      ctx.fillRect(col * ps + gap, row * ps + gap, ps - gap * 2, ps - gap * 2);
    }
  }

  var paused = false, lastTs = 0, FRAME_MS = 1000 / 30;
  function loop(ts) {
    if (!paused) {
      if (ts - lastTs >= FRAME_MS) { simulateFrame(); draw(); lastTs = ts; }
      requestAnimationFrame(loop);
    }
  }
  btn.addEventListener('click', function () {
    paused = !paused;
    btn.textContent = paused ? 'Resume simulation' : 'Pause simulation';
    if (!paused) { lastTs = 0; requestAnimationFrame(loop); }
  });
  window.addEventListener('resize', draw);
  requestAnimationFrame(loop);
})();
</script>



With so many parameters to tune, it was becoming quite tedious to make changes as I had to update and upload the code every time I wanted to tweak something. To make life easier I made a HTML/JavaScript based configurator that let me design the various curves graphically and have them update on the LEDs in real time by sending the parameters to the board via the Web Serial API. This was quite fun to play with but I found that it was very easy to lose track of the settings that I liked while I fiddled, so I added functionality to save and load presets.

With the electronics and firmware in a good place, the last thing to do was give it a proper home. I modelled an enclosure in Onshape, designing it as a two-piece shell with curved edges so the top and bottom mate together cleanly. I added internal snap clips so it holds together without any screws or glue, and had it 3D printed. The final result looks really polished and gives the whole thing a much more finished feel.

![The finished board in its 3D printed enclosure](/assets/firestring_quarter.png)
*The finished board in its 3D printed enclosure*

What started as a tangle of unsoldered wires has ended up as something I'm genuinely proud of. The project has a custom board, firmware that runs well, and a configuration interface that lets me dial in exactly the effect I want. The whole thing feels complete and more importantly it's still sitting in our fireplace doing its job every evening.

## Making it your own

<video autoplay loop muted playsinline controls style="width:100%;border-radius:8px">
  <source src="/assets/firestring_video_long_square.mp4" type="video/mp4">
</video>

I had a great time working on this project, and if you want a share in some of the joy I derived from it the code and files are available on [GitHub](https://github.com/thip/firestring). That repo contains everything you need to build your own version of what I'm calling the 'Fire String' either by getting your own boards made or doing something similar to my early prototype version with an off-the-shelf RP2040 dev board like the Raspberry Pi Pico. Alternatively if you just want a complete polished product in your hands like the one I ended up with I've got a limited number available for sale on my store at [hortus.dev/products/fire-string](https://hortus.dev/products/fire-string) along with a bunch of other projects that I have worked on.
