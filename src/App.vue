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
  quote:
    'Start where you are. Use what you have. Do what you can.',
  author: 'Arthur Ashe',
  cta: 'Comparte esto con alguien que necesita escucharlo.',
  backgroundMode: 'default',
  solidColor: '#2979E1',
  gradientStart: '#2979E1',
  gradientEnd: '#123D86',
  quoteColor: '#ffffff',
  fontSize: 76,
  autoFit: true,
  selectedPreset: 'default-blue',
  photo: null,
  showTiktokGuide: true,
});

const quoteFont = computed(() => {
  return 'Inter, Montserrat, "Avenir Next", Futura, "Century Gothic", Arial, sans-serif';
});

const diaUnoFont = computed(() => {
  return '"Alex Brush", "Great Vibes", Allura, "Rockybilly Regular", Rockybilly, cursive';
});

const logoFont = computed(() => {
  return `400 156px ${diaUnoFont.value}`;
});

function applyPreset(id) {
  const preset = presets.find((item) => item.id === id);
  if (!preset) return;

  Object.assign(state, preset.settings);
  state.selectedPreset = id;
  if (id === 'default-blue') {
    state.photo = null;
    previewUrl.value = '';
  }
  renderFrame();
}

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
      state.backgroundMode = 'photo';
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
  if (state.backgroundMode === 'photo') state.backgroundMode = 'default';
  renderFrame();
}

function drawBackground(ctx) {
  if (state.backgroundMode === 'photo' && state.photo) {
    drawCoverImage(ctx, state.photo, 0, 0, frame.width, frame.height);
    ctx.fillStyle = 'rgba(0, 0, 0, 0.32)';
    ctx.fillRect(0, 0, frame.width, frame.height);
    return;
  }

  if (state.backgroundMode === 'solid') {
    ctx.fillStyle = state.solidColor;
    ctx.fillRect(0, 0, frame.width, frame.height);
    return;
  }

  if (state.backgroundMode === 'gradient') {
    const gradient = ctx.createLinearGradient(0, 0, frame.width, frame.height);
    gradient.addColorStop(0, state.gradientStart);
    gradient.addColorStop(1, state.gradientEnd);
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, frame.width, frame.height);
    return;
  }

  ctx.fillStyle = '#2979E1';
  ctx.fillRect(0, 0, frame.width, frame.height);
}

function drawCoverImage(ctx, image, x, y, width, height) {
  const scale = Math.max(width / image.width, height / image.height);
  const scaledWidth = image.width * scale;
  const scaledHeight = image.height * scale;
  const dx = x + (width - scaledWidth) / 2;
  const dy = y + (height - scaledHeight) / 2;
  ctx.drawImage(image, dx, dy, scaledWidth, scaledHeight);
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

function getCtaLines(ctx, text, maxWidth) {
  const cleanText = text.trim();
  if (!cleanText) return [];

  const fontSize = 50;
  ctx.font = `600 ${fontSize}px ${quoteFont.value}`;
  return wrapText(ctx, cleanText, maxWidth).slice(0, 2);
}

function drawCta(ctx, lines, x, y) {
  if (!lines.length) return 0;

  const lineHeight = 64;
  ctx.font = `600 50px ${quoteFont.value}`;
  drawCenteredText(ctx, lines, x, y, lineHeight);
  return lines.length * lineHeight;
}

function drawBoldScriptLogo(ctx, text, x, y) {
  const offsets = [
    [0, 0],
    [-1.8, 0],
    [1.8, 0],
    [0, -1.2],
    [0, 1.2],
  ];

  offsets.forEach(([dx, dy]) => {
    ctx.fillText(text, x + dx, y + dy);
  });
}

function getSafeContentBox() {
  const safeLeft = TIKTOK_SAFE_ZONE.contentLeftMargin;
  const safeRight =
    frame.width - (TIKTOK_SAFE_ZONE.rightIconColumnWidth + TIKTOK_SAFE_ZONE.rightIconColumnMargin);
  return {
    safeWidth: safeRight - safeLeft,
    safeCenterX: (safeLeft + safeRight) / 2,
    safeRight,
  };
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

  ctx.font = logoFont.value;
  ctx.shadowColor = 'rgba(0, 0, 0, 0.38)';
  ctx.shadowBlur = 5;
  ctx.shadowOffsetX = 0;
  ctx.shadowOffsetY = 7;
  ctx.fillStyle = '#ffffff';
  drawBoldScriptLogo(ctx, 'DiaUno', frame.width / 2, 380);
  ctx.shadowColor = 'transparent';
  ctx.shadowBlur = 0;
  ctx.shadowOffsetX = 0;
  ctx.shadowOffsetY = 0;

  const quote = state.quote.trim()
    ? `"${state.quote.trim().replace(/^["“”]+|["“”]+$/g, '')}"`
    : '"Your quote goes here."';
  const { safeWidth, safeCenterX, safeRight } = getSafeContentBox();
  const maxWidth = safeWidth;
  const maxHeight = 900;
  const fitted = fitQuote(ctx, quote, maxWidth, maxHeight);
  const ctaLines = getCtaLines(ctx, state.cta, maxWidth);
  const quoteBlockHeight = fitted.lines.length * fitted.lineHeight;
  const ctaGap = ctaLines.length ? 86 : 0;
  const ctaHeight = ctaLines.length * 64;
  const totalHeight = quoteBlockHeight + ctaGap + ctaHeight;
  const startY = frame.height / 2 - totalHeight / 2 + fitted.lineHeight / 2 + 58;

  ctx.font = `600 ${fitted.size}px ${quoteFont.value}`;
  ctx.fillStyle = state.quoteColor;
  drawCenteredText(ctx, fitted.lines, safeCenterX, startY, fitted.lineHeight);

  const ctaY = startY + quoteBlockHeight + ctaGap;
  drawCta(ctx, ctaLines, safeCenterX, ctaY);

  if (state.author.trim()) {
    ctx.font = `600 ${Math.max(26, Math.round(fitted.size * 0.36))}px ${quoteFont.value}`;
    ctx.textAlign = 'right';
    ctx.fillText(
      state.author.trim(),
      safeRight,
      1548,
    );
    ctx.textAlign = 'center';
  }
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
          <canvas ref="canvasRef" aria-label="Generated DiaUno quote frame"></canvas>

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

      <label class="field">
        <span>Author</span>
        <input v-model="state.author" type="text" placeholder="Optional" />
      </label>

      <label class="field">
        <span>CTA</span>
        <input
          v-model="state.cta"
          type="text"
          placeholder="Comparte esto con alguien que necesita escucharlo."
        />
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
