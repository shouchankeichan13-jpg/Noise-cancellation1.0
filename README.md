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
    h1 {
      font-size: 1.4rem;
      font-weight: 700;
      text-align: center;
      margin-bottom: 8px;
      letter-spacing: -0.02em;
    }
    .subtitle {
      text-align: center;
      font-size: 0.85rem;
      color: #888;
      margin-bottom: 28px;
    }
    .warning {
      background: #2a1f1a;
      border: 1px solid #5a3a2a;
      color: #f0b080;
      font-size: 0.8rem;
      padding: 12px 14px;
      border-radius: 12px;
      margin-bottom: 24px;
      line-height: 1.5;
    }
    .control {
      margin-bottom: 20px;
    }
    label {
      display: block;
      font-size: 0.85rem;
      color: #aaa;
      margin-bottom: 8px;
    }
    input[type="range"] {
      width: 100%;
      height: 6px;
      -webkit-appearance: none;
      background: #333;
      border-radius: 3px;
      outline: none;
    }
    input[type="range"]::-webkit-slider-thumb {
      -webkit-appearance: none;
      width: 18px;
      height: 18px;
      background: #4f8cff;
      border-radius: 50%;
      cursor: pointer;
    }
    .value {
      text-align: right;
      font-size: 0.9rem;
      color: #4f8cff;
      margin-top: 4px;
    }
    .btn {
      width: 100%;
      padding: 16px;
      border: none;
      border-radius: 14px;
      font-size: 1.05rem;
      font-weight: 600;
      cursor: pointer;
      transition: all 0.2s;
    }
    .btn-start {
      background: linear-gradient(135deg, #4f8cff, #3a6fd8);
      color: white;
    }
    .btn-start:hover { filter: brightness(1.1); }
    .btn-stop {
      background: linear-gradient(135deg, #ff4f6a, #d83a52);
      color: white;
    }
    .status {
      text-align: center;
      margin-top: 18px;
      font-size: 0.9rem;
      color: #666;
    }
    .status.active {
      color: #4f8cff;
    }
    .status.error {
      color: #ff6b6b;
    }
    .meter {
      height: 6px;
      background: #222;
      border-radius: 3px;
      margin-top: 16px;
      overflow: hidden;
    }
    .meter-bar {
      height: 100%;
      width: 0%;
      background: linear-gradient(90deg, #4f8cff, #7b5cff);
      transition: width 0.05s;
    }
  </style>
</head>
<body>
  <div class="card">
    <h1>簡易ノイズキャンセリング</h1>
    <p class="subtitle">マイク → 位相反転 → スピーカー</p>

    <div class="warning">
      ⚠ マイクとスピーカーが近いとハウリングします。<br>
      最初は音量を小さくして、イヤホン推奨です。
    </div>

    <div class="control">
      <label>出力音量（ゲイン）</label>
      <input type="range" id="gain" min="0" max="100" value="25">
      <div class="value" id="gainValue">25%</div>
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
      if (gainNode) {
        // 位相反転なのでマイナス
        gainNode.gain.value = - (val / 100);
      }
    });

    async function start() {
      try {
        // マイク取得
        stream = await navigator.mediaDevices.getUserMedia({
          audio: {
            echoCancellation: false,   // ブラウザのエコーキャンセルを無効化
            noiseSuppression: false,
            autoGainControl: false,
            latency: 0
          },
          video: false
        });

        audioContext = new (window.AudioContext || window.webkitAudioContext)({
          latencyHint: 'interactive'
        });

        source = audioContext.createMediaStreamSource(stream);
        gainNode = audioContext.createGain();
        analyser = audioContext.createAnalyser();
        analyser.fftSize = 256;

        // 位相反転（ゲインをマイナスにする）
        gainNode.gain.value = - (gainSlider.value / 100);

        // マイク → ゲイン（反転） → 解析 → スピーカー
        source.connect(gainNode);
        gainNode.connect(analyser);
        analyser.connect(audioContext.destination);

        isRunning = true;
        toggleBtn.textContent = '停止';
        toggleBtn.classList.remove('btn-start');
        toggleBtn.classList.add('btn-stop');
        statusEl.textContent = '動作中…（マイクの音を反転して出力中）';
        statusEl.className = 'status active';

        // レベルメーター
        const dataArray = new Uint8Array(analyser.frequencyBinCount);
        function updateMeter() {
          if (!isRunning) return;
          analyser.getByteFrequencyData(dataArray);
          let sum = 0;
          for (let i = 0; i < dataArray.length; i++) sum += dataArray[i];
          const avg = sum / dataArray.length;
          meterBar.style.width = Math.min(100, avg * 1.5) + '%';
          animId = requestAnimationFrame(updateMeter);
        }
        updateMeter();

      } catch (err) {
        console.error(err);
        statusEl.textContent = 'エラー: マイクへのアクセスが拒否されました';
        statusEl.className = 'status error';
      }
    }

    function stop() {
      isRunning = false;
      if (animId) cancelAnimationFrame(animId);
      if (stream) {
        stream.getTracks().forEach(t => t.stop());
        stream = null;
      }
      if (audioContext) {
        audioContext.close();
        audioContext = null;
      }
      source = null;
      gainNode = null;
      analyser = null;

      toggleBtn.textContent = '開始';
      toggleBtn.classList.remove('btn-stop');
      toggleBtn.classList.add('btn-start');
      statusEl.textContent = '停止中';
      statusEl.className = 'status';
      meterBar.style.width = '0%';
    }

    toggleBtn.addEventListener('click', () => {
      if (isRunning) {
        stop();
      } else {
        start();
      }
    });

    // ページを閉じるときにクリーンアップ
    window.addEventListener('beforeunload', stop);
  </script>
</body>
</html>
