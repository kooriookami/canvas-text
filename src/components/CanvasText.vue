<template>
  <div class="canvas-text-container">
    <div class="canvas-text">
      <el-alert
        v-if="loading"
        effect="dark"
        title="字体加载中..."
        :closable="false"
      />
      <canvas ref="canvas" />
    </div>
    <div class="form">
      <div class="form-header">
        <div class="form-title">
          <span>文本排版 - Canvas Text</span>
          <Icon
            class="github-icon"
            icon="ri:github-fill"
            width="24"
            height="24"
            @click="toGithub"
          />
        </div>
        <div class="form-description">
          <span>一个原生 Canvas 排版的示例</span>
        </div>
      </div>

      <div class="form-main">
        <el-form :model="form" label-width="auto">
          <el-form-item label="宽度">
            <el-input-number
              v-model="form.width"
              :min="0"
              :max="2000"
              :precision="0"
            />
          </el-form-item>
          <el-form-item label="高度">
            <el-input-number
              v-model="form.height"
              :min="0"
              :max="2000"
              :precision="0"
            />
          </el-form-item>
          <el-form-item label="字号">
            <el-input-number
              v-model="form.fontSize"
              :min="0"
              :max="200"
              :precision="0"
            />
          </el-form-item>
          <el-form-item label="行距">
            <el-input-number
              v-model="form.lineHeight"
              :min="0"
              :max="10"
              :step="0.1"
            />
          </el-form-item>
          <el-form-item label="文本">
            <el-input
              v-model="form.text"
              style="margin-top: 10px"
              type="textarea"
              :autosize="{minRows: 3}"
              placeholder="请输入文本"
            />
          </el-form-item>
          <el-form-item label="字框">
            <el-switch v-model="form.border" />
          </el-form-item>
        </el-form>
      </div>
    </div>
  </div>
</template>

<script setup>
  import { Icon } from '@iconify/vue';
  import opentype from 'opentype.js';
  import { computed, onMounted, reactive, ref, watch } from 'vue';
  import fontUrl from '@/assets/font/SmileySans-Oblique.otf';

  let font = null;
  const loading = ref(false);
  const canvas = ref(null);
  const ctx = ref(null);
  const form = reactive({
    width: 1440,
    height: 800,
    fontSize: 48,
    lineHeight: 1.5,
    text: 'Canvas API 提供了一个通过JavaScript 和 HTML的<canvas>元素来绘制图形的方式。它可以用于动画、游戏画面、数据可视化、图片编辑以及实时视频处理等方面。\n' +
      'Canvas API 主要聚焦于 2D 图形。而同样使用<canvas>元素的 WebGL API 则用于绘制硬件加速的 2D 和 3D 图形。\n' +
      '\n' +
      'The Canvas API provides a means for drawing graphics via JavaScript and the HTML <canvas> element. Among other things, it can be used for animation, game graphics, data visualization, photo manipulation, and real-time video processing.\n' +
      'The Canvas API largely focuses on 2D graphics. The WebGL API, which also uses the <canvas> element, draws hardware-accelerated 2D and 3D graphics.',
    border: false,
  });
  const lastX = ref(0);
  const lastY = ref(0);
  const lastGlyph = ref(null);

  loadFont();

  function loadFont() {
    loading.value = true;
    const buffer = fetch(fontUrl).then(res => res.arrayBuffer());
    buffer.then(data => {
      font = opentype.parse(data);
      drawText();
    }).finally(() => {
      loading.value = false;
    });
  }

  function drawText() {
    if (!font) return;
    canvas.value.width = form.width;
    canvas.value.height = form.height;
    lastX.value = 0;
    lastY.value = font.ascender * fontRatio.value + halfLineHeight.value;
    lastGlyph.value = font.charToGlyph('');

    const charList = Array.from(form.text);
    // 绘制单个文字 + kerning 控制
    charList.forEach(char => {
      drawChar(char);
    });
  }

  function drawChar(char) {
    if (char === '\n') {
      newLine();
    } else {
      const glyph = font.charToGlyph(char);
      let kerning = font.getKerningValue(lastGlyph, glyph) * fontRatio.value;
      const actualWidth = glyph.advanceWidth * fontRatio.value + kerning;
      if (lastX.value + actualWidth > form.width) {
        newLine();
        kerning = 0;
      }
      font.draw(ctx.value, char, lastX.value + kerning, lastY.value, form.fontSize);
      if (form.border) {
        ctx.value.strokeStyle = '#409EFF';
        ctx.value.strokeRect(lastX.value + kerning, lastY.value - form.fontSize, actualWidth, form.fontSize - (font.descender * fontRatio.value));
      }
      lastGlyph.value = glyph;
      lastX.value += actualWidth;
    }
  }

  function newLine() {
    lastX.value = 0;
    lastY.value += (font.ascender - font.descender) * fontRatio.value + halfLineHeight.value;
  }

  function toGithub() {
    open('https://github.com/kooriookami/canvas-text');
  }

  const fontRatio = computed(() => {
    if (font) {
      return form.fontSize / font.unitsPerEm;
    }
    return 1;
  });

  const halfLineHeight = computed(() => form.fontSize * (form.lineHeight - 1) / 2);

  onMounted(() => {
    ctx.value = canvas.value.getContext('2d');
  });

  watch(() => form, () => {
    drawText();
  }, {
    deep: true,
  });
</script>

<style lang="scss" scoped>
.canvas-text-container {
  height: 100vh;
  display: flex;
  overflow: hidden;

  .canvas-text {
    height: 100%;
    overflow: auto;
    flex-grow: 1;
    position: relative;
    padding: 20px;

    .el-alert {
      margin-bottom: 20px;
      background: #409EFF;
    }

    canvas {
      display: inline-flex;
      box-shadow: 0 2px 4px rgba(0, 0, 0, .12), 0 0 6px rgba(0, 0, 0, .04);
    }
  }

  .form {
    height: 100%;
    overflow: auto;
    width: 400px;
    flex-shrink: 0;
    border-left: 1px solid var(--border-color);

    .form-header {
      padding: 30px 20px;
      font-size: 18px;
      font-weight: bold;
      border-bottom: 1px solid var(--border-color);

      .form-title {
        display: flex;
        flex-wrap: wrap;
        align-items: center;

        .github-icon {
          margin-left: 5px;
          cursor: pointer;
        }
      }

      .form-description {
        margin-top: 20px;
        font-size: 12px;
        font-weight: normal;
        color: var(--info-color);
      }
    }

    .form-main {
      padding: 20px;

      ::v-deep(.el-form) {
        .el-form-item {
          .tip {
            margin-left: 10px;
            color: var(--normal-color);
          }
        }
      }
    }
  }
}
</style>
