<script setup>
import { computed, nextTick, onMounted, reactive, ref, watch } from 'vue';

const canvasRef = ref(null);
const fileInputRef = ref(null);
const previewUrl = ref('');
const status = ref('');

const frame = {
  width: 1080,
  height: 1920,
};

const TIKTOK_SAFE_ZONE = {
  contentLeftMargin: 120,
  rightIconColumnWidth: 160,
  rightIconColumnMargin: 24,
  bottomCaptionHeight: 320,
};

const colorChoices = [
  { label: 'White', value: '#ffffff' },
  { label: 'Black', value: '#111111' },
  { label: 'Red', value: '#ef4444' },
  { label: 'Blue', value: '#2979E1' },
  { label: 'Yellow', value: '#fde047' },
  { label: 'Green', value: '#22c55e' },
];

const presets = [
  {
    id: 'default-blue',
    label: 'Default Blue',
    settings: {
      backgroundMode: 'default',
      quoteColor: '#ffffff',
      solidColor: '#2979E1',
      gradientStart: '#2979E1',
      gradientEnd: '#123D86',
      fontSize: 76,
    },
  },
];

const state = reactive({
  quote: '',
  photo: null,
  brightness: 100,
  contrast: 120,
  zoom: 1,
  panX: 0,
  panY: 0,
  boxX: 138,
  boxY: 685,
  boxWidth: 600,
  boxHeight: 270,
  quoteAreaMode: 'white',
  quoteAreaBrightness: 100,
  quoteAreaOpacity: 0,
  quoteAreaContrast: 120,
  logoText: 'DiaUno',
  logoSize: 90,
  fontSize: 76,
  autoFit: true,
  author: '',
  backgroundMode: 'default',
  solidColor: '#2979E1',
  gradientStart: '#2979E1',
  gradientEnd: '#123D86',
  quoteColor: '#ffffff',
  selectedPreset: 'default-blue',
  showTiktokGuide: false,
});

const quoteFont = computed(() => {
  return 'Inter, Montserrat, "Avenir Next", Futura, "Century Gothic", Arial, sans-serif';
});

const logoFont = computed(() => `500 ${state.logoSize}px Arial, sans-serif`);

function choosePhoto() {
  fileInputRef.value?.click();
}

function onPhotoSelected(event) {
  const [file] = event.target.files;
  if (!file) return;

  const reader = new FileReader();
  reader.addEventListener('load', () => {
    const image = new Image();
    image.addEventListener('load', () => {
      state.photo = image;
      previewUrl.value = reader.result;
      state.panX = 0;
      state.panY = 0;
      state.zoom = 1;
      renderFrame();
    });
    image.src = reader.result;
  });
  reader.readAsDataURL(file);
}

function clearPhoto() {
  state.photo = null;
  previewUrl.value = '';
  if (fileInputRef.value) fileInputRef.value.value = '';
  renderFrame();
}

function drawBackground(ctx) {
  ctx.fillStyle = '#222222';
  ctx.fillRect(0, 0, frame.width, frame.height);
  if (!state.photo) return;
  const scale = Math.max(frame.width / state.photo.width, frame.height / state.photo.height) * state.zoom;
  const width = state.photo.width * scale;
  const height = state.photo.height * scale;
  ctx.save();
  ctx.filter = `grayscale(1) brightness(${state.brightness}%) contrast(${state.contrast}%)`;
  ctx.drawImage(state.photo, (frame.width - width) / 2 + state.panX, (frame.height - height) / 2 + state.panY, width, height);
  ctx.restore();
}

const drag = { mode: null, startX: 0, startY: 0, originX: 0, originY: 0, originWidth: 0, originHeight: 0 };
function pointerPosition(event) {
  const bounds = canvasRef.value.getBoundingClientRect();
  return { x: (event.clientX - bounds.left) * frame.width / bounds.width, y: (event.clientY - bounds.top) * frame.height / bounds.height };
}
function onCanvasPointerDown(event) {
  const { x, y } = pointerPosition(event);
  const within = x >= state.boxX && x <= state.boxX + state.boxWidth && y >= state.boxY && y <= state.boxY + state.boxHeight;
  const nearRight = x >= state.boxX + state.boxWidth - 32 && y >= state.boxY && y <= state.boxY + state.boxHeight;
  const nearBottom = y >= state.boxY + state.boxHeight - 32 && x >= state.boxX && x <= state.boxX + state.boxWidth;
  drag.mode = nearRight && nearBottom ? 'resize-both' : nearRight ? 'resize-x' : nearBottom ? 'resize-y' : within ? 'box' : 'photo';
  drag.startX = x; drag.startY = y;
  drag.originX = drag.mode === 'photo' ? state.panX : state.boxX;
  drag.originY = drag.mode === 'photo' ? state.panY : state.boxY;
  drag.originWidth = state.boxWidth;
  drag.originHeight = state.boxHeight;
  canvasRef.value.style.cursor = drag.mode === 'resize-x' ? 'ew-resize' : drag.mode === 'resize-y' ? 'ns-resize' : drag.mode === 'resize-both' ? 'nwse-resize' : 'grabbing';
  canvasRef.value.setPointerCapture(event.pointerId);
}
function onCanvasPointerMove(event) {
  if (!drag.mode) return;
  const { x, y } = pointerPosition(event); const dx = x - drag.startX; const dy = y - drag.startY;
  if (drag.mode === 'box') {
    state.boxX = Math.max(0, Math.min(frame.width - state.boxWidth, drag.originX + dx));
    state.boxY = Math.max(0, Math.min(frame.height - state.boxHeight, drag.originY + dy));
  } else if (drag.mode === 'resize-x' || drag.mode === 'resize-both') {
    state.boxWidth = Math.max(360, Math.min(frame.width - state.boxX, drag.originWidth + dx));
  }
  if (drag.mode === 'resize-y' || drag.mode === 'resize-both') {
    state.boxHeight = Math.max(180, Math.min(frame.height - state.boxY, drag.originHeight + dy));
  }
  if (drag.mode === 'photo') { state.panX = drag.originX + dx; state.panY = drag.originY + dy; }
  renderFrame();
}
function onCanvasPointerUp() {
  drag.mode = null;
  if (canvasRef.value) canvasRef.value.style.cursor = 'grab';
}

function wrapTextLine(ctx, text, maxWidth) {
  const words = text.trim().split(/[^\S\r\n]+/).filter(Boolean);
  const lines = [];
  let line = '';

  for (const word of words) {
    const testLine = line ? `${line} ${word}` : word;
    if (ctx.measureText(testLine).width <= maxWidth || !line) {
      line = testLine;
    } else {
      lines.push(line);
      line = word;
    }
  }

  if (line) lines.push(line);
  return lines;
}

function wrapText(ctx, text, maxWidth) {
  const paragraphs = text.split(/\r?\n/);

  return paragraphs.flatMap((paragraph) => {
    if (!paragraph.trim()) return [''];
    return wrapTextLine(ctx, paragraph, maxWidth);
  });
}

function fitQuote(ctx, quote, maxWidth, maxHeight) {
  const requestedSize = Number(state.fontSize);
  let size = state.autoFit ? requestedSize : Math.max(34, requestedSize);
  let lines = [];
  let lineHeight = size * 1.22;

  while (size >= 34) {
    ctx.font = `600 ${size}px ${quoteFont.value}`;
    lines = wrapText(ctx, quote, maxWidth);
    lineHeight = size * 1.22;
    if (!state.autoFit || lines.length * lineHeight <= maxHeight) break;
    size -= 2;
  }

  return { size, lines, lineHeight };
}

function drawCenteredText(ctx, lines, x, y, lineHeight) {
  lines.forEach((line, index) => {
    ctx.fillText(line, x, y + index * lineHeight);
  });
}

function drawBoldScriptLogo(ctx, text, x, y) {
  [[0, 0], [-1.8, 0], [1.8, 0], [0, -1.2], [0, 1.2]].forEach(([dx, dy]) => {
    ctx.fillText(text, x + dx, y + dy);
  });
}

function getSafeContentBox() {
  const safeLeft = TIKTOK_SAFE_ZONE.contentLeftMargin;
  const safeRight = frame.width - (TIKTOK_SAFE_ZONE.rightIconColumnWidth + TIKTOK_SAFE_ZONE.rightIconColumnMargin);
  return { safeWidth: safeRight - safeLeft, safeCenterX: (safeLeft + safeRight) / 2 };
}


function renderFrame() {
  const canvas = canvasRef.value;
  if (!canvas) return;

  const ctx = canvas.getContext('2d');
  canvas.width = frame.width;
  canvas.height = frame.height;

  drawBackground(ctx);

  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';

  if (state.quoteAreaMode === 'white') {
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(state.boxX, state.boxY, state.boxWidth, state.boxHeight);
  } else if (state.photo) {
    const scale = Math.max(frame.width / state.photo.width, frame.height / state.photo.height) * state.zoom;
    const width = state.photo.width * scale;
    const height = state.photo.height * scale;
    ctx.save();
    ctx.beginPath();
    ctx.rect(state.boxX, state.boxY, state.boxWidth, state.boxHeight);
    ctx.clip();
    ctx.filter = `grayscale(1) brightness(${state.quoteAreaBrightness}%) contrast(${state.quoteAreaContrast}%)`;
    ctx.drawImage(state.photo, (frame.width - width) / 2 + state.panX, (frame.height - height) / 2 + state.panY, width, height);
    ctx.restore();
    if (state.quoteAreaOpacity > 0) {
      ctx.fillStyle = `rgba(255,255,255,${state.quoteAreaOpacity / 100})`;
      ctx.fillRect(state.boxX, state.boxY, state.boxWidth, state.boxHeight);
    }
  } else {
    const level = Math.round(35 + state.quoteAreaBrightness * 0.75);
    ctx.fillStyle = `rgb(${level}, ${level}, ${level})`;
    ctx.fillRect(state.boxX, state.boxY, state.boxWidth, state.boxHeight);
  }
  ctx.strokeStyle = '#111111';
  ctx.lineWidth = 5;
  ctx.strokeRect(state.boxX, state.boxY, state.boxWidth, state.boxHeight);

  const quote = state.quote.trim()
    ? `"${state.quote.trim().replace(/^["“”]+|["“”]+$/g, '')}"`
    : '"Your quote goes here."';
  const cleanQuote = state.quote.trim().replace(/^["“”]+|["“”]+$/g, '') || 'Escribe tu quote aquí.';
  const padding = 58;
  const fitted = fitQuote(ctx, cleanQuote, state.boxWidth - padding * 2, state.boxHeight - padding * 2);
  ctx.font = `400 ${fitted.size}px ${quoteFont.value}`;
  ctx.fillStyle = '#111111';
  drawCenteredText(ctx, fitted.lines, state.boxX + state.boxWidth / 2,
    state.boxY + state.boxHeight / 2 - (fitted.lines.length - 1) * fitted.lineHeight / 2, fitted.lineHeight);

  const logoX = frame.width - TIKTOK_SAFE_ZONE.rightIconColumnWidth - TIKTOK_SAFE_ZONE.rightIconColumnMargin - 20;
  const logoY = frame.height - TIKTOK_SAFE_ZONE.bottomCaptionHeight - 100;
  ctx.font = logoFont.value;
  ctx.textAlign = 'right';
  const logoWidth = ctx.measureText(state.logoText).width + 108;
  ctx.fillStyle = 'rgba(255,255,255,0.84)';
  ctx.fillRect(logoX - logoWidth, logoY - 81, logoWidth, 162);
  ctx.fillStyle = '#111111';
  ctx.fillText(state.logoText, logoX - 54, logoY);
}

function canvasToBlob() {
  return new Promise((resolve) => {
    canvasRef.value.toBlob((blob) => resolve(blob), 'image/png', 1);
  });
}

async function downloadPng() {
  renderFrame();
  const blob = await canvasToBlob();
  if (!blob) return;

  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = 'diauno-quote-frame.png';
  link.click();
  URL.revokeObjectURL(link.href);
  status.value = 'PNG downloaded.';
}

async function copyImage() {
  renderFrame();
  const blob = await canvasToBlob();
  if (!blob) return;

  try {
    await navigator.clipboard.write([
      new ClipboardItem({ [blob.type]: blob }),
    ]);
    status.value = 'Image copied to clipboard.';
  } catch {
    status.value = 'Clipboard copy is not available in this browser.';
  }
}

async function shareImage() {
  renderFrame();
  const blob = await canvasToBlob();
  if (!blob) return;

  const file = new File([blob], 'diauno-quote-frame.png', {
    type: 'image/png',
  });

  if (navigator.canShare?.({ files: [file] })) {
    await navigator.share({
      files: [file],
      title: 'DiaUno Quote Frame',
    });
    status.value = 'Share sheet opened.';
  } else {
    status.value = 'Sharing files is not available in this browser.';
  }
}

watch(
  state,
  () => {
    status.value = '';
    nextTick(renderFrame);
  },
  { deep: true },
);

onMounted(() => {
  document.fonts?.ready.then(renderFrame);
  renderFrame();
});
</script>

<template>
  <main class="app-shell">
    <section class="preview-panel" aria-label="Quote frame preview">
      <div class="preview-toolbar">
        <div>
          <p class="eyebrow">1080 x 1920</p>
          <h1>DiaUno Frame</h1>
        </div>
        <div class="toolbar-actions">
          <button type="button" class="icon-button" @click="copyImage" aria-label="Copy image">
            Copy
          </button>
          <button type="button" class="primary-button" @click="downloadPng">
            Download PNG
          </button>
        </div>
      </div>

      <div class="phone-stage">
        <div class="canvas-frame">
          <canvas
            ref="canvasRef"
            aria-label="Generated DiaUno quote frame"
            @pointerdown="onCanvasPointerDown"
            @pointermove="onCanvasPointerMove"
            @pointerup="onCanvasPointerUp"
            @pointercancel="onCanvasPointerUp"
          ></canvas>
          <div class="box-resize-handle box-resize-handle--right" :style="{ left: `${(state.boxX + state.boxWidth - 6) / frame.width * 100}%`, top: `${(state.boxY + state.boxHeight / 2 - 18) / frame.height * 100}%` }" aria-hidden="true"></div>
          <div class="box-resize-handle box-resize-handle--bottom" :style="{ left: `${(state.boxX + state.boxWidth / 2 - 18) / frame.width * 100}%`, top: `${(state.boxY + state.boxHeight - 6) / frame.height * 100}%` }" aria-hidden="true"></div>
          <div class="box-resize-handle box-resize-handle--corner" :style="{ left: `${(state.boxX + state.boxWidth - 18) / frame.width * 100}%`, top: `${(state.boxY + state.boxHeight - 18) / frame.height * 100}%` }" aria-hidden="true"></div>

          <div v-if="state.showTiktokGuide" class="tiktok-guide" aria-hidden="true">
            <div class="tiktok-guide__safe-outline"></div>

            <div class="tiktok-guide__icon-rail">
              <div class="tiktok-guide__icon tiktok-guide__icon--avatar">🙂</div>
              <div class="tiktok-guide__icon">
                <span>♡</span>
                <span class="tiktok-guide__icon-count">12.3K</span>
              </div>
              <div class="tiktok-guide__icon">
                <span>💬</span>
                <span class="tiktok-guide__icon-count">248</span>
              </div>
              <div class="tiktok-guide__icon">
                <span>🔖</span>
                <span class="tiktok-guide__icon-count">96</span>
              </div>
              <div class="tiktok-guide__icon">
                <span>↗</span>
                <span class="tiktok-guide__icon-count">Share</span>
              </div>
              <div class="tiktok-guide__icon tiktok-guide__icon--disc">🎵</div>
            </div>

            <div class="tiktok-guide__caption-zone">
              <span class="tiktok-guide__caption-line">@username</span>
              <span class="tiktok-guide__caption-line tiktok-guide__caption-line--muted">
                Caption text goes here #hashtag
              </span>
            </div>
          </div>
        </div>
      </div>

      <div class="share-row">
        <button type="button" class="secondary-button" @click="shareImage">
          Share
        </button>
        <p role="status">{{ status }}</p>
      </div>
    </section>

    <aside class="controls-panel" aria-label="Quote frame controls">
      <label class="field">
        <span>Quote</span>
        <textarea v-model="state.quote" rows="6" />
      </label>

      <fieldset>
        <legend>Quote background</legend>
        <div class="segmented">
          <label>
            <input v-model="state.quoteAreaMode" type="radio" value="white" />
            <span>White</span>
          </label>
          <label>
            <input v-model="state.quoteAreaMode" type="radio" value="transparent" />
            <span>Transparent</span>
          </label>
        </div>
        <label v-if="state.quoteAreaMode === 'transparent'" class="range-field">
          <span>Brillo {{ state.quoteAreaBrightness }}%</span>
          <input v-model.number="state.quoteAreaBrightness" min="25" max="175" step="5" type="range" />
        </label>
        <label v-if="state.quoteAreaMode === 'transparent'" class="range-field">
          <span>Opacidad {{ state.quoteAreaOpacity }}%</span>
          <input v-model.number="state.quoteAreaOpacity" min="0" max="100" step="5" type="range" />
        </label>
        <label v-if="state.quoteAreaMode === 'transparent'" class="range-field">
          <span>Contraste {{ state.quoteAreaContrast }}%</span>
          <input v-model.number="state.quoteAreaContrast" min="25" max="200" step="5" type="range" />
        </label>
      </fieldset>

      <label class="field">
        <span>Author</span>
        <input v-model="state.author" type="text" placeholder="Optional" />
      </label>

      <label class="field">
        <span>Preset</span>
        <select v-model="state.selectedPreset" @change="applyPreset(state.selectedPreset)">
          <option v-for="preset in presets" :key="preset.id" :value="preset.id">
            {{ preset.label }}
          </option>
        </select>
      </label>

      <fieldset>
        <legend>Background</legend>
        <div class="segmented">
          <label>
            <input v-model="state.backgroundMode" type="radio" value="default" />
            <span>Default</span>
          </label>
          <label>
            <input v-model="state.backgroundMode" type="radio" value="solid" />
            <span>Solid</span>
          </label>
          <label>
            <input v-model="state.backgroundMode" type="radio" value="gradient" />
            <span>Gradient</span>
          </label>
          <label>
            <input v-model="state.backgroundMode" type="radio" value="photo" />
            <span>Photo</span>
          </label>
        </div>

        <div v-if="state.backgroundMode === 'solid'" class="swatch-grid">
          <button
            v-for="color in colorChoices"
            :key="color.value"
            type="button"
            class="swatch"
            :class="{ selected: state.solidColor === color.value }"
            :style="{ backgroundColor: color.value }"
            :aria-label="`Set background ${color.label}`"
            @click="state.solidColor = color.value"
          />
          <input v-model="state.solidColor" type="color" aria-label="Custom background color" />
        </div>

        <div v-if="state.backgroundMode === 'gradient'" class="color-pair">
          <label>
            <span>Start</span>
            <input v-model="state.gradientStart" type="color" />
          </label>
          <label>
            <span>End</span>
            <input v-model="state.gradientEnd" type="color" />
          </label>
        </div>

        <div v-if="state.backgroundMode === 'photo'" class="photo-control">
          <input
            ref="fileInputRef"
            type="file"
            accept="image/*"
            class="hidden-input"
            @change="onPhotoSelected"
          />
          <button type="button" class="secondary-button" @click="choosePhoto">
            Upload Photo
          </button>
          <button
            v-if="previewUrl"
            type="button"
            class="ghost-button"
            @click="clearPhoto"
          >
            Remove
          </button>
        </div>
      </fieldset>

      <fieldset>
        <legend>Quote Color</legend>
        <div class="swatch-grid">
          <button
            v-for="color in colorChoices"
            :key="color.value"
            type="button"
            class="swatch"
            :class="{ selected: state.quoteColor === color.value }"
            :style="{ backgroundColor: color.value }"
            :aria-label="`Set quote ${color.label}`"
            @click="state.quoteColor = color.value"
          />
          <input v-model="state.quoteColor" type="color" aria-label="Custom quote color" />
        </div>
      </fieldset>

      <fieldset>
        <legend>Type Size</legend>
        <label class="toggle-row">
          <input v-model="state.autoFit" type="checkbox" />
          <span>Auto fit quote</span>
        </label>
        <label class="range-field">
          <span>{{ state.fontSize }}px</span>
          <input v-model.number="state.fontSize" min="38" max="128" step="2" type="range" />
        </label>
      </fieldset>

      <fieldset>
        <legend>Preview Guides</legend>
        <label class="toggle-row">
          <input v-model="state.showTiktokGuide" type="checkbox" />
          <span>Show TikTok UI guide</span>
        </label>
      </fieldset>
    </aside>
  </main>
</template>
