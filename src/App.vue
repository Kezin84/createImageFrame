<script setup lang="ts">
import { ref, computed, watch, nextTick, onMounted, onUnmounted } from 'vue';
import JSZip from 'jszip';

interface PhotoItem {
  id: string; src: string; imgElement?: HTMLImageElement; name: string;
  rotation: number; crop: { x: number; y: number; w: number; h: number } | null;
}
interface TextItem {
  id: string; content: string; x: number; y: number;
  fontSize: number; color: string; fontFamily: string; bold: boolean; italic: boolean;
}

const frameSrc = ref<string | null>(null);
const frameImage = ref<HTMLImageElement | null>(null);
const photos = ref<PhotoItem[]>([]);
const activePhotoId = ref<string | null>(null);
const activePhoto = computed(() => photos.value.find(p => p.id === activePhotoId.value));

const fbTargetSize = computed(() => {
  const photo = activePhoto.value;
  if (!photo?.imgElement) return null;
  const crop = photo.crop || { x: 0, y: 0, w: 1, h: 1 };
  const w = photo.imgElement.width * crop.w;
  const h = photo.imgElement.height * crop.h;
  const isRotated = photo.rotation === 90 || photo.rotation === 270;
  const finalW = isRotated ? h : w;
  const finalH = isRotated ? w : h;
  const ratio = finalW / finalH;
  if (ratio > 1.05) return { w: 1200, h: 630 }; // FB Landscape
  if (ratio < 0.95) return { w: 1080, h: 1350 }; // FB Portrait
  return { w: 1080, h: 1080 }; // FB Square
});

const canvasAspectRatio = computed(() => {
  if (fbTargetSize.value) {
    return `${fbTargetSize.value.w}/${fbTargetSize.value.h}`;
  }
  return frameImage.value ? `${frameImage.value.width}/${frameImage.value.height}` : '1/1';
});

function drawImageCover(ctx: CanvasRenderingContext2D, img: HTMLImageElement, sx: number, sy: number, sw: number, sh: number, dx: number, dy: number, dw: number, dh: number) {
  const rSrc = sw / sh;
  const rDst = dw / dh;
  let finalSx = sx, finalSy = sy, finalSw = sw, finalSh = sh;
  if (rSrc > rDst) {
    finalSw = sh * rDst;
    finalSx = sx + (sw - finalSw) / 2;
  } else {
    finalSh = sw / rDst;
    finalSy = sy + (sh - finalSh) / 2;
  }
  ctx.drawImage(img, finalSx, finalSy, finalSw, finalSh, dx, dy, dw, dh);
}

const textItems = ref<TextItem[]>([]);
const selectedTextId = ref<string | null>(null);
const activeTool = ref<string | null>(null);
const isDownloading = ref(false);
const upscaleExport = ref(true);

const container = ref<HTMLElement | null>(null);
const previewCanvas = ref<HTMLCanvasElement | null>(null);
const cropContainerRef = ref<HTMLElement | null>(null);
const cropRect = ref({ x: 0.1, y: 0.1, w: 0.8, h: 0.8 });
const cropDragging = ref<string | null>(null);
const cropDragStart = ref({ mx: 0, my: 0, x: 0, y: 0, w: 0, h: 0 });

const newText = ref({ content: '', fontSize: 32, color: '#ffffff', fontFamily: 'Outfit', bold: false, italic: false });

// ---- FILE UPLOADS ----
function onFrameUpload(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (!file) return;
  const url = URL.createObjectURL(file);
  frameSrc.value = url;
  const img = new Image();
  img.onload = () => { frameImage.value = img; nextTick(redrawPreview); };
  img.src = url;
}
function onPhotoUpload(e: Event) {
  const files = (e.target as HTMLInputElement).files;
  if (!files) return;
  for (let i = 0; i < files.length; i++) {
    const file = files[i];
    const url = URL.createObjectURL(file);
    const id = Date.now().toString() + '-' + i;
    const item: PhotoItem = { id, src: url, name: file.name.replace(/\.[^/.]+$/, ''), rotation: 0, crop: null };
    const img = new Image();
    img.onload = () => { item.imgElement = img; if (activePhotoId.value === id) nextTick(redrawPreview); };
    img.src = url;
    photos.value.push(item);
    if (!activePhotoId.value) activePhotoId.value = id;
  }
  (e.target as HTMLInputElement).value = '';
}
function removePhoto(id: string) {
  photos.value = photos.value.filter(p => p.id !== id);
  if (activePhotoId.value === id) activePhotoId.value = photos.value[0]?.id || null;
}

// ---- ROTATION ----
function rotateLeft() { if (activePhoto.value) { activePhoto.value.rotation = (activePhoto.value.rotation - 90 + 360) % 360; redrawPreview(); } }
function rotateRight() { if (activePhoto.value) { activePhoto.value.rotation = (activePhoto.value.rotation + 90) % 360; redrawPreview(); } }

// ---- CROP ----
function startCrop() {
  if (!activePhoto.value) return;
  activeTool.value = 'crop';
  cropRect.value = activePhoto.value.crop ? { ...activePhoto.value.crop } : { x: 0.05, y: 0.05, w: 0.9, h: 0.9 };
}
function applyCrop() { if (activePhoto.value) { activePhoto.value.crop = { ...cropRect.value }; } activeTool.value = null; nextTick(redrawPreview); }
function resetCrop() { if (activePhoto.value) activePhoto.value.crop = null; activeTool.value = null; nextTick(redrawPreview); }
function cancelCrop() { activeTool.value = null; }

function getCropCoords(e: MouseEvent | TouchEvent) {
  const rect = cropContainerRef.value?.getBoundingClientRect();
  if (!rect) return { px: 0, py: 0 };
  const cx = 'touches' in e ? e.touches[0].clientX : e.clientX;
  const cy = 'touches' in e ? e.touches[0].clientY : e.clientY;
  return { px: (cx - rect.left) / rect.width, py: (cy - rect.top) / rect.height };
}
function startCropDrag(type: string, e: MouseEvent | TouchEvent) {
  e.preventDefault(); e.stopPropagation();
  cropDragging.value = type;
  const { px, py } = getCropCoords(e);
  cropDragStart.value = { mx: px, my: py, ...cropRect.value };
  window.addEventListener('mousemove', onCropMove); window.addEventListener('mouseup', onCropEnd);
  window.addEventListener('touchmove', onCropMove); window.addEventListener('touchend', onCropEnd);
}
function onCropMove(e: MouseEvent | TouchEvent) {
  if (!cropDragging.value) return;
  const { px, py } = getCropCoords(e);
  const dx = px - cropDragStart.value.mx, dy = py - cropDragStart.value.my;
  const s = cropDragStart.value; const MIN = 0.05;
  let { x, y, w, h } = { x: s.x, y: s.y, w: s.w, h: s.h };
  if (cropDragging.value === 'move') { x = Math.max(0, Math.min(s.x + dx, 1 - w)); y = Math.max(0, Math.min(s.y + dy, 1 - h)); }
  else if (cropDragging.value === 'tl') { x = s.x + dx; y = s.y + dy; w = s.w - dx; h = s.h - dy; }
  else if (cropDragging.value === 'tr') { y = s.y + dy; w = s.w + dx; h = s.h - dy; }
  else if (cropDragging.value === 'bl') { x = s.x + dx; w = s.w - dx; h = s.h + dy; }
  else if (cropDragging.value === 'br') { w = s.w + dx; h = s.h + dy; }
  if (w < MIN) { if (cropDragging.value.includes('l')) x = s.x + s.w - MIN; w = MIN; }
  if (h < MIN) { if (cropDragging.value.includes('t')) y = s.y + s.h - MIN; h = MIN; }
  x = Math.max(0, x); y = Math.max(0, y);
  if (x + w > 1) w = 1 - x; if (y + h > 1) h = 1 - y;
  cropRect.value = { x, y, w, h };
}
function onCropEnd() {
  cropDragging.value = null;
  window.removeEventListener('mousemove', onCropMove); window.removeEventListener('mouseup', onCropEnd);
  window.removeEventListener('touchmove', onCropMove); window.removeEventListener('touchend', onCropEnd);
}

// ---- TEXT ----
const isAdding = ref(false);
const editingTextId = ref<string | null>(null);

function addText() {
  if (isAdding.value) return;
  if (!newText.value.content.trim()) return;
  isAdding.value = true;
  const t: TextItem = { id: Date.now().toString() + '-t', x: 50, y: 50, ...JSON.parse(JSON.stringify(newText.value)) };
  textItems.value.push(t);
  selectedTextId.value = t.id;
  newText.value.content = '';
  setTimeout(() => { isAdding.value = false; }, 300);
}
function removeText(id: string) {
  textItems.value = textItems.value.filter(t => t.id !== id);
  if (selectedTextId.value === id) { selectedTextId.value = null; editingTextId.value = null; }
}
function deleteSelected() {
  if (selectedTextId.value && editingTextId.value !== selectedTextId.value) {
    removeText(selectedTextId.value);
  }
}
function deselectText() {
  selectedTextId.value = null;
  editingTextId.value = null;
}
function selectText(id: string, e: MouseEvent | TouchEvent) {
  e.stopPropagation();
  if (selectedTextId.value !== id) {
    selectedTextId.value = id;
    editingTextId.value = null;
  }
}
function startEditText(id: string) {
  selectedTextId.value = id;
  editingTextId.value = id;
}
function onTextInput(id: string, e: Event) {
  const t = textItems.value.find(t => t.id === id);
  if (t) {
    const el = e.target as HTMLElement;
    t.content = el.innerText || '';
  }
}
function onTextBlur(id: string) {
  if (editingTextId.value === id) editingTextId.value = null;
  const t = textItems.value.find(t => t.id === id);
  if (t && !t.content.trim()) removeText(id);
}

// Keyboard: Delete/Backspace to remove selected text
function onKeyDown(e: KeyboardEvent) {
  if (e.key === 'Delete' || e.key === 'Backspace') {
    if (editingTextId.value) return; // don't delete while editing
    deleteSelected();
  }
  if (e.key === 'Escape') deselectText();
}
onMounted(() => window.addEventListener('keydown', onKeyDown));
onUnmounted(() => window.removeEventListener('keydown', onKeyDown));

// Text dragging (move)
const textDrag = ref<{ id: string; mx: number; my: number; sx: number; sy: number } | null>(null);
function startTextDrag(id: string, e: MouseEvent | TouchEvent) {
  if (editingTextId.value === id) return; // don't drag while editing
  e.preventDefault(); e.stopPropagation();
  selectedTextId.value = id;
  const t = textItems.value.find(t => t.id === id);
  if (!t) return;
  const cx = 'touches' in e ? e.touches[0].clientX : e.clientX;
  const cy = 'touches' in e ? e.touches[0].clientY : e.clientY;
  textDrag.value = { id, mx: cx, my: cy, sx: t.x, sy: t.y };
  window.addEventListener('mousemove', onTextMove); window.addEventListener('mouseup', onTextEnd);
  window.addEventListener('touchmove', onTextMove); window.addEventListener('touchend', onTextEnd);
}
function onTextMove(e: MouseEvent | TouchEvent) {
  if (!textDrag.value || !container.value) return;
  const t = textItems.value.find(t => t.id === textDrag.value!.id);
  if (!t) return;
  const cx = 'touches' in e ? e.touches[0].clientX : e.clientX;
  const cy = 'touches' in e ? e.touches[0].clientY : e.clientY;
  const rect = container.value.getBoundingClientRect();
  t.x = Math.max(0, Math.min(100, textDrag.value.sx + ((cx - textDrag.value.mx) / rect.width) * 100));
  t.y = Math.max(0, Math.min(100, textDrag.value.sy + ((cy - textDrag.value.my) / rect.height) * 100));
}
function onTextEnd() {
  textDrag.value = null;
  window.removeEventListener('mousemove', onTextMove); window.removeEventListener('mouseup', onTextEnd);
  window.removeEventListener('touchmove', onTextMove); window.removeEventListener('touchend', onTextEnd);
}

// Text resize (drag bottom-right corner to scale fontSize)
const textResize = ref<{ id: string; startY: number; startSize: number } | null>(null);
function startTextResize(id: string, e: MouseEvent | TouchEvent) {
  e.preventDefault(); e.stopPropagation();
  selectedTextId.value = id;
  const t = textItems.value.find(t => t.id === id);
  if (!t) return;
  const cy = 'touches' in e ? e.touches[0].clientY : e.clientY;
  textResize.value = { id, startY: cy, startSize: t.fontSize };
  window.addEventListener('mousemove', onTextResizeMove); window.addEventListener('mouseup', onTextResizeEnd);
  window.addEventListener('touchmove', onTextResizeMove); window.addEventListener('touchend', onTextResizeEnd);
}
function onTextResizeMove(e: MouseEvent | TouchEvent) {
  if (!textResize.value) return;
  const t = textItems.value.find(t => t.id === textResize.value!.id);
  if (!t) return;
  const cy = 'touches' in e ? e.touches[0].clientY : e.clientY;
  const dy = cy - textResize.value.startY;
  t.fontSize = Math.max(8, Math.min(200, textResize.value.startSize + dy * 0.5));
}
function onTextResizeEnd() {
  textResize.value = null;
  window.removeEventListener('mousemove', onTextResizeMove); window.removeEventListener('mouseup', onTextResizeEnd);
  window.removeEventListener('touchmove', onTextResizeMove); window.removeEventListener('touchend', onTextResizeEnd);
}

// ---- PREVIEW CANVAS ----
function redrawPreview() {
  nextTick(() => {
    const canvas = previewCanvas.value;
    const cont = container.value;
    if (!canvas || !cont) return;
    const cw = cont.clientWidth, ch = cont.clientHeight;
    const dpr = window.devicePixelRatio || 1;
    canvas.width = cw * dpr; canvas.height = ch * dpr;
    canvas.style.width = cw + 'px'; canvas.style.height = ch + 'px';
    const ctx = canvas.getContext('2d');
    if (!ctx) return;
    ctx.scale(dpr, dpr);
    ctx.clearRect(0, 0, cw, ch);

    // Draw photo
    const photo = activePhoto.value;
    if (photo?.imgElement) {
      const img = photo.imgElement;
      const crop = photo.crop || { x: 0, y: 0, w: 1, h: 1 };
      const sx = crop.x * img.width, sy = crop.y * img.height, sw = crop.w * img.width, sh = crop.h * img.height;
      ctx.save();
      ctx.translate(cw / 2, ch / 2);
      if (photo.rotation) { ctx.rotate((photo.rotation * Math.PI) / 180); }
      
      let drawW = cw, drawH = ch;
      if (photo.rotation === 90 || photo.rotation === 270) {
        drawW = ch; drawH = cw;
      }
      drawImageCover(ctx, img, sx, sy, sw, sh, -drawW / 2, -drawH / 2, drawW, drawH);
      ctx.restore();
    }
    // Draw frame
    if (frameImage.value) ctx.drawImage(frameImage.value, 0, 0, cw, ch);
    // Texts are rendered via HTML overlays (not on canvas) to allow drag interaction
  });
}

watch(activePhotoId, () => { activeTool.value = null; nextTick(redrawPreview); });
let resizeObs: ResizeObserver | null = null;
onMounted(() => { resizeObs = new ResizeObserver(redrawPreview); });
onUnmounted(() => resizeObs?.disconnect());
watch(container, (el) => { if (el) { resizeObs?.observe(el); redrawPreview(); } });

// ---- EXPORT ----
async function renderBlob(photo: PhotoItem): Promise<Blob | null> {
  if (!photo.imgElement) return null;
  const img = photo.imgElement;
  const crop = photo.crop || { x: 0, y: 0, w: 1, h: 1 };
  const sw = crop.w * img.width, sh = crop.h * img.height;
  const sx = crop.x * img.width, sy = crop.y * img.height;
  
  const isRotated = photo.rotation === 90 || photo.rotation === 270;
  const objW = isRotated ? sh : sw;
  const objH = isRotated ? sw : sh;
  const ratio = objW / objH;
  
  const scaleExport = upscaleExport.value ? 2 : 1;
  let tw = 1080 * scaleExport, th = 1080 * scaleExport;
  if (ratio > 1.05) { tw = 1200 * scaleExport; th = 630 * scaleExport; }
  else if (ratio < 0.95) { tw = 1080 * scaleExport; th = 1350 * scaleExport; }
  
  // Tùy chỉnh nếu không có ảnh sản phẩm nhưng lại có khung (cũng đã chặn trên)
  if (!img && frameImage.value) {
    tw = frameImage.value.width * scaleExport;
    th = frameImage.value.height * scaleExport;
  }

  const c = document.createElement('canvas'); c.width = tw; c.height = th;
  const ctx = c.getContext('2d'); if (!ctx) return null;
  ctx.imageSmoothingEnabled = true; ctx.imageSmoothingQuality = 'high';
  
  if (upscaleExport.value) {
    ctx.filter = 'contrast(102%) saturate(105%)'; // Làm nét và nổi bật màu sắc nhẹ
  }
  
  ctx.save();
  ctx.translate(tw / 2, th / 2);
  if (photo.rotation) { ctx.rotate((photo.rotation * Math.PI) / 180); }
  
  let drawW = tw, drawH = th;
  if (photo.rotation === 90 || photo.rotation === 270) {
    drawW = th; drawH = tw;
  }
  drawImageCover(ctx, img, sx, sy, sw, sh, -drawW / 2, -drawH / 2, drawW, drawH);
  ctx.restore();

  // Reset filter for frame and texts (so we don't double saturated them if frame is already saturated, though the frame might benefit too. Let's reset)
  ctx.filter = 'none';

  // Draw frame FIRST (under text), stretched to fit the photo size
  if (frameImage.value) {
    ctx.drawImage(frameImage.value, 0, 0, tw, th);
  }
  
  // Then draw texts ON TOP of frame
  const scale = tw / (container.value?.clientWidth || tw);
  for (const t of textItems.value) {
    ctx.save();
    let font = ''; if (t.italic) font += 'italic '; if (t.bold) font += 'bold ';
    font += `${Math.round(t.fontSize * scale)}px ${t.fontFamily}`;
    ctx.font = font; ctx.fillStyle = t.color; ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    ctx.shadowColor = 'rgba(0,0,0,0.7)'; ctx.shadowBlur = 6 * scale;
    ctx.fillText(t.content, (t.x / 100) * tw, (t.y / 100) * th);
    ctx.restore();
  }
  return new Promise(r => c.toBlob(b => r(b), 'image/png', 1.0));
}
async function downloadSingle(p?: PhotoItem) {
  const photo = p || activePhoto.value; if (!photo) return;
  isDownloading.value = true;
  try {
    const blob = await renderBlob(photo); if (!blob) return;
    const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = `${photo.name}-framed.png`;
    document.body.appendChild(a); a.click(); document.body.removeChild(a);
  } finally { isDownloading.value = false; }
}
async function downloadAll() {
  if (!photos.value.length) { alert('Cần có ít nhất 1 ảnh!'); return; }
  isDownloading.value = true;
  try {
    const zip = new JSZip(); const folder = zip.folder('MixPhoto-Export')!;
    await Promise.all(photos.value.map(async (p, i) => { const b = await renderBlob(p); if (b) folder.file(`${p.name}-${i + 1}.png`, b); }));
    const zb = await zip.generateAsync({ type: 'blob' });
    const a = document.createElement('a'); a.href = URL.createObjectURL(zb); a.download = 'MixPhoto-All.zip';
    document.body.appendChild(a); a.click(); document.body.removeChild(a);
  } finally { isDownloading.value = false; }
}
</script>

<template>
  <div class="app-layout">
    <aside class="sidebar">
      <div class="brand">✨ MixPhoto Studio</div>
      <p class="desc">Ghép ảnh vào khung • Crop • Rotate • Add Text</p>

      <div class="step-card">
        <div class="step-title"><span class="sn">1</span> Chọn khung ảnh</div>
        <label class="upload-btn ub-primary">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.5"/><polyline points="21 15 16 10 5 21"/></svg>
          Tải khung (PNG trong suốt)
          <input type="file" accept="image/png,image/webp" @change="onFrameUpload"/>
        </label>
      </div>

      <div class="step-card">
        <div class="step-title"><span class="sn">2</span> Chọn ảnh nền (nhiều file)</div>
        <label class="upload-btn">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M23 19a2 2 0 0 1-2 2H3a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h4l2-3h6l2 3h4a2 2 0 0 1 2 2z"/><circle cx="12" cy="13" r="4"/></svg>
          Tải ảnh chụp của bạn
          <input type="file" accept="image/*" multiple @change="onPhotoUpload"/>
        </label>
        <p class="hint">Tự căn chuẩn kích thước FB cực nét (Dọc 1080x1350, Ngang 1200x630, Vuông 1080x1080).</p>
      </div>

      <!-- TEXT TOOL PANEL -->
      <div class="tool-panel" v-if="activeTool === 'text'">
        <h3>✍️ Thêm Text</h3>
        <input class="ti" v-model="newText.content" placeholder="Nhập nội dung..."/>
        <div class="text-opts">
          <input type="number" class="ti small" v-model.number="newText.fontSize" min="8" max="200"/>
          <input type="color" class="color-pick" v-model="newText.color"/>
          <select class="ti" v-model="newText.fontFamily">
            <option>Outfit</option><option>Arial</option><option>Times New Roman</option><option>Roboto</option><option>Georgia</option>
          </select>
          <button class="tog" :class="{on: newText.bold}" @click="newText.bold=!newText.bold">B</button>
          <button class="tog" :class="{on: newText.italic}" @click="newText.italic=!newText.italic"><i>I</i></button>
        </div>
        <button type="button" class="add-text-btn" @click="addText">+ Thêm</button>

        <div class="text-list">
          <div v-for="t in textItems" :key="t.id" class="text-item-row" :class="{sel: selectedTextId===t.id}" @click="selectedTextId=t.id">
            <span class="text-preview">{{ t.content }}</span>
            <button class="del-btn" @click.stop="removeText(t.id)">✕</button>
          </div>
        </div>
        <!-- Edit selected -->
        <div v-if="selectedTextId && textItems.find(t=>t.id===selectedTextId)" class="edit-selected">
          <h4>Sửa text đã chọn</h4>
          <input class="ti" v-model="textItems.find(t=>t.id===selectedTextId)!.content"/>
          <div class="text-opts">
            <input type="number" class="ti small" v-model.number="textItems.find(t=>t.id===selectedTextId)!.fontSize" min="8" max="200"/>
            <input type="color" class="color-pick" v-model="textItems.find(t=>t.id===selectedTextId)!.color"/>
            <select class="ti" v-model="textItems.find(t=>t.id===selectedTextId)!.fontFamily">
              <option>Outfit</option><option>Arial</option><option>Times New Roman</option><option>Roboto</option><option>Georgia</option>
            </select>
            <button type="button" class="tog" :class="{on: textItems.find(t=>t.id===selectedTextId)!.bold}" @click="textItems.find(t=>t.id===selectedTextId)!.bold=!textItems.find(t=>t.id===selectedTextId)!.bold">B</button>
            <button type="button" class="tog" :class="{on: textItems.find(t=>t.id===selectedTextId)!.italic}" @click="textItems.find(t=>t.id===selectedTextId)!.italic=!textItems.find(t=>t.id===selectedTextId)!.italic"><i>I</i></button>
          </div>
        </div>
        <button class="close-tool" @click="activeTool=null">Đóng panel</button>
      </div>

      <!-- CROP PANEL -->
      <div class="tool-panel" v-if="activeTool === 'crop'">
        <h3>✂️ Cắt ảnh</h3>
        <p class="hint">Kéo các góc hoặc kéo vùng cắt trên ảnh xem trước.</p>
        <div class="crop-actions">
          <button class="crop-btn apply" @click="applyCrop">✓ Áp dụng</button>
          <button class="crop-btn" @click="resetCrop">↺ Đặt lại</button>
          <button class="crop-btn" @click="cancelCrop">✕ Hủy</button>
        </div>
      </div>

      <div class="export-zone" v-if="photos.length">
        <label class="upscale-box">
          <input type="checkbox" v-model="upscaleExport"/>
          <span>✨ <b>Upscale 2x & Làm nét</b> (Chống mờ FB)</span>
        </label>
        <button class="dl-btn" @click="downloadSingle()" :disabled="isDownloading">{{ isDownloading ? 'Đang xử lý...' : '⬇ Tải ảnh đang xem' }}</button>
        <button v-if="photos.length>1" class="dl-btn all" @click="downloadAll" :disabled="isDownloading">{{ isDownloading ? 'Đang nén...' : `⬇ Tải tất cả (${photos.length}) [ZIP]` }}</button>
      </div>
    </aside>

    <main class="workspace">
      <!-- TOOLBAR -->
      <div class="toolbar" v-if="activePhoto || frameSrc">
        <button class="tb" :class="{active:activeTool==='crop'}" @click="startCrop" title="Cắt">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M6.13 1L6 16a2 2 0 0 0 2 2h15"/><path d="M1 6.13L16 6a2 2 0 0 1 2 2v15"/></svg> Cắt
        </button>
        <button class="tb" @click="rotateLeft" title="Xoay trái">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 2.13-9.36L1 10"/></svg> ⟲
        </button>
        <button class="tb" @click="rotateRight" title="Xoay phải">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 1 1-2.13-9.36L23 10"/></svg> ⟳
        </button>
        <button class="tb" :class="{active:activeTool==='text'}" @click="activeTool = activeTool==='text'?null:'text'" title="Thêm chữ">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="4 7 4 4 20 4 20 7"/><line x1="9" y1="20" x2="15" y2="20"/><line x1="12" y1="4" x2="12" y2="20"/></svg> Text
        </button>
        <span class="tb-info" v-if="activePhoto">{{ activePhoto.rotation }}°</span>
      </div>

      <div class="viewport">
        <!-- CROP MODE -->
        <div v-if="activeTool==='crop' && activePhoto" class="crop-workspace" ref="cropContainerRef">
          <img :src="activePhoto.src" class="crop-img"/>
          <div class="shade" :style="{top:0,left:0,right:0,height:cropRect.y*100+'%'}"></div>
          <div class="shade" :style="{top:(cropRect.y+cropRect.h)*100+'%',left:0,right:0,bottom:0}"></div>
          <div class="shade" :style="{top:cropRect.y*100+'%',left:0,width:cropRect.x*100+'%',height:cropRect.h*100+'%'}"></div>
          <div class="shade" :style="{top:cropRect.y*100+'%',left:(cropRect.x+cropRect.w)*100+'%',right:0,height:cropRect.h*100+'%'}"></div>
          <div class="crop-rect" :style="{top:cropRect.y*100+'%',left:cropRect.x*100+'%',width:cropRect.w*100+'%',height:cropRect.h*100+'%'}"
            @mousedown="startCropDrag('move',$event)" @touchstart="startCropDrag('move',$event)">
            <div class="handle tl" @mousedown.stop="startCropDrag('tl',$event)" @touchstart.stop="startCropDrag('tl',$event)"></div>
            <div class="handle tr" @mousedown.stop="startCropDrag('tr',$event)" @touchstart.stop="startCropDrag('tr',$event)"></div>
            <div class="handle bl" @mousedown.stop="startCropDrag('bl',$event)" @touchstart.stop="startCropDrag('bl',$event)"></div>
            <div class="handle br" @mousedown.stop="startCropDrag('br',$event)" @touchstart.stop="startCropDrag('br',$event)"></div>
          </div>
        </div>

        <!-- NORMAL PREVIEW -->
        <div v-else-if="frameSrc || activePhoto" class="canvas-container" ref="container"
          :style="{aspectRatio: canvasAspectRatio}"
          @mousedown="deselectText">
          <canvas ref="previewCanvas" class="preview-canvas"></canvas>
          <!-- Canva-style text boxes -->
          <div v-for="t in textItems" :key="t.id" class="text-box"
            :class="{selected: selectedTextId===t.id, editing: editingTextId===t.id}"
            :style="{left:t.x+'%',top:t.y+'%'}"
            @mousedown="startTextDrag(t.id,$event)" @touchstart="startTextDrag(t.id,$event)"
            @click="selectText(t.id,$event)"
            @dblclick.stop="startEditText(t.id)">
            <div class="text-content"
              :contenteditable="editingTextId===t.id"
              :style="{fontSize:t.fontSize+'px',fontFamily:t.fontFamily,color:t.color,fontWeight:t.bold?'bold':'normal',fontStyle:t.italic?'italic':'normal'}"
              @input="onTextInput(t.id,$event)"
              @blur="onTextBlur(t.id)"
              @keydown.stop
              spellcheck="false"
            >{{ t.content }}</div>
            <!-- Resize handle -->
            <div v-if="selectedTextId===t.id && editingTextId!==t.id" class="resize-handle"
              @mousedown.stop="startTextResize(t.id,$event)"
              @touchstart.stop="startTextResize(t.id,$event)">
            </div>
            <!-- Delete button -->
            <button v-if="selectedTextId===t.id && editingTextId!==t.id" class="text-delete-btn"
              @mousedown.stop @click.stop="removeText(t.id)">✕</button>
          </div>
        </div>

        <div class="empty-state" v-else>
          <div class="empty-icon">🖼️</div>
          <h2>Khu vực xem trước</h2>
          <p>Tải khung và ảnh lên để bắt đầu.</p>
        </div>
      </div>

      <!-- REEL -->
      <div class="reel" v-if="photos.length">
        <div class="reel-track">
          <div v-for="p in photos" :key="p.id" class="thumb" :class="{active:activePhotoId===p.id}" @click="activePhotoId=p.id">
            <img :src="p.src"/>
            <div class="thumb-hover">
              <button class="ib rm" @click.stop="removePhoto(p.id)" title="Xóa">✕</button>
              <button class="ib dl" @click.stop="downloadSingle(p)" title="Tải">⬇</button>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@400;500;600;700&display=swap');
:root{--bg:#0b0d17;--panel:rgba(255,255,255,.03);--border:rgba(255,255,255,.08);--txt:#fff;--muted:#8b92a5;--pri:#8a2be2;--acc:#ff2a85}
body{margin:0;background:var(--bg);color:var(--txt);font-family:'Outfit',sans-serif;min-height:100vh;background-image:radial-gradient(ellipse at 10% 10%,rgba(138,43,226,.15) 0%,transparent 50%),radial-gradient(ellipse at 90% 90%,rgba(255,42,133,.1) 0%,transparent 50%)}
.app-layout{display:grid;grid-template-columns:340px 1fr;gap:20px;max-width:1440px;margin:0 auto;padding:20px;height:100vh;box-sizing:border-box}
.sidebar{background:var(--panel);backdrop-filter:blur(20px);border:1px solid var(--border);border-radius:16px;padding:24px;display:flex;flex-direction:column;gap:16px;overflow-y:auto;box-shadow:0 20px 40px -10px rgba(0,0,0,.5)}
.brand{font-size:24px;font-weight:700;background:linear-gradient(to right,#e0c3fc,#8ec5fc);background-clip:text;-webkit-background-clip:text;-webkit-text-fill-color:transparent}
.desc{font-size:13px;color:var(--muted);margin:0}
.step-card{background:rgba(255,255,255,.015);border:1px solid var(--border);border-radius:10px;padding:14px;display:flex;flex-direction:column;gap:10px}
.step-title{font-weight:600;font-size:14px;display:flex;align-items:center;gap:8px;color:#e2e8f0}
.sn{background:var(--pri);color:#fff;width:22px;height:22px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-size:11px;font-weight:700}
.upload-btn{display:flex;align-items:center;justify-content:center;gap:8px;padding:12px;background:rgba(255,255,255,.05);border:1px dashed rgba(255,255,255,.2);border-radius:8px;cursor:pointer;transition:all .3s;font-size:13px;font-weight:500;position:relative;color:var(--txt)}
.upload-btn:hover{background:rgba(255,255,255,.1);transform:translateY(-2px)}
.ub-primary{border-color:var(--pri);color:#e0c3fc;background:rgba(138,43,226,.08)}
.upload-btn input{position:absolute;top:0;left:0;opacity:0;width:100%;height:100%;cursor:pointer}

.tool-panel{background:rgba(255,255,255,.02);border:1px solid var(--border);border-radius:10px;padding:14px;display:flex;flex-direction:column;gap:10px}
.tool-panel h3{margin:0;font-size:15px}
.tool-panel h4{margin:8px 0 4px;font-size:13px;color:var(--muted)}
.hint{font-size:12px;color:var(--muted);margin:0}
.ti{background:rgba(255,255,255,.08);border:1px solid rgba(255,255,255,.15);border-radius:6px;padding:8px 10px;color:#fff;font-family:inherit;font-size:13px;width:100%;box-sizing:border-box}
.ti option{background-color:#1e1e2d;color:#ffffff}
.ti.small{width:70px}
.text-opts{display:flex;gap:6px;align-items:center;flex-wrap:wrap}
.color-pick{width:36px;height:36px;border:none;border-radius:6px;cursor:pointer;background:none;padding:0}
.tog{background:rgba(255,255,255,.08);border:1px solid rgba(255,255,255,.15);border-radius:6px;width:32px;height:32px;color:#fff;cursor:pointer;font-weight:700;font-size:14px;display:flex;align-items:center;justify-content:center}
.tog.on{background:var(--pri);border-color:var(--pri)}
.add-text-btn{background:var(--pri);color:#fff;border:none;padding:8px;border-radius:6px;cursor:pointer;font-family:inherit;font-weight:600;font-size:13px}
.text-list{display:flex;flex-direction:column;gap:4px;max-height:120px;overflow-y:auto}
.text-item-row{display:flex;justify-content:space-between;align-items:center;padding:6px 8px;background:rgba(255,255,255,.04);border-radius:6px;cursor:pointer;border:1px solid transparent;font-size:12px}
.text-item-row.sel{border-color:var(--pri)}
.text-preview{overflow:hidden;text-overflow:ellipsis;white-space:nowrap;max-width:200px}
.del-btn{background:none;border:none;color:#ef4444;cursor:pointer;font-size:14px;padding:2px 4px}
.edit-selected{border-top:1px solid var(--border);padding-top:8px}
.close-tool{background:rgba(255,255,255,.06);border:1px solid var(--border);border-radius:6px;padding:6px;color:var(--muted);cursor:pointer;font-family:inherit;font-size:12px}
.crop-actions{display:flex;gap:6px}
.crop-btn{flex:1;padding:8px;border:1px solid var(--border);border-radius:6px;background:rgba(255,255,255,.06);color:#fff;cursor:pointer;font-family:inherit;font-size:12px;font-weight:600}
.crop-btn.apply{background:var(--pri);border-color:var(--pri)}

.upscale-box{display:flex;align-items:center;gap:10px;font-size:13px;color:var(--txt);cursor:pointer;background:rgba(255,255,255,.03);padding:12px 14px;border-radius:10px;border:1px solid rgba(255,255,255,.1);transition:all .2s;user-select:none}
.upscale-box:hover{background:rgba(255,255,255,.08);border-color:var(--pri)}
.upscale-box input{width:16px;height:16px;cursor:pointer;accent-color:var(--pri)}
.upscale-box b{color:#e0c3fc}

.export-zone{margin-top:auto;display:flex;flex-direction:column;gap:10px}
.dl-btn{color:#fff;border:none;padding:14px;border-radius:10px;font-size:14px;font-weight:600;font-family:inherit;cursor:pointer;transition:all .3s;background:rgba(255,255,255,.08);border:1px solid rgba(255,255,255,.15)}
.dl-btn:hover:not(:disabled){background:rgba(255,255,255,.15);transform:translateY(-2px)}
.dl-btn.all{background:linear-gradient(135deg,var(--pri),var(--acc));border:none;box-shadow:0 8px 16px -8px rgba(255,42,133,.5)}
.dl-btn.all:hover:not(:disabled){transform:translateY(-3px) scale(1.02)}
.dl-btn:disabled{opacity:.5;cursor:not-allowed}

.workspace{background:var(--panel);backdrop-filter:blur(20px);border:1px solid var(--border);border-radius:16px;display:flex;flex-direction:column;overflow:hidden;box-shadow:inset 0 0 80px rgba(0,0,0,.4)}
.toolbar{display:flex;gap:8px;padding:12px 16px;border-bottom:1px solid var(--border);align-items:center;flex-wrap:wrap}
.tb{display:flex;align-items:center;gap:5px;background:rgba(255,255,255,.06);border:1px solid rgba(255,255,255,.12);border-radius:8px;padding:8px 12px;color:#ccc;cursor:pointer;font-family:inherit;font-size:12px;font-weight:500;transition:all .2s}
.tb:hover{background:rgba(255,255,255,.12);color:#fff}
.tb.active{background:var(--pri);border-color:var(--pri);color:#fff}
.tb-info{margin-left:auto;font-size:12px;color:var(--muted)}

.viewport{flex:1;display:flex;align-items:center;justify-content:center;padding:24px;overflow:hidden;position:relative}
.empty-state{text-align:center;color:var(--muted);max-width:360px}
.empty-icon{font-size:56px;margin-bottom:12px}
.empty-state h2{color:var(--txt);margin:0 0 8px;font-weight:500;font-size:20px}
.empty-state p{line-height:1.6;font-size:14px}

.canvas-container{position:relative;width:100%;max-width:100%;max-height:100%;border-radius:4px;box-shadow:0 16px 32px rgba(0,0,0,.5)}
.preview-canvas{width:100%;height:100%;display:block;border-radius:4px}

/* Canva-style text boxes */
.text-box{position:absolute;transform:translate(-50%,-50%);cursor:grab;user-select:none;z-index:20;white-space:nowrap;padding:0;border:2px solid transparent;border-radius:4px;transition:border-color .15s,box-shadow .15s}
.text-box:hover{border-color:rgba(138,43,226,.4)}
.text-box.selected{border-color:#8a2be2;box-shadow:0 0 0 1px rgba(138,43,226,.3)}
.text-box.editing{cursor:text;border-color:#8a2be2;border-style:solid}
.text-box:active:not(.editing){cursor:grabbing}
.text-content{outline:none;padding:6px 12px;text-shadow:0 2px 6px rgba(0,0,0,.7);line-height:1.2;min-width:20px}
.text-box.editing .text-content{cursor:text;background:rgba(0,0,0,.3);border-radius:2px}

/* Resize handle (bottom-right corner) */
.resize-handle{position:absolute;bottom:-6px;right:-6px;width:14px;height:14px;background:#8a2be2;border:2px solid #fff;border-radius:2px;cursor:nwse-resize;z-index:25;transition:transform .15s}
.resize-handle:hover{transform:scale(1.3)}

/* Delete button on selected text */
.text-delete-btn{position:absolute;top:-12px;right:-12px;width:22px;height:22px;background:#ef4444;color:#fff;border:2px solid #fff;border-radius:50%;font-size:11px;cursor:pointer;display:flex;align-items:center;justify-content:center;z-index:25;padding:0;line-height:1;transition:transform .15s,background .15s}
.text-delete-btn:hover{background:#dc2626;transform:scale(1.15)}

/* Crop workspace */
.crop-workspace{position:relative;width:100%;max-width:100%;max-height:100%;border-radius:4px;overflow:hidden}
.crop-img{width:100%;height:100%;object-fit:contain;display:block}
.shade{position:absolute;background:rgba(0,0,0,.6);pointer-events:none;z-index:2}
.crop-rect{position:absolute;border:2px dashed #fff;cursor:move;z-index:3;box-shadow:0 0 0 9999px rgba(0,0,0,0)}
.handle{position:absolute;width:14px;height:14px;background:#fff;border-radius:50%;border:2px solid var(--pri);z-index:4}
.handle.tl{top:-7px;left:-7px;cursor:nw-resize}
.handle.tr{top:-7px;right:-7px;cursor:ne-resize}
.handle.bl{bottom:-7px;left:-7px;cursor:sw-resize}
.handle.br{bottom:-7px;right:-7px;cursor:se-resize}

.reel{height:90px;background:rgba(0,0,0,.25);border-top:1px solid var(--border);padding:10px 20px;display:flex;align-items:center;overflow-x:auto}
.reel::-webkit-scrollbar{height:6px}
.reel::-webkit-scrollbar-thumb{background:rgba(255,255,255,.2);border-radius:3px}
.reel-track{display:flex;gap:12px;height:100%}
.thumb{aspect-ratio:1;height:100%;background:#111;border-radius:6px;border:2px solid transparent;cursor:pointer;position:relative;overflow:hidden;flex-shrink:0;transition:all .2s}
.thumb img{width:100%;height:100%;object-fit:cover;opacity:.7;transition:opacity .2s}
.thumb:hover img{opacity:1}
.thumb.active{border-color:var(--pri);box-shadow:0 0 0 2px rgba(138,43,226,.4)}
.thumb.active img{opacity:1}
.thumb-hover{position:absolute;inset:0;background:rgba(0,0,0,.5);display:flex;align-items:center;justify-content:center;gap:6px;opacity:0;transition:opacity .2s}
.thumb:hover .thumb-hover{opacity:1}
.ib{background:rgba(255,255,255,.2);border:none;border-radius:50%;width:26px;height:26px;display:flex;align-items:center;justify-content:center;color:#fff;cursor:pointer;font-size:12px;transition:all .2s}
.ib.rm:hover{background:#ef4444}
.ib.dl:hover{background:var(--pri)}

@media(max-width:900px){.app-layout{grid-template-columns:1fr;grid-template-rows:auto 70vh;height:auto}}
</style>
