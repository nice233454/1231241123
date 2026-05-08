<template>
  <div class="text-editor">
    <ToolBar
      :zoom="zoom"
      :font-family="fontFamily"
      :font-size="fontSize"
      :background-color="backgroundColor"
      :orientation="orientation"
      @undo="handleUndo"
      @redo="handleRedo"
      @zoom-change="handleZoomChange"
      @print="handlePrint"
      @background-color-change="handleBackgroundColorChange"
      @font-family-change="handleFontFamilyChange"
      @font-size-change="handleFontSizeChange"
      @format="handleFormat"
      @align="handleAlign"
      @text-size="handleTextSize"
      @insert-image="handleInsertImage"
      @upload-image="handleUploadImage"
      @text-color-change="handleTextColorChange"
      @highlight-change="handleHighlightChange"
      @text-background-change="handleTextBackgroundChange"
      @orientation-change="handleOrientationChange"
      @insert-emoji="handleInsertEmoji"
      @insert-link="handleInsertLink"
      @insert-table="handleInsertTable"
      @toggle-header="handleToggleHeader"
      @toggle-footer="handleToggleFooter"
    />
    <EditorContent
      ref="editorContent"
      :zoom="zoom"
      :background-color="backgroundColor"
      :left-margin="leftMargin"
      :right-margin="rightMargin"
      :orientation="orientation"
      @content-change="handleContentChange"
      @margin-change="handleMarginChange"
      @reinit-handlers="reinitAllHandlers"
    />
  </div>
</template>

<script lang="ts">
import ToolBar from './ToolBar.vue'
import EditorContent from './EditorContent.vue'

export default {
  name: 'TextEditor',
  components: {
    ToolBar,
    EditorContent
  },
  data() {
    return {
      zoom: 100,
      fontFamily: 'Arial',
      fontSize: 20,
      backgroundColor: '#ffffff',
      leftMargin: 1,
      rightMargin: 0,
      content: '',
      orientation: 'portrait'
    }
  },
  methods: {
    handleUndo() {
      document.execCommand('undo', false)
      this.emitChange('undo', null)
    },
    handleRedo() {
      document.execCommand('redo', false)
      this.emitChange('redo', null)
    },
    handleZoomChange(value: number) {
      this.zoom = value
      this.emitChange('zoom', value)
    },
    handlePrint() {
      const style = document.createElement('style')
      style.textContent = `
        @media print {
          @page {
            size: ${this.orientation === 'landscape' ? 'landscape' : 'portrait'};
          }
        }
      `
      document.head.appendChild(style)
      window.print()
      document.head.removeChild(style)
      this.emitChange('print', null)
    },
    handleBackgroundColorChange(color: string) {
      this.backgroundColor = color
      this.emitChange('backgroundColor', color)
    },
    handleFontFamilyChange(font: string) {
      this.fontFamily = font
      document.execCommand('fontName', false, font)
      this.emitChange('fontFamily', font)
    },
    handleFontSizeChange(size: number) {
      this.fontSize = size

      const selection = window.getSelection()
      if (!selection || selection.rangeCount === 0) {
        return
      }

      const range = selection.getRangeAt(0)
      if (range.collapsed) {
        return
      }

      document.execCommand('fontSize', false, '7')

      const fontElements = document.getElementsByTagName('font')
      for (let i = 0; i < fontElements.length; i++) {
        if (fontElements[i].size === '7') {
          fontElements[i].removeAttribute('size')
          fontElements[i].style.fontSize = size + 'px'
        }
      }

      selection.removeAllRanges()
      selection.addRange(range)

      this.emitChange('fontSize', size)
    },
    handleFormat(command: string) {
      document.execCommand(command, false)
      this.emitChange('format', command)
    },
    handleAlign(alignment: string) {
      document.execCommand(alignment, false)
      this.emitChange('align', alignment)
    },
    handleTextSize(command: string) {
      if (command === 'increase') {
        this.fontSize += 2
      } else {
        this.fontSize = Math.max(8, this.fontSize - 2)
      }
      this.handleFontSizeChange(this.fontSize)
      this.emitChange('textSize', command)
    },
    handleInsertImage() {
      const url = prompt('Введите URL изображения:')
      if (url) {
        this.executeCommand('insertImage', url)
        this.emitChange('insertImage', url)
        this.$nextTick(() => {
          this.makeImagesResizable()
        })
      }
    },
    handleUploadImage() {
      const input = document.createElement('input')
      input.type = 'file'
      input.accept = 'image/*'
      input.onchange = (e: Event) => {
        const file = (e.target as HTMLInputElement).files?.[0]
        if (file) {
          const reader = new FileReader()
          reader.onload = (event: ProgressEvent<FileReader>) => {
            const dataUrl = event.target?.result as string
            this.executeCommand('insertImage', dataUrl)
            this.emitChange('uploadImage', file.name)
            this.$nextTick(() => {
              this.makeImagesResizable()
            })
          }
          reader.readAsDataURL(file)
        }
      }
      input.click()
    },
    handleTextColorChange(color: string) {
      const selection = window.getSelection()
      if (selection && selection.rangeCount > 0) {
        const range = selection.getRangeAt(0)
        this.executeCommand('foreColor', color)
        if (!range.collapsed) {
          selection.removeAllRanges()
          selection.addRange(range)
        }
      }
      this.emitChange('textColor', color)
    },
    handleHighlightChange(color: string) {
      const selection = window.getSelection()
      if (selection && selection.rangeCount > 0) {
        const range = selection.getRangeAt(0)
        this.executeCommand('hiliteColor', color)
        if (!range.collapsed) {
          selection.removeAllRanges()
          selection.addRange(range)
        }
      }
      this.emitChange('highlight', color)
    },
    handleTextBackgroundChange(color: string) {
      const selection = window.getSelection()
      if (selection && selection.rangeCount > 0) {
        const range = selection.getRangeAt(0)
        this.executeCommand('backColor', color)
        if (!range.collapsed) {
          selection.removeAllRanges()
          selection.addRange(range)
        }
      }
      this.emitChange('textBackground', color)
    },
    handleOrientationChange(orientation: string) {
      this.orientation = orientation
      this.emitChange('orientation', orientation)
    },
    handleInsertEmoji(emoji: string) {
      this.executeCommand('insertText', emoji)
      this.emitChange('insertEmoji', emoji)
    },
    handleInsertLink() {
      const selection = window.getSelection()
      let linkText = ''

      if (selection && selection.rangeCount > 0) {
        const range = selection.getRangeAt(0)
        linkText = range.toString().trim()
      }

      if (!linkText) {
        linkText = prompt('Введите текст ссылки:') || ''
        if (!linkText) return
      }

      const url = prompt('Введите URL:', 'https://')
      if (url) {
        if (selection && selection.rangeCount > 0) {
          const range = selection.getRangeAt(0)
          if (!range.collapsed) {
            this.executeCommand('createLink', url)
          } else {
            const link = document.createElement('a')
            link.href = url
            link.textContent = linkText
            link.style.color = '#0066cc'
            link.style.textDecoration = 'underline'
            range.insertNode(link)
            range.collapse(false)
          }
        } else {
          const link = `<a href="${url}" style="color: #0066cc; text-decoration: underline;">${linkText}</a>`
          this.executeCommand('insertHTML', link)
        }
        this.emitChange('insertLink', { url, text: linkText })
      }
    },
    handleInsertTable(data: any) {
      const rows = data.rows
      const cols = data.cols

      const tableStyle = 'border: 1px solid #000; border-collapse: collapse; table-layout: fixed; width: 100%; margin: 10px 0;'
      const cellStyle = 'border: 1px solid #000; padding: 8px; min-width: 50px; max-width: 200px; word-wrap: break-word; overflow-wrap: break-word; white-space: normal; overflow: hidden; text-overflow: ellipsis; position: relative;'

      let table = `<table contenteditable="true" style="${tableStyle}">`
      for (let i = 0; i < rows; i++) {
        table += '<tr>'
        for (let j = 0; j < cols; j++) {
          table += `<td contenteditable="true" style="${cellStyle}">&nbsp;</td>`
        }
        table += '</tr>'
      }
      table += '</table><p><br></p>'
      this.executeCommand('insertHTML', table)
      this.emitChange('insertTable', { rows, cols })

      this.$nextTick(() => {
        this.makeTablesEditable()
        this.makeTableColumnsResizable()
      })
    },
    makeTablesEditable() {
      const editor = this.$refs.editorContent as any
      if (editor && editor.$el) {
        const tables = editor.$el.querySelectorAll('.editor-content table')
        tables.forEach((table: HTMLTableElement) => {
          if (!table.hasAttribute('data-editable-setup')) {
            table.setAttribute('data-editable-setup', 'true')
            table.addEventListener('contextmenu', (e: MouseEvent) => {
              e.preventDefault()
              const target = e.target as HTMLElement
              if (target.tagName === 'TD' || target.tagName === 'TH') {
                this.showTableCellMenu(table, target)
              }
            })
          }
        })
      }
    },
    showTableCellMenu(table: HTMLTableElement, cell: HTMLElement) {
      const action = prompt('Действие:\n1. Добавить строку выше\n2. Добавить строку ниже\n3. Добавить столбец слева\n4. Добавить столбец справа\n5. Удалить строку\n6. Удалить столбец\n\nВведите номер:', '1')

      if (!action) return

      const tr = cell.parentElement as HTMLTableRowElement
      const cellIndex = Array.from(tr.children).indexOf(cell)
      const rowIndex = Array.from(table.rows).indexOf(tr)

      switch(action) {
        case '1':
          this.addTableRow(table, rowIndex, false)
          break
        case '2':
          this.addTableRow(table, rowIndex, true)
          break
        case '3':
          this.addTableColumn(table, cellIndex, false)
          break
        case '4':
          this.addTableColumn(table, cellIndex, true)
          break
        case '5':
          if (table.rows.length > 1) {
            table.deleteRow(rowIndex)
          }
          break
        case '6':
          if (table.rows[0].cells.length > 1) {
            for (let i = 0; i < table.rows.length; i++) {
              table.rows[i].deleteCell(cellIndex)
            }
          }
          break
      }
    },
    addTableRow(table: HTMLTableElement, index: number, after: boolean) {
      const newRow = table.insertRow(after ? index + 1 : index)
      const cellCount = table.rows[index].cells.length
      const cellStyle = (table.rows[index].cells[0] as HTMLTableCellElement).style.cssText

      for (let i = 0; i < cellCount; i++) {
        const cell = newRow.insertCell()
        cell.contentEditable = 'true'
        cell.style.cssText = cellStyle
        cell.innerHTML = '&nbsp;'
      }
    },
    addTableColumn(table: HTMLTableElement, index: number, after: boolean) {
      const cellStyle = (table.rows[0].cells[index] as HTMLTableCellElement).style.cssText

      for (let i = 0; i < table.rows.length; i++) {
        const cell = table.rows[i].insertCell(after ? index + 1 : index)
        cell.contentEditable = 'true'
        cell.style.cssText = cellStyle
        cell.innerHTML = '&nbsp;'
      }
    },
    makeTableColumnsResizable() {
      const editor = this.$refs.editorContent as any
      if (editor && editor.$el) {
        const tables = editor.$el.querySelectorAll('.editor-content table')
        tables.forEach((table: HTMLTableElement) => {
          if (!table.hasAttribute('data-resizable-setup')) {
            table.setAttribute('data-resizable-setup', 'true')

            const rows = table.rows
            for (let r = 0; r < rows.length; r++) {
              const cells = rows[r].cells
              for (let c = 0; c < cells.length; c++) {
                const cell = cells[c] as HTMLTableCellElement
                this.makeTableCellResizable(cell, table)
              }
            }
          }
        })
      }
    },
    makeTableCellResizable(cell: HTMLTableCellElement, table: HTMLTableElement) {
      cell.style.position = 'relative'

      cell.addEventListener('mousemove', (e: MouseEvent) => {
        const rect = cell.getBoundingClientRect()
        const isNearRightEdge = e.clientX > rect.right - 5

        if (isNearRightEdge) {
          cell.style.cursor = 'col-resize'
        } else {
          cell.style.cursor = 'text'
        }
      })

      cell.addEventListener('mousedown', (e: MouseEvent) => {
        const rect = cell.getBoundingClientRect()
        const isNearRightEdge = e.clientX > rect.right - 5

        if (isNearRightEdge) {
          e.preventDefault()
          e.stopPropagation()

          const startX = e.clientX
          const startWidth = cell.offsetWidth
          const cellIndex = Array.from(cell.parentElement!.children).indexOf(cell)

          const doDrag = (e: MouseEvent) => {
            const newWidth = startWidth + (e.clientX - startX)
            const constrainedWidth = Math.max(50, Math.min(newWidth, 500))

            for (let i = 0; i < table.rows.length; i++) {
              const targetCell = table.rows[i].cells[cellIndex] as HTMLTableCellElement
              if (targetCell) {
                targetCell.style.width = constrainedWidth + 'px'
                targetCell.style.minWidth = constrainedWidth + 'px'
                targetCell.style.maxWidth = constrainedWidth + 'px'
              }
            }
          }

          const stopDrag = () => {
            document.removeEventListener('mousemove', doDrag)
            document.removeEventListener('mouseup', stopDrag)
          }

          document.addEventListener('mousemove', doDrag)
          document.addEventListener('mouseup', stopDrag)
        }
      })
    },
    executeCommand(command: string, value?: string) {
      const editor = this.$refs.editorContent as any
      if (editor && editor.executeCommand) {
        editor.executeCommand(command, value)
      } else {
        document.execCommand(command, false, value)
      }
    },
    makeImagesResizable() {
      const editor = this.$refs.editorContent as any
      if (editor && editor.makeImagesResizable) {
        editor.makeImagesResizable()
      }
    },
    handleContentChange(content: string) {
      this.content = content
      this.emitChange('content', content)
    },
    handleMarginChange(margins: any) {
      this.leftMargin = margins.left
      this.rightMargin = margins.right
      this.emitChange('margins', margins)
    },
    handleToggleHeader() {
      const editor = this.$refs.editorContent as any
      if (editor && editor.toggleAllHeaders) {
        editor.toggleAllHeaders()
      }
      this.emitChange('toggleHeaderControls', null)
    },
    handleToggleFooter() {
      const editor = this.$refs.editorContent as any
      if (editor && editor.toggleAllFooters) {
        editor.toggleAllFooters()
      }
      this.emitChange('toggleHeaderControls', null)
    },
    emitChange(type: string, value: any) {
      this.$emit('editor-change', { type, value, timestamp: Date.now() })
    },
    reinitAllHandlers() {
      this.$nextTick(() => {
        this.makeImagesResizable()
        this.makeTablesEditable()
        this.makeTableColumnsResizable()
      })
    }
  },
  mounted() {
    this.makeImagesResizable()
    this.makeTablesEditable()
    this.makeTableColumnsResizable()
  }
}
</script>

<style scoped>
.text-editor {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #f5f5f5;
}

@media print {
  .text-editor {
    background-color: white;
  }
}
</style>
