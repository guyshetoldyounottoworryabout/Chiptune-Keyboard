<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Chiptune Keyboard</title>
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
  <style>
    * { box-sizing: border-box; }
    :root { --pad: 16px; }
    html, body { margin: 0; padding: 0; font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; background:#fff; color:#111; }
    header { position: sticky; top: 0; background: #fff; padding: var(--pad); border-bottom: 1px solid #eee; }
    h1 { margin: 0 0 8px 0; font-size: 20px; }
    .controls { display: flex; flex-wrap: wrap; gap: 12px; align-items: center; }
    .controls label { display: flex; gap: 6px; align-items: center; font-size: 14px; }
    .keys { display: grid; grid-template-columns: repeat(12, minmax(44px, 1fr)); gap: 8px; padding: var(--pad); }
    .key { padding: 18px 0; text-align: center; border: 1px solid #ddd; border-radius: 10px; background: #fafafa; user-select: none; touch-action: none; }
    .key:active, .key.playing { transform: translateY(1px); border-color: #aaa; background: #f2f2f2; }
    footer { padding: 6px 16px 16px; color: #666; font-size: 12px; }
    .hint { opacity: 0.75; }
    .gate { position: fixed; inset: 0; display: grid; place-items: center; background: rgba(255,255,255,0.98); z-index: 9999; }
    .gate button { font-size: 18px; padding: 12px 18px; border-radius: 10px; border: 1px solid #ddd; background: #fff; color:#111; }
    .gate .hint { margin-top: 8px; text-align: center; color:#333; }
    @media (min-width: 780px){ .keys { grid-template-columns: repeat(24, minmax(44px, 1fr)); } }
  </style>
</head>
<body>
  <header>
    <h1>Chiptune Keyboard</h1>
    <div class="controls">
      <label>Wave
        <select id="wave">
          <option value="square">square</option>
          <option value="triangle">triangle</option>
          <option value="noise">noise</option>
        </select>
      </label>
      <label>Attack <input id="attack" type="range" min="0" max="0.2" step="0.005" value="0.01"></label>
      <label>Release <input id="release" type="range" min="0" max="1.0" step="0.01" value="0.25"></label>
      <label>Vibrato <input id="vibrato" type="range" min="0" max="10" step="0.1" value="0"></label>
      <label>Arp
        <select id="arp">
          <option value="off">off</option>
          <option value="maj">maj</option>
          <option value="min">min</option>
          <option value="p5">power</option>
        </select>
      </label>
      <label>BPM <input id="bpm" type="range" min="60" max="220" step="1" value="160"></label>
    </div>
  </header>

  <div id="keys" class="keys"></div>

  <div id="startGate" class="gate">
    <div style="text-align:center">
      <button id="startBtn">Tap to Start Audio</button>
      <p class="hint">If you see a white screen, this is it — tap the button to begin.</p>
    </div>
  </div>

  <footer>
    <p class="hint">Tap keys or use Z S X D C V G B H N J M (hardware keyboard).</p>
  </footer>

  <script>
    // --- JS inline so nothing can fail to load ---
    const AudioContext = window.AudioContext || window.webkitAudioContext;
    let ctx = null;

    const waveSel = document.getElementById('wave');
    const attackEl = document.getElementById('attack');
    const releaseEl = document.getElementById('release');
    const vibratoEl = document.getElementById('vibrato');
    const arpSel = document.getElementById('arp');
    const bpmEl = document.getElementById('bpm');

    const keysDiv = document.getElementById('keys');
    const startGate = document.getElementById('startGate');
    const startBtn = document.getElementById('startBtn');

    const notes = [
      {name:'C4', f:261.63}, {name:'C#4', f:277.18}, {name:'D4', f:293.66}, {name:'D#4', f:311.13},
      {name:'E4', f:329.63}, {name:'F4', f:349.23},  {name:'F#4', f:369.99}, {name:'G4', f:392.00},
      {name:'G#4', f:415.30},{name:'A4', f:440.00},  {name:'A#4', f:466.16},{name:'B4', f:493.88}
    ];

    function ensureCtx() {
      if (!ctx) ctx = new AudioContext({ latencyHint: 'interactive' });
    }

    startBtn.addEventListener('click', async () => {
      ensureCtx();
      await ctx.resume();
      startGate.style.display = 'none';
    });

    notes.forEach((n) => {
      const btn = document.createElement('button');
      btn.className = 'key';
      btn.textContent = n.name;
      btn.onpointerdown = () => startNote(n.f, btn);
      btn.onpointerup = () => stopNote(btn);
      btn.onpointerleave = () => stopNote(btn);
      keysDiv.appendChild(btn);
    });

    const active = new Map();

    function makeNoiseBuffer() {
      ensureCtx();
      const buffer = ctx.createBuffer(1, ctx.sampleRate * 1, ctx.sampleRate);
      const data = buffer.getChannelData(0);
      for (let i=0;i<data.length;i++) data[i] = Math.random()*2-1;
      return buffer;
    }
    const noiseBufferLazy = { buffer: null };

    function startNote(freq, el){
      ensureCtx();
      const now = ctx.currentTime;
      const a = parseFloat(attackEl.value);
      const r = parseFloat(releaseEl.value);
      const vibAmt = parseFloat(vibratoEl.value);
      const wave = waveSel.value;
      const arp = arpSel.value;
      const bpm = parseInt(bpmEl.value, 10);
      const stepDur = 60 / bpm / 4;

      if (active.has(el)) return;

      const gain = ctx.createGain();
      gain.gain.setValueAtTime(0, now);
      gain.gain.linearRampToValueAtTime(0.9, now + a);
      gain.connect(ctx.destination);

      let osc, src, vibratoOsc, vibratoGain, seqTimer = null;

      if (wave === 'noise') {
        if (!noiseBufferLazy.buffer) noiseBufferLazy.buffer = makeNoiseBuffer();
        src = ctx.createBufferSource();
        src.buffer = noiseBufferLazy.buffer;
        src.loop = true;
        src.connect(gain);
        src.start();
      } else {
        osc = ctx.createOscillator();
        osc.type = wave;
        osc.frequency.setValueAtTime(freq, now);

        if (vibAmt > 0) {
          vibratoOsc = ctx.createOscillator();
          vibratoGain = ctx.createGain();
          vibratoOsc.frequency.value = 6;
          vibratoGain.gain.value = vibAmt;
          vibratoOsc.connect(vibratoGain);
          vibratoGain.connect(osc.frequency);
          vibratoOsc.start();
        }

        osc.connect(gain);
        osc.start();

        if (arp !== 'off') {
          const ratios = { maj:[1,1.25,1.5,2], min:[1,1.2,1.5,2], p5:[1,1.5,2,3] }[arp];
          let step = 0;
          seqTimer = setInterval(() => {
            const next = freq * ratios[step % ratios.length];
            osc.frequency.setTargetAtTime(next, ctx.currentTime, 0.005);
            step++;
          }, stepDur * 1000);
        }
      }

      el.classList.add('playing');
      active.set(el, {gain, osc, src, vibratoOsc, vibratoGain, release:r, seqTimer});
    }

    function stopNote(el){
      const node = active.get(el);
      if (!node) return;
      const now = ctx.currentTime;
      const end = now + node.release;

      node.gain.gain.cancelScheduledValues(now);
      node.gain.gain.setValueAtTime(node.gain.gain.value, now);
      node.gain.gain.linearRampToValueAtTime(0.0001, end);

      if (node.seqTimer) clearInterval(node.seqTimer);
      if (node.osc) { if (node.vibratoOsc) node.vibratoOsc.stop(end); node.osc.stop(end); }
      if (node.src) node.src.stop(end);

      setTimeout(() => {
        node.gain.disconnect();
        if (node.osc) node.osc.disconnect();
        if (node.vibratoGain) node.vibratoGain.disconnect();
        if (node.vibratoOsc) node.vibratoOsc.disconnect();
      }, node.release * 1000 + 60);

      el.classList.remove('playing');
      active.delete(el);
    }

    const map = ['KeyZ','KeyS','KeyX','KeyD','KeyC','KeyV','KeyG','KeyB','KeyH','KeyN','KeyJ','KeyM'];
    window.addEventListener('keydown', (e)=>{
      const i = map.indexOf(e.code);
      if (i>=0) keysDiv.children[i].dispatchEvent(new PointerEvent('pointerdown'));
    });
    window.addEventListener('keyup', (e)=>{
      const i = map.indexOf(e.code);
      if (i>=0) keysDiv.children[i].dispatchEvent(new PointerEvent('pointerup'));
    });
  </script>
</body>
</html>
