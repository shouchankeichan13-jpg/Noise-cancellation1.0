<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>簡易ノイズキャンセリング（逆位相）</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Hiragino Sans", sans-serif;
      background: #0f0f13;
      color: #e8e8e8;
      min-height: 100vh;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      padding: 20px;
    }
    .card {
      background: #1a1a22;
      border-radius: 20px;
      padding: 32px 28px;
      max-width: 420px;
      width: 100%;
      box-shadow: 0 12px 40px rgba(0,0,0,0.5);
      border: 1px solid #2a2a35;
    }
    h1 { font-size: 1.35rem; font-weight: 700; text-align: center; margin-bottom: 6px; }
    .subtitle { text-align: center; font-size: 0.85rem; color: #888; margin-bottom: 22px; }
    .warning {
      background: #2a1f1a; border: 1px solid #5a3a2a; color: #f0b080;
      font-size: 0.8rem; padding: 12px 14px; border-radius: 12px;
      margin-bottom: 22px; line-height: 1.55;
    }
    .control { margin-bottom: 18px; }
    label { display: block; font-size: 0.85rem; color: #aaa; margin-bottom: 8px; }
    input[type="range"] {
      width: 100%; height: 6px; -webkit-appearance: none;
      background: #333; border-radius: 3px; outline: none;
    }
    input[type="range"]::-webkit-slider-thumb {
      -webkit-appearance: none; width: 18px; height: 18px;
      background: #4f8cff; border-radius: 50%; cursor: pointer;
    }
    .value { text-align: right; font-size: 0.9rem; color: #4f8cff; margin-top: 4px; }
    .btn {
      width: 100%; padding: 16px; border: none; border-radius: 14px;
      font-size: 1.05rem; font-weight: 600; cursor: pointer; transition: all 0.2s;
    }
    .btn-start { background: linear-gradient(135deg, #4f8cff, #3a6fd8); color: white; }
    .btn-stop  { background: linear-gradient(135deg, #ff4f6a, #d83a52); color: white; }
    .status { text-align: center; margin-top: 16px; font-size: 0.9rem; color: #666; }
    .status.active { color: #4f8cff; }
    .status.error  { color: #ff6b6b; }
    .meter { height: 6px; background: #222; border-radius: 3px; margin-top: 14px; overflow: hidden; }
    .meter-bar { height: 100%; width: 0%; background: linear-gradient(90deg, #4f8cff, #7b5cff); }
  </style>
</head>
<body>
  <div class="card">
    <h1>簡易ノイズキャンセリング</h1>
    <p class="subtitle">マイク → 位相反転 → スピーカー</p>

    <div class="warning">
      ⚠ 「普通に聞こえる」のは正常です。<br>
      位相反転だけでは耳にはほとんど変化がわかりません。<br>
      ヘッドホン＋マイクを耳の近くに置いて試してください。
    </div>

    <div class="control">
      <label>出力音量（ゲイン）</label>
      <input type="range" id="gain" min="0" max="80" value="20">
      <div class="value" id="gainValue">20%</div>
    </div>

    <button class="btn btn-start" id="toggleBtn">開始</button>
    <div class="status" id="status">停止中</div>
    <div class="meter"><div class="meter-bar" id="meter"></div></div>
  </div>

  <script>
    let audioContext = null;
    let stream = null;
    let source = null;
    let gainNode = null;
    let analyser = null;
    let isRunning = false;
    let animId = null;

    const toggleBtn = document.getElementById('toggleBtn');
    const statusEl = document.getElementById('status');
    const gainSlider = document.getElementById('gain');
    const gainValue = document.getElementById('gainValue');
    const meterBar = document.getElementById('meter');

    gainSlider.addEventListener('input', () => {
      const val = gainSlider.value;
      gainValue.textContent = val + '%';
      if (gainNode) gainNode.gain.value = -(val / 100);
    });

    async function start() {
      try {
        stream = await navigator.mediaDevices.getUserMedia({
          audio: {
            echoCancellation: false,
            noiseSuppression: false,
            autoGainControl: false,
            channelCount: 1,
            sampleRate: 48000
          }
        });

        audioContext = new (window.AudioContext || window.webkitAudioContext)({
          latencyHint: 'interactive',
          sampleRate: 48000
        });

        // 可能な限り低遅延に
        await audioContext.resume();

        source = audioContext.createMediaStreamSource(stream);
        gainNode = audioContext.createGain();
        analyser = audioContext.createAnalyser();
        analyser.fftSize = 256;

        gainNode.gain.value = -(gainSlider.value / 100);

        source.connect(gainNode);
        gainNode.connect(analyser);
        analyser.connect(audioContext.destination);

        isRunning = true;
        toggleBtn.textContent = '停止';
        toggleBtn.classList.replace('btn-start', 'btn-stop');
        statusEl.textContent = '動作中（位相反転出力中）';
        statusEl.className = 'status active';

        const dataArray = new Uint8Array(analyser.frequencyBinCount);
        function updateMeter() {
          if (!isRunning) return;
          analyser.getByteFrequencyData(dataArray);
          let sum = 0;
          for (let i = 0; i < dataArray.length; i++) sum += dataArray[i];
          meterBar.style.width = Math.min(100, (sum / dataArray.length) * 1.8) + '%';
          animId = requestAnimationFrame(updateMeter);
        }
        updateMeter();

      } catch (err) {
        console.error(err);
        statusEl.textContent = 'マイクへのアクセスが拒否されました';
        statusEl.className = 'status error';
      }
    }

    function stop() {
      isRunning = false;
      if (animId) cancelAnimationFrame(animId);
      if (stream) stream.getTracks().forEach(t => t.stop());
      if (audioContext) audioContext.close();
      stream = audioContext = source = gainNode = analyser = null;

      toggleBtn.textContent = '開始';
      toggleBtn.classList.replace('btn-stop', 'btn-start');
      statusEl.textContent = '停止中';
      statusEl.className = 'status';
      meterBar.style.width = '0%';
    }

    toggleBtn.addEventListener('click', () => isRunning ? stop() : start());
    window.addEventListener('beforeunload', stop);
  </script>
</body>
</html>
