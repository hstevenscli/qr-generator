<script>
  import { onDestroy } from "svelte";
  import QRCode from "qrcode";

  const FINDER_PRESETS = [
    {
      id: "classic",
      label: "Classic",
      moduleStyle: "square",
      finderOuterStyle: "square",
      finderInnerStyle: "square",
      finderOuterColor: "#000000",
      finderInnerColor: "#000000"
    },
    {
      id: "soft",
      label: "Soft rounded",
      moduleStyle: "rounded",
      finderOuterStyle: "rounded",
      finderInnerStyle: "rounded",
      finderOuterColor: "#000000",
      finderInnerColor: "#000000"
    },
    {
      id: "rounded_eyes",
      label: "Rounded eyes",
      moduleStyle: "square",
      finderOuterStyle: "rounded",
      finderInnerStyle: "rounded",
      finderOuterColor: "#000000",
      finderInnerColor: "#000000"
    },
    {
      id: "circle_eyes",
      label: "Circular eyes",
      moduleStyle: "square",
      finderOuterStyle: "circle",
      finderInnerStyle: "circle",
      finderOuterColor: "#000000",
      finderInnerColor: "#000000"
    },
    {
      id: "dots",
      label: "Dot matrix",
      moduleStyle: "circle",
      finderOuterStyle: "circle",
      finderInnerStyle: "circle",
      finderOuterColor: "#000000",
      finderInnerColor: "#000000"
    },
    {
      id: "contrast",
      label: "Two-tone eyes",
      moduleStyle: "rounded",
      finderOuterStyle: "rounded",
      finderInnerStyle: "circle",
      finderOuterColor: "#111827",
      finderInnerColor: "#6366f1"
    }
  ];

  let text = "https://example.com";
  let eyePresetId = "classic";
  let size = 512;
  let fgColor = "#000000";
  let bgColor = "#ffffff";
  let useGradient = false;
  let gradientStart = "#4f46e5";
  let gradientEnd = "#06b6d4";
  let gradientDirection = "top_to_bottom";
  let logoScale = 0.2;
  let logoFile = null;
  let moduleStyle = "square";
  let finderOuterStyle = "square";
  let finderInnerStyle = "square";
  let finderOuterColor = "#000000";
  let finderInnerColor = "#000000";

  let loading = false;
  let error = "";
  let qrImageUrl = "";
  let autoGenerateTimer = null;
  let generationId = 0;

  function applyFinderPreset(id) {
    const preset = FINDER_PRESETS.find((p) => p.id === id);
    if (!preset) return;
    eyePresetId = id;
    moduleStyle = preset.moduleStyle;
    finderOuterStyle = preset.finderOuterStyle;
    finderInnerStyle = preset.finderInnerStyle;
    finderOuterColor = preset.finderOuterColor;
    finderInnerColor = preset.finderInnerColor;
  }

  function markFinderCustom() {
    eyePresetId = "custom";
  }

  async function generateQR() {
    const currentGeneration = ++generationId;
    error = "";
    loading = true;

    try {
      if (!text.trim()) {
        qrImageUrl = "";
        return;
      }

      const canvas = document.createElement("canvas");
      canvas.width = size;
      canvas.height = size;
      const ctx = canvas.getContext("2d");
      if (!ctx) throw new Error("Could not create canvas context.");

      renderStyledQRCode(ctx, size);

      if (logoFile) {
        await drawLogoOverlay(ctx, canvas.width, canvas.height, logoFile, logoScale);
      }

      if (currentGeneration === generationId) {
        qrImageUrl = canvas.toDataURL("image/png");
      }
    } catch (err) {
      error = err?.message || "Failed to generate QR code.";
    } finally {
      if (currentGeneration === generationId) {
        loading = false;
      }
    }
  }

  function renderStyledQRCode(ctx, canvasSize) {
    const qrData = QRCode.create(text, { errorCorrectionLevel: "M" });
    const count = qrData.modules.size;
    const quietZone = 4;
    const totalModules = count + quietZone * 2;
    const modulePx = Math.max(1, Math.floor(canvasSize / totalModules));
    const drawSize = modulePx * totalModules;
    const offset = Math.floor((canvasSize - drawSize) / 2);

    ctx.fillStyle = bgColor;
    ctx.fillRect(0, 0, canvasSize, canvasSize);

    for (let y = 0; y < count; y++) {
      for (let x = 0; x < count; x++) {
        if (!qrData.modules.get(x, y)) continue;
        if (isFinderModule(x, y, count)) continue;

        const px = offset + (x + quietZone) * modulePx;
        const py = offset + (y + quietZone) * modulePx;
        const color = useGradient
          ? toCssRgb(gradientColorAt(px, py, canvasSize, canvasSize, gradientStart, gradientEnd, gradientDirection))
          : fgColor;
        drawCell(ctx, px, py, modulePx, moduleStyle, color);
      }
    }

    const topLeft = offset + quietZone * modulePx;
    const topRight = offset + (quietZone + count - 7) * modulePx;
    const bottom = offset + (quietZone + count - 7) * modulePx;

    drawFinder(ctx, topLeft, topLeft, modulePx);
    drawFinder(ctx, topRight, topLeft, modulePx);
    drawFinder(ctx, topLeft, bottom, modulePx);
  }

  function drawFinder(ctx, x, y, modulePx) {
    drawCell(ctx, x, y, modulePx * 7, finderOuterStyle, finderOuterColor);
    drawCell(ctx, x + modulePx, y + modulePx, modulePx * 5, finderOuterStyle, bgColor);
    drawCell(ctx, x + modulePx * 2, y + modulePx * 2, modulePx * 3, finderInnerStyle, finderInnerColor);
  }

  function drawCell(ctx, x, y, sizePx, style, fillStyle) {
    ctx.fillStyle = fillStyle;

    if (style === "circle") {
      ctx.beginPath();
      ctx.arc(x + sizePx / 2, y + sizePx / 2, sizePx / 2, 0, Math.PI * 2);
      ctx.fill();
      return;
    }

    if (style === "rounded") {
      roundedRectPath(ctx, x, y, sizePx, sizePx, Math.max(1, sizePx * 0.22));
      ctx.fill();
      return;
    }

    ctx.fillRect(x, y, sizePx, sizePx);
  }

  function roundedRectPath(ctx, x, y, width, height, radius) {
    const r = Math.min(radius, width / 2, height / 2);
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.lineTo(x + width - r, y);
    ctx.quadraticCurveTo(x + width, y, x + width, y + r);
    ctx.lineTo(x + width, y + height - r);
    ctx.quadraticCurveTo(x + width, y + height, x + width - r, y + height);
    ctx.lineTo(x + r, y + height);
    ctx.quadraticCurveTo(x, y + height, x, y + height - r);
    ctx.lineTo(x, y + r);
    ctx.quadraticCurveTo(x, y, x + r, y);
    ctx.closePath();
  }

  function isFinderModule(x, y, count) {
    return (x <= 6 && y <= 6) || (x >= count - 7 && y <= 6) || (x <= 6 && y >= count - 7);
  }

  function gradientColorAt(x, y, width, height, startHex, endHex, direction) {
    const start = hexToRgb(startHex);
    const end = hexToRgb(endHex);
    const nx = width <= 1 ? 0 : x / (width - 1);
    const ny = height <= 1 ? 0 : y / (height - 1);

    let t = ny;
    switch (direction) {
      case "left_to_right":
        t = nx;
        break;
      case "right_to_left":
        t = 1 - nx;
        break;
      case "top_to_bottom":
        t = ny;
        break;
      case "bottom_to_top":
        t = 1 - ny;
        break;
      case "top_left_to_bottom_right":
        t = (nx + ny) / 2;
        break;
      case "bottom_right_to_top_left":
        t = 1 - (nx + ny) / 2;
        break;
      case "top_right_to_bottom_left":
        t = ((1 - nx) + ny) / 2;
        break;
      case "bottom_left_to_top_right":
        t = (nx + (1 - ny)) / 2;
        break;
      default:
        t = ny;
    }

    const clamped = clamp01(t);
    return {
      r: Math.round(start.r * (1 - clamped) + end.r * clamped),
      g: Math.round(start.g * (1 - clamped) + end.g * clamped),
      b: Math.round(start.b * (1 - clamped) + end.b * clamped)
    };
  }

  async function drawLogoOverlay(ctx, width, height, file, scale) {
    const logo = await fileToImage(file);
    const clampedScale = Math.min(0.35, Math.max(0.1, Number(scale) || 0.2));
    const logoSize = Math.max(1, Math.round(width * clampedScale));
    const logoX = Math.round((width - logoSize) / 2);
    const logoY = Math.round((height - logoSize) / 2);
    const pad = Math.max(8, Math.round(logoSize * 0.12));

    ctx.fillStyle = "rgba(255, 255, 255, 0.9)";
    ctx.fillRect(logoX - pad, logoY - pad, logoSize + pad * 2, logoSize + pad * 2);
    ctx.drawImage(logo, logoX, logoY, logoSize, logoSize);
  }

  function fileToImage(file) {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => {
        const img = new Image();
        img.onload = () => resolve(img);
        img.onerror = () => reject(new Error("Invalid logo image."));
        img.src = reader.result;
      };
      reader.onerror = () => reject(new Error("Could not read logo file."));
      reader.readAsDataURL(file);
    });
  }

  function hexToRgb(hex) {
    const sanitized = String(hex || "")
      .trim()
      .replace("#", "");
    if (sanitized.length !== 6) return { r: 0, g: 0, b: 0 };

    const r = Number.parseInt(sanitized.slice(0, 2), 16);
    const g = Number.parseInt(sanitized.slice(2, 4), 16);
    const b = Number.parseInt(sanitized.slice(4, 6), 16);

    if (Number.isNaN(r) || Number.isNaN(g) || Number.isNaN(b)) return { r: 0, g: 0, b: 0 };
    return { r, g, b };
  }

  function clamp01(value) {
    return Math.max(0, Math.min(1, value));
  }

  function toCssRgb({ r, g, b }) {
    return `rgb(${r}, ${g}, ${b})`;
  }

  function onLogoChange(event) {
    const [file] = event.target.files || [];
    logoFile = file || null;
    scheduleGenerate();
  }

  function scheduleGenerate() {
    if (!text.trim()) return;
    if (autoGenerateTimer) clearTimeout(autoGenerateTimer);
    autoGenerateTimer = setTimeout(() => {
      generateQR();
    }, 250);
  }

  function downloadQR() {
    if (!qrImageUrl) return;
    const a = document.createElement("a");
    a.href = qrImageUrl;
    a.download = "qr-code.png";
    document.body.appendChild(a);
    a.click();
    a.remove();
  }

  $: if (
    text ||
    size ||
    fgColor ||
    bgColor ||
    useGradient ||
    gradientStart ||
    gradientEnd ||
    gradientDirection ||
    logoScale ||
    moduleStyle ||
    finderOuterStyle ||
    finderInnerStyle ||
    finderOuterColor ||
    finderInnerColor
  ) {
    scheduleGenerate();
  }

  onDestroy(() => {
    if (autoGenerateTimer) clearTimeout(autoGenerateTimer);
  });
</script>

<main>
  <h1>QR Generator</h1>

  <div class="grid">
    <div class="panel">
      <label>
        QR content
        <input bind:value={text} placeholder="Enter text or URL" />
      </label>

      <div class="preset-section">
        <span class="preset-label">Eye style presets</span>
        <div class="preset-row">
          {#each FINDER_PRESETS as preset}
            <button
              type="button"
              class="preset-btn"
              class:active={eyePresetId === preset.id}
              on:click={() => applyFinderPreset(preset.id)}
            >
              {preset.label}
            </button>
          {/each}
        </div>
      </div>

      <label>
        Module style
        <select bind:value={moduleStyle} on:change={markFinderCustom}>
          <option value="square">Square</option>
          <option value="rounded">Rounded</option>
          <option value="circle">Circle</option>
        </select>
      </label>

      <label>
        Finder (eye) outer shape
        <select bind:value={finderOuterStyle} on:change={markFinderCustom}>
          <option value="square">Square</option>
          <option value="rounded">Rounded</option>
          <option value="circle">Circle</option>
        </select>
      </label>

      <label>
        Finder (eye) inner shape
        <select bind:value={finderInnerStyle} on:change={markFinderCustom}>
          <option value="square">Square</option>
          <option value="rounded">Rounded</option>
          <option value="circle">Circle</option>
        </select>
      </label>

      <label>
        Finder outer color
        <input type="color" bind:value={finderOuterColor} on:input={markFinderCustom} />
      </label>

      <label>
        Finder inner color
        <input type="color" bind:value={finderInnerColor} on:input={markFinderCustom} />
      </label>

      <label>
        Size ({size}px)
        <input type="range" min="128" max="1024" step="32" bind:value={size} />
      </label>

      <label>
        Foreground Color
        <input type="color" bind:value={fgColor} />
      </label>

      <label>
        Background Color
        <input type="color" bind:value={bgColor} />
      </label>

      <label class="inline">
        <input type="checkbox" bind:checked={useGradient} />
        Use gradient
      </label>

      {#if useGradient}
        <label>
          Gradient Start
          <input type="color" bind:value={gradientStart} />
        </label>

        <label>
          Gradient End
          <input type="color" bind:value={gradientEnd} />
        </label>

        <label>
          Direction
          <select bind:value={gradientDirection}>
            <option value="left_to_right">Left to right</option>
            <option value="right_to_left">Right to left</option>
            <option value="top_to_bottom">Top to bottom</option>
            <option value="bottom_to_top">Bottom to top</option>
            <option value="top_left_to_bottom_right">Top-left to bottom-right</option>
            <option value="bottom_right_to_top_left">Bottom-right to top-left</option>
            <option value="top_right_to_bottom_left">Top-right to bottom-left</option>
            <option value="bottom_left_to_top_right">Bottom-left to top-right</option>
          </select>
        </label>
      {/if}

      <label>
        Logo scale ({Math.round(logoScale * 100)}%)
        <input type="range" min="0.1" max="0.35" step="0.01" bind:value={logoScale} />
      </label>

      <label>
        Center logo
        <input type="file" accept="image/*" on:change={onLogoChange} />
      </label>

      <button on:click={generateQR} disabled={loading || !text.trim()}>
        {#if loading}Generating...{:else}Generate QR{/if}
      </button>

      {#if error}
        <p class="error">{error}</p>
      {/if}
    </div>

    <div class="panel preview">
      {#if qrImageUrl}
        <img src={qrImageUrl} alt="Generated QR code" />
        <button class="download-btn" on:click={downloadQR} disabled={loading}>
          Download PNG
        </button>
      {:else}
        <p>No QR generated yet.</p>
      {/if}
    </div>
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    font-family: Inter, system-ui, sans-serif;
    background: #f6f7fb;
    color: #1f2937;
  }

  main {
    max-width: 1100px;
    margin: 0 auto;
    padding: 2rem 1rem;
  }

  h1 {
    margin-top: 0;
  }

  .grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1rem;
  }

  .panel {
    background: white;
    border-radius: 0.75rem;
    padding: 1rem;
    box-shadow: 0 4px 18px rgba(0, 0, 0, 0.06);
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  label {
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
    font-size: 0.92rem;
  }

  .inline {
    flex-direction: row;
    align-items: center;
    gap: 0.5rem;
  }

  .preset-section {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .preset-label {
    font-size: 0.85rem;
    font-weight: 600;
    color: #374151;
  }

  .preset-row {
    display: flex;
    flex-wrap: wrap;
    gap: 0.45rem;
  }

  .preset-btn {
    padding: 0.45rem 0.7rem;
    font-size: 0.82rem;
    background: #f3f4f6;
    color: #1f2937;
    border: 1px solid #e5e7eb;
  }

  .preset-btn:hover {
    background: #e5e7eb;
  }

  .preset-btn.active {
    background: #111827;
    color: #fff;
    border-color: #111827;
  }

  input,
  select,
  button {
    padding: 0.6rem;
    border: 1px solid #d1d5db;
    border-radius: 0.5rem;
    font-size: 0.95rem;
  }

  input[type="color"] {
    padding: 0.2rem;
    min-height: 2.3rem;
  }

  button {
    background: #111827;
    color: white;
    border: none;
    cursor: pointer;
  }

  button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .preview {
    align-items: center;
    justify-content: center;
    min-height: 450px;
  }

  .download-btn {
    width: min(100%, 420px);
  }

  img {
    width: min(100%, 420px);
    border-radius: 0.5rem;
    border: 1px solid #e5e7eb;
  }

  .error {
    color: #dc2626;
    margin: 0;
  }

  @media (max-width: 900px) {
    .grid {
      grid-template-columns: 1fr;
    }
  }
</style>
