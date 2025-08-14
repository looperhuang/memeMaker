<template>
  <q-page>
    <q-card>
      <q-card-section>
        <div class="row q-col-gutter-md items-center">
          <!-- 選擇圖片 -->
          <div class="col-12 col-md-5">
            <q-file
              filled
              v-model="fileModel"
              label="選擇圖片進行編輯"
              accept="image/*"
              use-chips
              @update:model-value="onPick"
            />
          </div>

          <!-- 模式切換 -->
          <div class="col-12 col-md-7">
            <q-btn-toggle
              v-model="tool"
              push
              toggle-color="primary"
              :options="[
                { label: '選取', value: 'select', icon: 'mouse' },
                { label: '繪圖', value: 'draw', icon: 'brush' },
                { label: '文字', value: 'text', icon: 'text_fields' },
              ]"
              class="q-mr-md"
            />

            <!-- 繪圖模式設定 -->
            <span
              v-if="tool === 'draw'"
              class="row inline items-center q-gutter-xs"
            >
              <q-input
                dense
                filled
                v-model="brushColor"
                label="筆刷顏色"
                style="width: 140px"
              />
              <q-slider
                dense
                v-model="brushWidth"
                :min="1"
                :max="80"
                label
                label-always
                style="width: 200px"
              />
            </span>

            <!-- 文字模式設定 -->
            <span
              v-if="tool === 'text'"
              class="row inline items-center q-gutter-xs"
            >
              <q-input
                dense
                filled
                v-model="textString"
                label="文字內容"
                style="width: 220px"
              />
              <q-input
                dense
                filled
                v-model.number="fontSize"
                type="number"
                label="字級"
                style="width: 100px"
              />
              <q-input
                dense
                filled
                v-model="textColor"
                label="文字顏色"
                style="width: 140px"
              />
              <q-btn dense flat icon="add" label="點畫布加文字" />
            </span>
          </div>
        </div>

        <div class="q-mt-md">
          <canvas id="canvasRef"></canvas>
        </div>
      </q-card-section>

      <q-card-actions>
        <q-btn color="secondary" @click="undo" :disable="!canUndo">Undo</q-btn>
        <q-btn color="secondary" @click="redo" :disable="!canRedo">Redo</q-btn>
        <q-btn color="negative" @click="deleteSelected">Delete</q-btn>
        <q-space />
        <q-btn
          flat
          icon="center_focus_strong"
          @click="fitCanvas"
          label="重置視圖"
        />
      </q-card-actions>
    </q-card>
  </q-page>
</template>

<script setup lang="ts">
import * as fabric from 'fabric';
import { onMounted, onBeforeUnmount, ref, watch } from 'vue';
import { Notify } from 'quasar';

const canvas = ref<fabric.Canvas>();
const history = ref<string[]>([]);
const future = ref<string[]>([]);
const canUndo = ref(false);
const canRedo = ref(false);

// ===== 模式：select / draw / text =====
const tool = ref<'select' | 'draw' | 'text'>('select');

watch(tool, (t) => {
  const c = canvas.value;
  if (!c) return;
  if (t === 'draw') {
    c.isDrawingMode = true;
    c.selection = false;
    ensureBrush();
    applyBrushProps();
    c.setCursor('crosshair');
  } else {
    c.isDrawingMode = false;
    c.selection = t === 'select';
    c.setCursor('default');
  }
  Notify.create({
    message: `模式：${
      t === 'select' ? '選取' : t === 'draw' ? '繪圖' : '文字'
    }`,
    color: 'primary',
    position: 'top',
  });
});

// 檔案選擇
const fileModel = ref<File | null>(null);
const onPick = (f: File | null) => {
  if (f) addImage(f);
};

// ====== 繪圖設定 ======
const brushColor = ref('#222222');
const brushWidth = ref(8);

function ensureBrush() {
  const c = canvas.value;
  if (!c) return;
  if (!c.freeDrawingBrush) c.freeDrawingBrush = new fabric.PencilBrush(c);
}

function applyBrushProps() {
  const c = canvas.value;
  if (!c || !c.isDrawingMode) return;
  ensureBrush();
  (c.freeDrawingBrush as any).color = brushColor.value;
  (c.freeDrawingBrush as any).width = brushWidth.value;
}
watch([brushColor, brushWidth], () => {
  if (tool.value === 'draw') applyBrushProps();
});

// ====== 文字設定與新增 ======
const textString = ref('雙擊可編輯');
const textColor = ref('#111111');
const fontSize = ref(36);

function addTextAtPointer(opt: fabric.TEvent) {
  const c = canvas.value;
  if (!c) return;
  const p = c.getPointer(opt.e);
  const itext = new fabric.IText(textString.value, {
    left: p.x,
    top: p.y,
    fill: textColor.value,
    fontSize: fontSize.value,
    editable: true,
  });
  c.add(itext);
  c.setActiveObject(itext);
  c.requestRenderAll();
  saveHistory();
}

// 在文字模式下，點擊畫布即可新增文字
function attachCanvasEvents() {
  const c = canvas.value!;
  c.on('mouse:down', (opt) => {
    if (tool.value === 'text') addTextAtPointer(opt);
  });
  // 變更即記錄歷史
  const record = () => saveHistory();
  c.on('object:added', record);
  c.on('object:modified', record);
  c.on('object:removed', record);
  c.on('path:created', record);
}

// ====== 歷史（Undo / Redo） ======
const saveHistory = () => {
  if (!canvas.value) return;
  future.value = [];
  history.value.push(JSON.stringify(canvas.value.toJSON()));
  if (history.value.length > 100) history.value.shift();
  canUndo.value = history.value.length > 1;
  canRedo.value = future.value.length > 0;
};

const undo = () => {
  if (!canvas.value || history.value.length < 2) return;
  const current = history.value.pop()!;
  future.value.push(current);
  const prev = history.value[history.value.length - 1];
  canvas.value.loadFromJSON(prev, () => canvas.value!.renderAll());
  canUndo.value = history.value.length > 1;
  canRedo.value = future.value.length > 0;
};

const redo = () => {
  if (!canvas.value || future.value.length === 0) return;
  const next = future.value.pop()!;
  history.value.push(next);
  canvas.value.loadFromJSON(next, () => canvas.value!.renderAll());
  canUndo.value = history.value.length > 1;
  canRedo.value = future.value.length > 0;
};

const deleteSelected = () => {
  if (!canvas.value) return;
  const activeObjects = canvas.value.getActiveObjects();
  activeObjects.forEach((obj) => canvas.value!.remove(obj));
  canvas.value.discardActiveObject();
  canvas.value.renderAll();
  saveHistory();
};

// ===== 圖片加入（可縮放旋轉） =====
const addImage = (file: File) => {
  const c = canvas.value;
  if (!c) return;
  tool.value = 'select'; // 自動切到選取模式，方便立即調整
  c.selection = true;

  const url = URL.createObjectURL(file);
  fabric.Image.fromURL(url).then((img) => {
    URL.revokeObjectURL(url);
    if (!canvas.value) return;

    const targetW = c.getWidth() / 2;
    const scale = targetW / (img.width || targetW);
    img.set({
      left: 40,
      top: 40,
      scaleX: scale,
      scaleY: scale,
      selectable: true,
      evented: true,
      hasControls: true,
      hasBorders: true,
      cornerStyle: 'circle',
      transparentCorners: false,
      lockScalingFlip: false,
      lockRotation: false,
    });
    if ((img as any).controls?.mtr) (img as any).controls.mtr.visible = true;

    c.add(img);
    img.setCoords();
    c.setActiveObject(img);
    c.requestRenderAll();
    saveHistory();
  });
};

// ===== Resize （讓 canvas 填滿容器） =====
const fitCanvas = () => {
  const c = canvas.value;
  const container = document.getElementById('canvasRef')
    ?.parentElement as HTMLElement | null;
  if (!c || !container) return;
  const w = Math.max(320, container.clientWidth);
  const h = Math.max(300, window.innerHeight - 250);
  c.setWidth(w);
  c.setHeight(h);
  c.renderAll();
};

const onResize = () => fitCanvas();

onMounted(() => {
  canvas.value = new fabric.Canvas('canvasRef', {
    width: 1200,
    height: 600,
    backgroundColor: '#fff',
  });
  attachCanvasEvents();
  saveHistory();
  window.addEventListener('resize', onResize);
  // 初始化尺寸與筆刷屬性
  fitCanvas();
  applyBrushProps();
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', onResize);
  canvas.value?.dispose();
});
</script>

<style scoped></style>
