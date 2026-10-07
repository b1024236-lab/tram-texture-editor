<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <!-- iPhone表示最適化: ズーム防止 & フルスクリーン対応 -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
  <title>JATCHU テクスチャ合成エンジン</title>
  <style>
    * { box-sizing: border-box; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      margin: 0;
      padding: 12px;
      background: #f2f2f7;
      color: #1c1c1e;
      overscroll-behavior: none;
    }
    h2 { 
      font-size: 17px; 
      margin: 4px 0 10px 0; 
      text-align: center; 
      font-weight: 700;
    }

    #canvas-container {
      width: 100%;
      background: #111;
      border-radius: 12px;
      overflow: hidden;
      box-shadow: 0 4px 12px rgba(0,0,0,0.12);
      margin-bottom: 12px;
    }
    #preview-canvas { 
      width: 100%; 
      height: auto; 
      display: block; 
      touch-action: none; /* スマホ画面のスクロール防止 */
      cursor: move; 
    }

    .controls {
      background: #ffffff;
      padding: 14px;
      border-radius: 14px;
      display: flex;
      flex-direction: column;
      gap: 10px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.05);
    }
    .control-group {
      display: flex;
      flex-direction: column;
      gap: 4px;
    }
    label { font-size: 11px; font-weight: bold; color: #8e8e93; }

    input[type="text"], select, input[type="file"] {
      font-size: 14px;
      padding: 8px 10px;
      border: 1px solid #d1d1d6;
      border-radius: 8px;
      background: #f2f2f7;
      width: 100%;
      -webkit-appearance: none;
    }

    .row { display: flex; gap: 8px; }
    .row > * { flex: 1; }

    .slider-box {
      display: flex;
      align-items: center;
      justify-content: space-between;
      background: #f2f2f7;
      padding: 8px 10px;
      border-radius: 8px;
    }
    .slider-box label { font-size: 11px; margin: 0; color: #3a3a3c; }
    .slider-box input[type="range"] { width: 50%; }
    .slider-val { font-size: 11px; font-weight: bold; color: #007aff; min-width: 38px; text-align: right; }

    .btn-row {
      display: flex;
      gap: 8px;
      margin-top: 4px;
    }
    button {
      flex: 1;
      background: #007aff;
      color: white;
      border: none;
      padding: 10px;
      font-size: 14px;
      font-weight: bold;
      border-radius: 8px;
      cursor: pointer;
    }
    button:active { opacity: 0.7; }
    button.secondary {
      background: #8e8e93;
    }
  </style>
</head>
<body>

  <h2>市電ラッピング テクスチャ合成</h2>

  <!-- プレビューCanvas -->
  <div id="canvas-container">
    <canvas id="preview-canvas" width="2048" height="1024"></canvas>
  </div>

  <div class="controls">
    <!-- 背景プリセット & ファイル挿入 -->
    <div class="row">
      <div class="control-group">
        <label>🎨 背景プリセット</label>
        <select id="bg-style">
          <option value="wood">高級木目調</option>
          <option value="red-stripe">赤グラデーション</option>
          <option value="gold-pattern">ゴールド和柄</option>
        </select>
      </div>

      <div class="control-group">
        <label>📷 背景画像挿入</label>
        <input type="file" id="bg-file" accept="image/*">
      </div>
    </div>

    <!-- タイトル ＆ サブタイトル -->
    <div class="row">
      <div class="control-group">
        <label>タイトル</label>
        <input type="text" id="main-title" value="花もめん">
      </div>
      <div class="control-group">
        <label>サブタイトル</label>
        <input type="text" id="sub-title" value="黄金塩ラーメン 800円">
      </div>
    </div>

    <!-- 背景拡大率 -->
    <div class="slider-box">
      <label>背景拡大率</label>
      <input type="range" id="bg-scale" min="10" max="300" value="100">
      <span id="bg-scale-val" class="slider-val">100%</span>
    </div>

    <!-- 文字サイズ -->
    <div class="row">
      <div class="slider-box">
        <label>タイトル大</label>
        <input type="range" id="title-size" min="40" max="220" value="130">
        <span id="title-size-val" class="slider-val">130px</span>
      </div>
      <div class="slider-box">
        <label>サブ大</label>
        <input type="range" id="sub-size" min="20" max="100" value="48">
        <span id="sub-size-val" class="slider-val">48px</span>
      </div>
    </div>

    <div class="btn-row">
      <button id="btn-save">画像を保存</button>
      <button id="btn-reset" class="secondary">リセット</button>
    </div>
  </div>

  <script>
    const canvas = document.getElementById('preview-canvas');
    const ctx = canvas.getContext('2d');

    const bgScaleInput = document.getElementById('bg-scale');
    const titleSizeInput = document.getElementById('title-size');
    const subSizeInput = document.getElementById('sub-size');
    
    const bgScaleVal = document.getElementById('bg-scale-val');
    const titleSizeVal = document.getElementById('title-size-val');
    const subSizeVal = document.getElementById('sub-size-val');
    
    const bgFileInput = document.getElementById('bg-file');
    const bgStyleSelect = document.getElementById('bg-style');

    let customBgImage = null;
    let bgX = 0, bgY = 0;
    let titleX = 1024, titleY = 730;
    let subX = 1024, subY = 885;

    let activeDragTarget = null;
    let dragOffsetX = 0, dragOffsetY = 0;

    let titleBoundingBox = { x: 0, y: 0, w: 0, h: 0 };
    let subBoundingBox = { x: 0, y: 0, w: 0, h: 0 };

    // --- 背景描画 ---
    function drawBackgroundPattern(theme) {
      const w = canvas.width, h = canvas.height;
      if (customBgImage) {
        const scale = parseInt(bgScaleInput.value, 10) / 100;
        ctx.drawImage(customBgImage, bgX, bgY, customBgImage.width * scale, customBgImage.height * scale);
        return;
      }
      if (theme === 'wood') {
        const grad = ctx.createLinearGradient(0, 0, 0, h);
        grad.addColorStop(0, '#2b1704'); grad.addColorStop(0.5, '#4a2a0c'); grad.addColorStop(1, '#1a0d02');
        ctx.fillStyle = grad; ctx.fillRect(0, 0, w, h);
        ctx.strokeStyle = 'rgba(0, 0, 0, 0.2)';
        ctx.lineWidth = 4;
        for (let i = 0; i < h; i += 16) {
          ctx.beginPath();
          ctx.moveTo(0, i + Math.sin(i * 0.05) * 10);
          ctx.bezierCurveTo(w * 0.3, i - 20, w * 0.7, i + 20, w, i);
          ctx.stroke();
        }
      } else if (theme === 'red-stripe') {
        const grad = ctx.createLinearGradient(0, 0, 0, h);
        grad.addColorStop(0, '#8b0000'); grad.addColorStop(0.5, '#d32f2f'); grad.addColorStop(1, '#4a0000');
        ctx.fillStyle = grad; ctx.fillRect(0, 0, w, h);
      } else if (theme === 'gold-pattern') {
        const grad = ctx.createRadialGradient(w/2, h/2, 100, w/2, h/2, w/1.2);
        grad.addColorStop(0, '#ffe082'); grad.addColorStop(0.5, '#ffb300'); grad.addColorStop(1, '#ff6f00');
        ctx.fillStyle = grad; ctx.fillRect(0, 0, w, h);
      }
    }

    // --- テキスト描画 ---
    function drawContent(titleText, subtitleText) {
      const titleSize = parseInt(titleSizeInput.value, 10);
      const subSize = parseInt(subSizeInput.value, 10);

      // タイトル
      ctx.save();
      ctx.font = `bold ${titleSize}px "Yu Mincho", "Hiragino Mincho ProN", serif`;
      ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
      const titleMetrics = ctx.measureText(titleText);
      const titleW = titleMetrics.width + 40, titleH = titleSize * 1.2;
      titleBoundingBox = { x: titleX - titleW / 2, y: titleY - titleH / 2, w: titleW, h: titleH };

      ctx.fillStyle = '#ffffff'; ctx.strokeStyle = '#000000';
      ctx.lineWidth = Math.max(6, titleSize * 0.14);
      ctx.strokeText(titleText, titleX, titleY);
      ctx.fillText(titleText, titleX, titleY);
      ctx.restore();

      // サブタイトル
      ctx.save();
      ctx.font = `bold ${subSize}px sans-serif`;
      const subMetrics = ctx.measureText(subtitleText);
      const bannerW = Math.max(subMetrics.width + 120, 300), bannerH = subSize * 1.8;
      subBoundingBox = { x: subX - bannerW / 2, y: subY - bannerH / 2, w: bannerW, h: bannerH };

      ctx.fillStyle = 'rgba(255, 248, 225, 0.9)';
      ctx.fillRect(subBoundingBox.x, subBoundingBox.y, bannerW, bannerH);
      ctx.fillStyle = '#d32f2f'; ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
      ctx.fillText(subtitleText, subX, subY);
      ctx.restore();

      // 左右の提灯を両方とも「ジャッチュ」に変更
      drawLantern(80, 560, 'ジャッチュ');
      drawLantern(canvas.width - 200, 560, 'ジャッチュ');
    }

    // 提灯描画（文字数に合わせて自動でサイズとピッチを最適化）
    function drawLantern(x, y, text) {
      ctx.save();
      const lanternH = 240;
      const lanternW = 110;

      ctx.fillStyle = '#d32f2f';
      ctx.beginPath();
      ctx.roundRect(x, y, lanternW, lanternH, 25);
      ctx.fill();
      ctx.strokeStyle = '#000000';
      ctx.lineWidth = 5;
      ctx.stroke();

      // 5文字「ジャッチュ」がきれいに収まるフォントサイズと行間
      const fontSize = text.length >= 5 ? 32 : 40;
      const step = text.length >= 5 ? 38 : 45;
      const startY = y + (lanternH - ((text.length - 1) * step)) / 2;

      ctx.fillStyle = '#ffffff';
      ctx.font = `bold ${fontSize}px sans-serif`;
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';

      for (let i = 0; i < text.length; i++) {
        ctx.fillText(text[i], x + lanternW / 2, startY + (i * step));
      }
      ctx.restore();
    }

    // --- 窓・ドアマスク描画 ---
    function drawWindowAndDoorMask() {
      const w = canvas.width, windowY = 220, windowH = 320;
      ctx.save();
      const windowCount = 6, startX = 300, totalWidth = w - 600;
      const windowW = (totalWidth - (windowCount - 1) * 30) / windowCount;

      for (let i = 0; i < windowCount; i++) {
        const x = startX + i * (windowW + 30);
        ctx.fillStyle = '#111111';
        ctx.fillRect(x - 6, windowY - 6, windowW + 12, windowH + 12);
        const glassGrad = ctx.createLinearGradient(x, windowY, x + windowW, windowY + windowH);
        glassGrad.addColorStop(0, 'rgba(180, 220, 240, 0.65)');
        glassGrad.addColorStop(0.5, 'rgba(120, 170, 200, 0.45)');
        glassGrad.addColorStop(1, 'rgba(200, 240, 255, 0.7)');
        ctx.fillStyle = glassGrad;
        ctx.fillRect(x, windowY, windowW, windowH);
      }
      ctx.restore();
    }

    function generateTexture() {
      bgScaleVal.textContent = `${bgScaleInput.value}%`;
      titleSizeVal.textContent = `${titleSizeInput.value}px`;
      subSizeVal.textContent = `${subSizeInput.value}px`;

      drawBackgroundPattern(bgStyleSelect.value);
      drawContent(document.getElementById('main-title').value, document.getElementById('sub-title').value);
      drawWindowAndDoorMask();
    }

    // --- 座標変換 ---
    function getCanvasCoords(e) {
      const rect = canvas.getBoundingClientRect();
      const scaleX = canvas.width / rect.width;
      const scaleY = canvas.height / rect.height;
      return {
        x: (e.clientX - rect.left) * scaleX,
        y: (e.clientY - rect.top) * scaleY
      };
    }

    function isInside(p, box) {
      return p.x >= box.x && p.x <= box.x + box.w && p.y >= box.y && p.y <= box.y + box.h;
    }

    canvas.addEventListener('pointerdown', (e) => {
      const coords = getCanvasCoords(e);
      if (isInside(coords, subBoundingBox)) {
        activeDragTarget = 'sub'; dragOffsetX = coords.x - subX; dragOffsetY = coords.y - subY;
      } else if (isInside(coords, titleBoundingBox)) {
        activeDragTarget = 'title'; dragOffsetX = coords.x - titleX; dragOffsetY = coords.y - titleY;
      } else if (customBgImage) {
        activeDragTarget = 'bg'; dragOffsetX = coords.x - bgX; dragOffsetY = coords.y - bgY;
      }
      if (activeDragTarget) canvas.setPointerCapture(e.pointerId);
    });

    canvas.addEventListener('pointermove', (e) => {
      if (!activeDragTarget) return;
      const coords = getCanvasCoords(e);
      const newX = Math.round(coords.x - dragOffsetX);
      const newY = Math.round(coords.y - dragOffsetY);

      if (activeDragTarget === 'title') { titleX = newX; titleY = newY; }
      else if (activeDragTarget === 'sub') { subX = newX; subY = newY; }
      else if (activeDragTarget === 'bg') { bgX = newX; bgY = newY; }
      generateTexture();
    });

    function stopDrag(e) {
      if (activeDragTarget) {
        activeDragTarget = null;
        try { canvas.releasePointerCapture(e.pointerId); } catch(err) {}
      }
    }
    canvas.addEventListener('pointerup', stopDrag);
    canvas.addEventListener('pointercancel', stopDrag);

    // イベント
    bgFileInput.addEventListener('change', (e) => {
      const file = e.target.files[0];
      if (file) {
        const img = new Image();
        img.onload = () => {
          customBgImage = img;
          const scaleX = canvas.width / img.width, scaleY = canvas.height / img.height;
          const fitScale = Math.max(scaleX, scaleY);
          bgScaleInput.value = Math.round(fitScale * 100);
          bgX = (canvas.width - img.width * fitScale) / 2;
          bgY = (canvas.height - img.height * fitScale) / 2;
          generateTexture();
        };
        img.src = URL.createObjectURL(file);
      }
    });

    bgStyleSelect.addEventListener('change', () => { customBgImage = null; generateTexture(); });
    bgScaleInput.addEventListener('input', generateTexture);
    titleSizeInput.addEventListener('input', generateTexture);
    subSizeInput.addEventListener('input', generateTexture);
    document.getElementById('main-title').addEventListener('input', generateTexture);
    document.getElementById('sub-title').addEventListener('input', generateTexture);

    document.getElementById('btn-reset').addEventListener('click', () => {
      titleSizeInput.value = 130; subSizeInput.value = 48;
      titleX = 1024; titleY = 730; subX = 1024; subY = 885;
      bgScaleInput.value = 100; bgX = 0; bgY = 0; customBgImage = null;
      generateTexture();
    });

    // 画像保存ボタン
    document.getElementById('btn-save').addEventListener('click', () => {
      const link = document.createElement('a');
      link.download = 'jatchu_tram_texture.png';
      link.href = canvas.toDataURL('image/png');
      link.click();
    });

    generateTexture();
  </script>
</body>
</html>
```

### 変更点
- 車体両サイドの赤提灯を、左右ともに **「ジャッチュ」** 表示へ変更しました。
- 5文字（ジ・ャ・ッ・チ・ュ）が枠内にバランスよく収まるよう、提灯の高さを240px、フォントサイズを32px、行間ピッチを38pxに自動最適化しています。
