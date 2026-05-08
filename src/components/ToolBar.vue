<template>
  <div class="toolbar" @mousedown="handleToolbarMouseDown">
    <div class="toolbar-group">
      <button @click="$emit('undo')" class="tool-btn" title="Отменить">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M3 7v6h6"/>
          <path d="M21 17a9 9 0 00-9-9 9 9 0 00-6 2.3L3 13"/>
        </svg>
      </button>
      <button @click="$emit('redo')" class="tool-btn" title="Повторить">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M21 7v6h-6"/>
          <path d="M3 17a9 9 0 019-9 9 9 0 016 2.3l3 2.7"/>
        </svg>
      </button>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <select v-model="localZoom" @change="changeZoom" class="zoom-select">
        <option value="50">50%</option>
        <option value="75">75%</option>
        <option value="100">100%</option>
        <option value="125">125%</option>
        <option value="150">150%</option>
        <option value="200">200%</option>
      </select>
      <button @click="$emit('print')" class="tool-btn" title="Печать">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M6 9V2h12v7"/>
          <path d="M6 18H4a2 2 0 01-2-2v-5a2 2 0 012-2h16a2 2 0 012 2v5a2 2 0 01-2 2h-2"/>
          <path d="M6 14h12v8H6z"/>
        </svg>
      </button>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <div class="color-picker-wrapper" title="Цвет фона страницы">
        <input
          type="color"
          v-model="localBackgroundColor"
          @input="changeBackgroundColor"
          class="color-input"
        />
        <div class="color-preview" :style="{ backgroundColor: localBackgroundColor }"></div>
      </div>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <div class="color-picker-wrapper" title="Цвет текста">
        <input
          type="color"
          v-model="textColor"
          @focus="saveSelection"
          @change="changeTextColor"
          class="color-input"
        />
        <div class="color-preview color-text-icon" :style="{ color: textColor }">A</div>
      </div>
      <div class="color-picker-wrapper" title="Цвет выделения текста">
        <input
          type="color"
          v-model="highlightColor"
          @focus="saveSelection"
          @change="changeHighlight"
          class="color-input"
        />
        <div class="color-preview highlight-icon" :style="{ backgroundColor: highlightColor }">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M12 20h9M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4L16.5 3.5z"/>
          </svg>
        </div>
      </div>
      <div class="color-picker-wrapper" title="Цвет фона текста">
        <input
          type="color"
          v-model="textBackgroundColor"
          @focus="saveSelection"
          @change="changeTextBackground"
          class="color-input"
        />
        <div class="color-preview bg-text-icon" :style="{ backgroundColor: textBackgroundColor }">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <rect x="3" y="3" width="18" height="18" rx="2"/>
          </svg>
        </div>
      </div>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <select v-model="localFontFamily" @change="changeFontFamily" class="font-select">
        <option value="Arial">Arial</option>
        <option value="Times New Roman">Times New Roman</option>
        <option value="Courier New">Courier New</option>
        <option value="Georgia">Georgia</option>
        <option value="Verdana">Verdana</option>
        <option value="Comic Sans MS">Comic Sans MS</option>
      </select>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <div class="font-size-combo">
        <select v-model="localFontSize" @focus="saveSelection" @change="changeFontSize" class="size-select">
          <option v-for="size in fontSizes" :key="size" :value="size">{{ size }}</option>
        </select>
        <input
          type="number"
          v-model.number="fontSizeInput"
          @focus="saveSelection"
          @input="changeFontSizeFromInput"
          @keypress.enter="applyFontSizeFromInput"
          @blur="applyFontSizeFromInput"
          class="size-input-combo"
          min="1"
          title="Размер шрифта"
        />
      </div>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <button @click="$emit('format', 'bold')" class="tool-btn" title="Жирный">
        <strong>B</strong>
      </button>
      <button @click="$emit('format', 'italic')" class="tool-btn" title="Курсив">
        <em>I</em>
      </button>
      <button @click="$emit('format', 'strikethrough')" class="tool-btn" title="Зачеркнутый">
        <span style="text-decoration: line-through">S</span>
      </button>
      <button @click="$emit('format', 'underline')" class="tool-btn" title="Подчеркнутый">
        <span style="text-decoration: underline">U</span>
      </button>
      <button @click="$emit('format', 'removeFormat')" class="tool-btn" title="Очистить форматирование">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M4 7h16M9 20h6M9 7v13"/>
          <line x1="18" y1="6" x2="6" y2="18"/>
        </svg>
      </button>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <button @click="$emit('align', 'justifyLeft')" class="tool-btn" title="Влево">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <line x1="3" y1="6" x2="21" y2="6"/>
          <line x1="3" y1="12" x2="15" y2="12"/>
          <line x1="3" y1="18" x2="21" y2="18"/>
        </svg>
      </button>
      <button @click="$emit('align', 'justifyCenter')" class="tool-btn" title="По центру">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <line x1="3" y1="6" x2="21" y2="6"/>
          <line x1="6" y1="12" x2="18" y2="12"/>
          <line x1="3" y1="18" x2="21" y2="18"/>
        </svg>
      </button>
      <button @click="$emit('align', 'justifyRight')" class="tool-btn" title="Вправо">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <line x1="3" y1="6" x2="21" y2="6"/>
          <line x1="9" y1="12" x2="21" y2="12"/>
          <line x1="3" y1="18" x2="21" y2="18"/>
        </svg>
      </button>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <button @click="$emit('text-size', 'increase')" class="tool-btn" title="Увеличить">
        <span style="font-size: 18px">A↑</span>
      </button>
      <button @click="$emit('text-size', 'decrease')" class="tool-btn" title="Уменьшить">
        <span style="font-size: 14px">A↓</span>
      </button>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <button @click="$emit('insert-image')" class="tool-btn" title="Вставить изображение по URL">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <rect x="3" y="3" width="18" height="18" rx="2"/>
          <circle cx="8.5" cy="8.5" r="1.5"/>
          <path d="M21 15l-5-5L5 21"/>
        </svg>
      </button>
      <button @click="$emit('upload-image')" class="tool-btn" title="Загрузить изображение с компьютера">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
          <polyline points="17 8 12 3 7 8"/>
          <line x1="12" y1="3" x2="12" y2="15"/>
        </svg>
      </button>
      <div class="table-selector-wrapper">
        <button @click="showTableGrid = !showTableGrid" class="tool-btn" title="Вставить таблицу">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <rect x="3" y="3" width="18" height="18" rx="2"/>
            <line x1="3" y1="9" x2="21" y2="9"/>
            <line x1="3" y1="15" x2="21" y2="15"/>
            <line x1="9" y1="3" x2="9" y2="21"/>
            <line x1="15" y1="3" x2="15" y2="21"/>
          </svg>
        </button>
        <div v-if="showTableGrid" class="table-grid-popup">
          <div class="table-grid-label">{{ hoverRows }} x {{ hoverCols }}</div>
          <div class="table-grid">
            <div
              v-for="row in 10"
              :key="row"
              class="table-grid-row"
            >
              <div
                v-for="col in 10"
                :key="col"
                class="table-grid-cell"
                :class="{ 'active': row <= hoverRows && col <= hoverCols }"
                @mouseenter="hoverRows = row; hoverCols = col"
                @click="insertTable(row, col)"
              ></div>
            </div>
          </div>
        </div>
      </div>
      <button @click="$emit('insert-link')" class="tool-btn" title="Вставить гиперссылку">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/>
          <path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/>
        </svg>
      </button>
      <button @click="showEmojiPicker = !showEmojiPicker" class="tool-btn" title="Вставить эмодзи">
        <span style="font-size: 18px">😀</span>
      </button>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <button @click="$emit('format', 'insertUnorderedList')" class="tool-btn" title="Маркированный список">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <line x1="8" y1="6" x2="21" y2="6"/>
          <line x1="8" y1="12" x2="21" y2="12"/>
          <line x1="8" y1="18" x2="21" y2="18"/>
          <circle cx="4" cy="6" r="1" fill="currentColor"/>
          <circle cx="4" cy="12" r="1" fill="currentColor"/>
          <circle cx="4" cy="18" r="1" fill="currentColor"/>
        </svg>
      </button>
      <button @click="$emit('format', 'insertOrderedList')" class="tool-btn" title="Нумерованный список">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <line x1="10" y1="6" x2="21" y2="6"/>
          <line x1="10" y1="12" x2="21" y2="12"/>
          <line x1="10" y1="18" x2="21" y2="18"/>
          <path d="M4 6h1v4"/>
          <path d="M4 10h2"/>
          <path d="M6 18H4c0-1 2-2 2-3s-1-1.5-2-1"/>
        </svg>
      </button>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <select v-model="localOrientation" @change="changeOrientation" class="orientation-select">
        <option value="portrait">Книжная</option>
        <option value="landscape">Альбомная</option>
      </select>
    </div>

    <div class="toolbar-separator"></div>

    <div class="toolbar-group">
      <button @click="$emit('toggle-header')" class="tool-btn" title="Показать/Скрыть элементы управления колонтитулами">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <rect x="3" y="3" width="18" height="18" rx="2"/>
          <line x1="3" y1="8" x2="21" y2="8"/>
          <line x1="3" y1="16" x2="21" y2="16"/>
        </svg>
      </button>
    </div>

    <div v-if="showEmojiPicker" class="emoji-picker">
      <button
        v-for="emoji in emojis"
        :key="emoji"
        @click="insertEmoji(emoji)"
        class="emoji-btn"
      >{{ emoji }}</button>
    </div>
  </div>
</template>

<script lang="ts">
export default {
  name: 'ToolBar',
  props: {
    zoom: {
      type: Number,
      default: 100
    },
    fontFamily: {
      type: String,
      default: 'Arial'
    },
    fontSize: {
      type: Number,
      default: 20
    },
    backgroundColor: {
      type: String,
      default: '#ffffff'
    },
    orientation: {
      type: String,
      default: 'portrait'
    }
  },
  data() {
    return {
      localZoom: this.zoom,
      localFontFamily: this.fontFamily,
      localFontSize: this.fontSize,
      localBackgroundColor: this.backgroundColor,
      localOrientation: this.orientation,
      fontSizes: [8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 36, 48, 72],
      fontSizeInput: this.fontSize,
      textColor: '#000000',
      highlightColor: '#ffff00',
      textBackgroundColor: '#ffffff',
      showEmojiPicker: false,
      showTableGrid: false,
      hoverRows: 1,
      hoverCols: 1,
      emojis: ['😀', '😃', '😄', '😁', '😆', '😅', '🤣', '😂', '🙂', '🙃', '😉', '😊', '😇', '🥰', '😍', '🤩', '😘', '😗', '😚', '😙', '🥲', '😋', '😛', '😜', '🤪', '😝', '🤑', '🤗', '🤭', '🤫', '🤔', '🤐', '🤨', '😐', '😑', '😶', '😏', '😒', '🙄', '😬', '🤥', '😌', '😔', '😪', '🤤', '😴', '👍', '👎', '👌', '✌️', '🤞', '🤟', '🤘', '🤙', '👈', '👉', '👆', '👇', '☝️', '✋', '🤚', '🖐️', '🖖', '👋', '🤝', '🙏', '💪', '🦾', '🦿', '🦵', '🦶', '👂', '🦻', '👃', '❤️', '🧡', '💛', '💚', '💙', '💜', '🖤', '🤍', '🤎', '💔', '❣️', '💕', '💞', '💓', '💗', '💖', '💘', '💝'],
      savedRange: null as Range | null
    }
  },
  watch: {
    zoom(val) {
      this.localZoom = val
    },
    fontFamily(val) {
      this.localFontFamily = val
    },
    fontSize(val) {
      this.fontSizeInput = val
      this.localFontSize = val
    },
    backgroundColor(val) {
      this.localBackgroundColor = val
    },
    orientation(val) {
      this.localOrientation = val
    }
  },
  methods: {
    handleToolbarMouseDown(event: MouseEvent) {
      const target = event.target as HTMLElement
      if (!target.matches('input, select, button, .tool-btn, .emoji-btn, .table-grid-cell, svg, path, line, rect, circle, polyline, polygon')) {
        event.preventDefault()
        this.saveSelection()
      } else {
        this.saveSelection()
      }
    },
    changeZoom() {
      this.$emit('zoom-change', parseInt(this.localZoom.toString()))
    },
    changeBackgroundColor() {
      this.$emit('background-color-change', this.localBackgroundColor)
    },
    changeFontFamily() {
      this.$emit('font-family-change', this.localFontFamily)
    },
    changeFontSize() {
      const size = parseInt(this.localFontSize.toString())
      this.fontSizeInput = size
      this.restoreSelection()
      this.$emit('font-size-change', size)
    },
    changeFontSizeFromInput() {
      if (this.fontSizeInput >= 1) {
        this.localFontSize = this.fontSizeInput
      }
    },
    applyFontSizeFromInput() {
      if (this.fontSizeInput >= 1) {
        this.localFontSize = this.fontSizeInput
        this.restoreSelection()
        this.$emit('font-size-change', this.fontSizeInput)
      }
    },
    saveSelection() {
      const selection = window.getSelection()
      if (selection && selection.rangeCount > 0) {
        this.savedRange = selection.getRangeAt(0).cloneRange()
      }
    },
    restoreSelection() {
      if (this.savedRange) {
        const selection = window.getSelection()
        if (selection) {
          selection.removeAllRanges()
          selection.addRange(this.savedRange)
        }
      }
    },
    changeTextColor() {
      requestAnimationFrame(() => {
        this.restoreSelection()
        this.$emit('text-color-change', this.textColor)
      })
    },
    changeHighlight() {
      requestAnimationFrame(() => {
        this.restoreSelection()
        this.$emit('highlight-change', this.highlightColor)
      })
    },
    changeTextBackground() {
      requestAnimationFrame(() => {
        this.restoreSelection()
        this.$emit('text-background-change', this.textBackgroundColor)
      })
    },
    changeOrientation() {
      this.$emit('orientation-change', this.localOrientation)
    },
    insertEmoji(emoji: string) {
      this.$emit('insert-emoji', emoji)
      this.showEmojiPicker = false
    },
    insertTable(rows: number, cols: number) {
      this.$emit('insert-table', { rows, cols })
      this.showTableGrid = false
      this.hoverRows = 1
      this.hoverCols = 1
    }
  }
}
</script>

<style scoped>
.toolbar {
  display: flex;
  align-items: center;
  padding: 8px 16px;
  background-color: #ffffff;
  border-bottom: 1px solid #e0e0e0;
  gap: 8px;
  flex-wrap: wrap;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.toolbar-group {
  display: flex;
  align-items: center;
  gap: 4px;
}

.toolbar-separator {
  width: 1px;
  height: 24px;
  background-color: #e0e0e0;
  margin: 0 4px;
}

.tool-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border: 1px solid #d0d0d0;
  background-color: #fafafa;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
  color: #333;
}

.tool-btn:hover {
  background-color: #e8e8e8;
  border-color: #999;
}

.tool-btn:active {
  background-color: #d0d0d0;
}

.zoom-select,
.font-select,
.size-select {
  padding: 6px 8px;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  background-color: #fafafa;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
}

.zoom-select {
  width: 80px;
}

.font-select {
  width: 140px;
}

.size-select {
  width: 60px;
}

.font-size-combo {
  display: flex;
  gap: 4px;
  align-items: center;
}

.size-input-combo {
  width: 50px;
  padding: 6px 4px;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  background-color: #fafafa;
  font-size: 14px;
  transition: all 0.2s;
}

.size-input-combo:hover {
  border-color: #999;
  background-color: #fff;
}

.size-input-combo:focus {
  outline: none;
  border-color: #2196F3;
}

.orientation-select {
  padding: 6px 8px;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  background-color: #fafafa;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
  width: 120px;
}

.orientation-select:hover {
  border-color: #999;
  background-color: #fff;
}

.zoom-select:hover,
.font-select:hover,
.size-select:hover {
  border-color: #999;
  background-color: #fff;
}

.color-picker-wrapper {
  position: relative;
  width: 32px;
  height: 32px;
}

.color-input {
  position: absolute;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
}

.color-preview {
  width: 32px;
  height: 32px;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
}

.color-preview:hover {
  border-color: #999;
  box-shadow: 0 0 4px rgba(0, 0, 0, 0.2);
}

.color-text-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  font-weight: bold;
}

.highlight-icon,
.bg-text-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 4px;
}

.emoji-picker {
  position: absolute;
  top: 60px;
  right: 16px;
  background: white;
  border: 1px solid #d0d0d0;
  border-radius: 8px;
  padding: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  display: grid;
  grid-template-columns: repeat(8, 1fr);
  gap: 4px;
  max-width: 320px;
  max-height: 300px;
  overflow-y: auto;
  z-index: 1000;
}

.emoji-btn {
  width: 36px;
  height: 36px;
  border: 1px solid transparent;
  background: transparent;
  border-radius: 4px;
  cursor: pointer;
  font-size: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.emoji-btn:hover {
  background-color: #f0f0f0;
  border-color: #d0d0d0;
}

.table-selector-wrapper {
  position: relative;
}

.table-grid-popup {
  position: absolute;
  top: 40px;
  left: 0;
  background: white;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  padding: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 1000;
}

.table-grid-label {
  text-align: center;
  font-size: 12px;
  color: #666;
  margin-bottom: 8px;
  font-weight: 500;
}

.table-grid {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.table-grid-row {
  display: flex;
  gap: 2px;
}

.table-grid-cell {
  width: 20px;
  height: 20px;
  border: 1px solid #d0d0d0;
  cursor: pointer;
  transition: all 0.1s;
}

.table-grid-cell:hover {
  border-color: #2196F3;
}

.table-grid-cell.active {
  background-color: #bbdefb;
  border-color: #2196F3;
}

@media print {
  .toolbar {
    display: none !important;
  }
}
</style>
