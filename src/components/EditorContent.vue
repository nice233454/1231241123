<template>
  <div class="editor-wrapper">
    <div class="ruler-horizontal">
      <div class="ruler-content" :style="rulerHorizontalStyle">
        <div class="ruler-page-indicator" :style="{ width: pageWidthPx + 'px' }"></div>
        <div
          v-for="i in Math.ceil(pageWidthCm * 10)"
          :key="i"
          class="ruler-mark"
          :class="{ 'ruler-mark-major': i % 10 === 0 }"
          :style="{ left: (i * (cmToPx / 10)) + 'px' }"
        >
          <span v-if="i % 10 === 0" class="ruler-number">{{ i / 10 }}</span>
        </div>
        <div
          class="ruler-marker ruler-marker-left"
          :style="{ left: leftMarkerPosition + 'px' }"
          @mousedown="startDragMargin('left', $event)"
        ></div>
        <div
          class="ruler-marker ruler-marker-right"
          :style="{ left: rightMarkerPosition + 'px' }"
          @mousedown="startDragMargin('right', $event)"
        ></div>
      </div>
    </div>

    <div class="editor-container">
      <div class="editor-scroll" ref="scrollContainer" @scroll="handleScroll">
        <div class="editor-pages-wrapper">
          <div
            v-for="(page, index) in pages"
            :key="'page-' + index"
            class="editor-page-wrapper"
          >
            <div class="ruler-vertical">
              <div class="ruler-vertical-content">
                <div class="ruler-page-indicator-vertical" :style="{ height: pageHeightPx + 'px' }"></div>
                <div
                  v-for="i in Math.ceil(pageHeightCm * 10)"
                  :key="'v-' + i"
                  class="ruler-mark-vertical"
                  :class="{ 'ruler-mark-major-vertical': i % 10 === 0 }"
                  :style="{ top: (i * (cmToPx / 10)) + 'px' }"
                >
                  <span v-if="i % 10 === 0" class="ruler-number-vertical">{{ i / 10 }}</span>
                </div>
                <div
                  class="ruler-marker-vertical ruler-marker-top"
                  :style="{ top: topMarkerPosition + 'px' }"
                  @mousedown="startDragMargin('top', $event)"
                ></div>
                <div
                  class="ruler-marker-vertical ruler-marker-bottom"
                  :style="{ top: bottomMarkerPosition + 'px' }"
                  @mousedown="startDragMargin('bottom', $event)"
                ></div>
              </div>
            </div>
            <div
              class="editor-page"
              :class="{ 'landscape': orientation === 'landscape' }"
              :style="editorPageStyle"
            >
              <div class="page-header-row">
                <div class="page-label">Лист {{ index + 1 }}</div>
                <div class="page-action-buttons">
                  <button @click="addPage(index)" class="page-action-btn" :data-index="index" title="Добавить страницу">+</button>
                  <button
                    v-if="pages.length > 1"
                    @click="removePage(index)"
                    class="page-action-btn remove"
                    :data-index="index"
                    title="Удалить страницу"
                  >-</button>
                </div>
              </div>
              <div v-if="showHeaderFooterControls" class="page-controls">
                <button
                  v-if="!page.showHeader"
                  @click="toggleHeader(index, true)"
                  class="control-btn"
                  title="Добавить верхний колонтитул"
                >+ Верхний колонтитул</button>
                <button
                  v-if="page.showHeader"
                  @click="toggleHeader(index, false)"
                  class="control-btn remove-btn"
                  title="Удалить верхний колонтитул"
                >- Верхний колонтитул</button>
              </div>
            <div
              v-if="page.showHeader"
              class="page-header"
              contenteditable="true"
              @input="handleHeaderInput(index, $event)"
              :style="headerFooterStyle"
              data-placeholder="Верхний колонтитул"
            ></div>
            <div
              :ref="(el) => setEditorRef(el, index)"
              class="editor-content"
              :data-page-index="index"
              contenteditable="true"
              @input="handleInput"
              @paste="handlePaste"
              @mouseup="handleMouseUp"
              @keyup="handleKeyUp"
              @click="handleClick"
              :style="editorContentStyle"
            ></div>
            <div v-if="showHeaderFooterControls" class="page-controls bottom-controls">
              <button
                v-if="!page.showFooter"
                @click="toggleFooter(index, true)"
                class="control-btn"
                title="Добавить нижний колонтитул"
              >+ Нижний колонтитул</button>
              <button
                v-if="page.showFooter"
                @click="toggleFooter(index, false)"
                class="control-btn remove-btn"
                title="Удалить нижний колонтитул"
              >- Нижний колонтитул</button>
            </div>
              <div
                v-if="page.showFooter"
                class="page-footer"
                contenteditable="true"
                @input="handleFooterInput(index, $event)"
                data-placeholder="Нижний колонтитул"
              ></div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
export default {
  name: 'EditorContent',
  props: {
    zoom: {
      type: Number,
      default: 100
    },
    backgroundColor: {
      type: String,
      default: '#ffffff'
    },
    leftMargin: {
      type: Number,
      default: 1
    },
    rightMargin: {
      type: Number,
      default: 0
    },
    orientation: {
      type: String,
      default: 'portrait'
    }
  },
  data() {
    return {
      content: '',
      draggingMargin: null as string | null,
      dragStartX: 0,
      dragStartY: 0,
      dragStartValue: 0,
      cmToPx: 37.795275591,
      scrollLeft: 0,
      internalLeftMargin: 1,
      internalRightMargin: 0,
      internalTopMargin: 1,
      internalBottomMargin: this.orientation === 'portrait' ? 4.7 : 0,
      pages: [{ header: '', footer: '', content: '', showHeader: false, showFooter: false }] as Array<{header: string, footer: string, content: string, showHeader: boolean, showFooter: boolean}>,
      editorRefs: [] as any[],
      savedSelection: null as Range | null,
      previousContentLength: 0,
      overflowCheckTimeout: null as any,
      showHeaderFooterControls: true,
      pasteInProgress: false,
      pasteTimeout: null as any
    }
  },
  watch: {
    orientation(newVal) {
      if (newVal === 'portrait') {
        this.internalBottomMargin = 4.7
      } else {
        this.internalBottomMargin = 0
      }
    }
  },
  computed: {
    zoomFactor() {
      return this.zoom / 100
    },
    pageWidthCm() {
      return this.orientation === 'portrait' ? 21 : 29.7
    },
    pageWidthPx() {
      return this.orientation === 'portrait' ? 800 : 1000
    },
    pageHeightCm() {
      return this.orientation === 'portrait' ? 29.7 : 21
    },
    pageHeightPx() {
      return this.orientation === 'portrait' ? 1000 : 800
    },
    editorPageStyle() {
      return {
        transform: `scale(${this.zoomFactor})`,
        transformOrigin: 'top left',
        backgroundColor: this.backgroundColor
      }
    },
    editorContentStyle() {
      return {
        paddingLeft: `${this.internalLeftMargin * this.cmToPx}px`,
        paddingRight: `${this.internalRightMargin * this.cmToPx}px`,
        paddingTop: `${this.internalTopMargin * this.cmToPx}px`,
        paddingBottom: `${this.internalBottomMargin * this.cmToPx}px`
      }
    },
    rulerHorizontalStyle() {
      const scrollContainer = this.$refs.scrollContainer as HTMLElement
      let centerOffset = 0

      if (scrollContainer) {
        const containerWidth = scrollContainer.clientWidth
        const pageWidth = this.pageWidthPx
        const rulerWidth = 35
        centerOffset = (containerWidth - pageWidth) / 2 + rulerWidth
      }

      return {
        transform: `translateX(${centerOffset - this.scrollLeft}px)`,
        width: `${this.pageWidthPx}px`
      }
    },
    headerFooterStyle() {
      return {
        paddingLeft: `${this.internalLeftMargin * this.cmToPx}px`,
        paddingRight: `${this.internalRightMargin * this.cmToPx}px`
      }
    },
    leftMarkerPosition() {
      return this.internalLeftMargin * this.cmToPx
    },
    rightMarkerPosition() {
      return this.pageWidthPx - (this.internalRightMargin * this.cmToPx)
    },
    topMarkerPosition() {
      return this.internalTopMargin * this.cmToPx
    },
    bottomMarkerPosition() {
      return this.pageHeightPx - (this.internalBottomMargin * this.cmToPx)
    }
  },
  methods: {
    setEditorRef(el: any, index: number) {
      if (el) {
        this.editorRefs[index] = el
      }
    },
    handleInput(e: Event) {
      const editor = e.target as HTMLElement
      const pageIndex = parseInt(editor.getAttribute('data-page-index') || '0')

      if (!this.pages[pageIndex]) {
        return
      }

      this.pages[pageIndex].content = editor.innerHTML
      this.$emit('content-change', this.pages.map(p => p.content).join('\n'))

      if (this.overflowCheckTimeout) {
        clearTimeout(this.overflowCheckTimeout)
      }

      this.overflowCheckTimeout = setTimeout(() => {
        this.checkAllPagesOverflow()
        this.$emit('reinit-handlers')
      }, 100)
    },
    handleHeaderInput(index: number, e: Event) {
      if (this.pages[index]) {
        this.pages[index].header = (e.target as HTMLElement).innerHTML
      }
    },
    handleFooterInput(index: number, e: Event) {
      if (this.pages[index]) {
        this.pages[index].footer = (e.target as HTMLElement).innerHTML
      }
    },
    checkAllPagesOverflow() {
      for (let i = 0; i < this.pages.length; i++) {
        this.checkPageOverflow(i)
      }
    },
    checkPageOverflow(pageIndex: number) {
      const editor = this.editorRefs[pageIndex]
      if (!editor) return

      const isOverflowing = editor.scrollHeight > editor.clientHeight + 10

      if (!isOverflowing) {
        return
      }

      const nextPageIndex = pageIndex + 1
      if (!this.pages[nextPageIndex]) {
        this.pages.push({
          header: '',
          footer: '',
          content: '',
          showHeader: false,
          showFooter: false
        })
      }

      this.$nextTick(() => {
        this.moveOverflowContent(pageIndex)
      })
    },
    moveOverflowContent(pageIndex: number) {
      const editor = this.editorRefs[pageIndex]
      if (!editor) return

      const nextPageIndex = pageIndex + 1
      if (!this.pages[nextPageIndex]) return

      let moved = false
      let iterations = 0
      const maxIterations = 100

      while (editor.scrollHeight > editor.clientHeight + 10 && iterations < maxIterations) {
        const lastChild = editor.lastChild
        if (!lastChild) break

        const lastChildHTML = lastChild instanceof HTMLElement ? lastChild.outerHTML : lastChild.textContent || ''
        if (!lastChildHTML.trim()) {
          editor.removeChild(lastChild)
          continue
        }

        editor.removeChild(lastChild)
        this.pages[nextPageIndex].content = lastChildHTML + this.pages[nextPageIndex].content
        moved = true
        iterations++
      }

      this.pages[pageIndex].content = editor.innerHTML

      if (moved) {
        this.$nextTick(() => {
          const nextEditor = this.editorRefs[nextPageIndex]
          if (nextEditor) {
            nextEditor.innerHTML = this.pages[nextPageIndex].content
            this.checkPageOverflow(nextPageIndex)
          }
        })
      }
    },
    checkPullContentFromNext() {
      return
    },
    removeEmptyPages() {
      for (let i = this.pages.length - 1; i > 0; i--) {
        const page = this.pages[i]
        const tempDiv = document.createElement('div')
        tempDiv.innerHTML = page.content
        const textContent = tempDiv.textContent || tempDiv.innerText || ''
        const hasImages = tempDiv.querySelectorAll('img').length > 0
        const hasTables = tempDiv.querySelectorAll('table').length > 0
        const isEmpty = !textContent.trim() && !page.header.trim() && !page.footer.trim() && !hasImages && !hasTables

        if (isEmpty) {
          this.pages.splice(i, 1)
        }
      }
    },
    printDocument() {
      window.print()
    },
    toggleHeader(index: number, show: boolean) {
      this.pages[index].showHeader = show
      if (!show) {
        this.pages[index].header = ''
      } else {
        this.$nextTick(() => {
          const pageElements = document.querySelectorAll('.page-header')
          const headerElement = pageElements[index] as HTMLElement
          if (headerElement && this.pages[index].header) {
            headerElement.innerHTML = this.pages[index].header
          }
        })
      }
    },
    toggleFooter(index: number, show: boolean) {
      this.pages[index].showFooter = show
      if (!show) {
        this.pages[index].footer = ''
      } else {
        this.$nextTick(() => {
          const pageElements = document.querySelectorAll('.page-footer')
          const footerIndex = this.pages.slice(0, index + 1).filter(p => p.showFooter).length - 1
          const footerElement = pageElements[footerIndex] as HTMLElement
          if (footerElement && this.pages[index].footer) {
            footerElement.innerHTML = this.pages[index].footer
          }
        })
      }
    },
    toggleAllHeaders() {
      this.showHeaderFooterControls = !this.showHeaderFooterControls
    },
    toggleAllFooters() {
      this.showHeaderFooterControls = !this.showHeaderFooterControls
    },
    handleMouseUp() {
      this.saveSelection()
    },
    handleKeyUp() {
      this.saveSelection()
    },
    handleClick(e: MouseEvent) {
      const target = e.target as HTMLElement
      if (target.tagName === 'A') {
        e.preventDefault()
        if (e.ctrlKey || e.metaKey) {
          const url = (target as HTMLAnchorElement).href
          window.open(url, '_blank')
        } else {
          const newUrl = prompt('Редактировать ссылку:', (target as HTMLAnchorElement).href)
          if (newUrl !== null && newUrl !== (target as HTMLAnchorElement).href) {
            (target as HTMLAnchorElement).href = newUrl
          }
        }
      }
    },
    saveSelection() {
      const selection = window.getSelection()
      if (selection && selection.rangeCount > 0) {
        this.savedSelection = selection.getRangeAt(0)
      }
    },
    restoreSelection() {
      if (this.savedSelection) {
        const selection = window.getSelection()
        if (selection) {
          selection.removeAllRanges()
          selection.addRange(this.savedSelection)
        }
      }
    },
    executeCommand(command: string, value?: string) {
      this.restoreSelection()
      document.execCommand(command, false, value)
      this.saveSelection()
    },
    handlePaste(e: ClipboardEvent) {
      e.preventDefault()

      if (this.pasteInProgress) {
        return
      }

      this.pasteInProgress = true

      if (this.pasteTimeout) {
        clearTimeout(this.pasteTimeout)
      }

      const target = e.target as HTMLElement
      const pageIndex = parseInt(target.getAttribute('data-page-index') || '0')

      const selection = window.getSelection()
      let isInsideTable = false

      if (selection && selection.rangeCount > 0) {
        let node = selection.getRangeAt(0).startContainer
        while (node && node !== target) {
          if (node.nodeName === 'TABLE' || (node as HTMLElement).tagName === 'TABLE') {
            isInsideTable = true
            break
          }
          node = node.parentNode as Node
        }
      }

      let html = e.clipboardData?.getData('text/html')
      const text = e.clipboardData?.getData('text/plain')

      if (html) {
        const tempDiv = document.createElement('div')
        tempDiv.innerHTML = html
        const hasTable = tempDiv.querySelector('table') !== null

        if (isInsideTable && hasTable) {
          alert('Нельзя вставить таблицу внутрь другой таблицы')
          this.pasteInProgress = false
          return
        }

        html = html.replace(/<!--StartFragment-->|<!--EndFragment-->/g, '')
        html = html.replace(/StartFragment|EndFragment/g, '')

        tempDiv.innerHTML = html

        const walker = document.createTreeWalker(
          tempDiv,
          NodeFilter.SHOW_TEXT,
          null
        )

        let node
        while (node = walker.nextNode()) {
          if (node.textContent) {
            node.textContent = node.textContent.replace(/StartFragment|EndFragment/g, '')
          }
        }

        html = tempDiv.innerHTML

        document.execCommand('insertHTML', false, html)

        this.pasteTimeout = setTimeout(() => {
          this.$nextTick(() => {
            this.makeImagesResizable()
            this.pages[pageIndex].content = target.innerHTML
            this.checkAllPagesOverflow()
            this.$emit('reinit-handlers')
            this.pasteInProgress = false
          })
        }, 150)
      } else if (text) {
        const cleanText = text.replace(/StartFragment|EndFragment/g, '')
        document.execCommand('insertText', false, cleanText)

        this.pasteTimeout = setTimeout(() => {
          this.$nextTick(() => {
            this.pages[pageIndex].content = target.innerHTML
            this.checkAllPagesOverflow()
            this.pasteInProgress = false
          })
        }, 150)
      } else {
        this.pasteInProgress = false
      }
    },
    makeImagesResizable() {
      const images = document.querySelectorAll('.editor-content img')
      images.forEach((img: any) => {
        if (!img._hasResizableHandlers) {
          img._hasResizableHandlers = true
          img.setAttribute('data-resizable', 'true')

          const parent = img.closest('.editor-content') as HTMLElement
          if (parent) {
            const maxWidth = parent.clientWidth * 0.9
            if (!img.style.maxWidth) {
              img.style.maxWidth = maxWidth + 'px'
            }
            if (img.naturalWidth > maxWidth && !img.style.width) {
              img.style.width = maxWidth + 'px'
              img.style.height = 'auto'
            }
          }

          img.style.position = 'relative'
          img.style.display = 'inline-block'
          img.style.border = '2px solid transparent'
          img.style.transition = 'border-color 0.2s'

          img.addEventListener('mousedown', (e: MouseEvent) => {
            e.preventDefault()
            const rect = img.getBoundingClientRect()
            const isRightEdge = e.clientX > rect.right - 15
            const isBottomEdge = e.clientY > rect.bottom - 15
            const isBottomRightCorner = isRightEdge && isBottomEdge

            if (isBottomRightCorner) {
              img.style.cursor = 'nwse-resize'
              this.startResize(e, img)
            } else {
              img.style.cursor = 'move'
              this.startDrag(e, img)
            }
          })

          img.addEventListener('mousemove', (e: MouseEvent) => {
            const rect = img.getBoundingClientRect()
            const isRightEdge = e.clientX > rect.right - 15
            const isBottomEdge = e.clientY > rect.bottom - 15
            const isBottomRightCorner = isRightEdge && isBottomEdge

            if (isBottomRightCorner) {
              img.style.cursor = 'nwse-resize'
              img.style.borderColor = '#2196F3'
            } else {
              img.style.cursor = 'move'
              img.style.borderColor = 'transparent'
            }
          })

          img.addEventListener('mouseleave', () => {
            img.style.borderColor = 'transparent'
          })
        }
      })
    },
    startDrag(e: MouseEvent, img: HTMLImageElement) {
      e.preventDefault()
      e.stopPropagation()

      const startX = e.clientX
      const startY = e.clientY
      const startLeft = img.offsetLeft
      const startTop = img.offsetTop

      let parent = img.closest('.editor-content') as HTMLElement
      if (!parent) return

      const originalParent = parent
      const originalPageIndex = parseInt(parent.getAttribute('data-page-index') || '0')

      img.style.position = 'absolute'
      img.style.zIndex = '1000'

      const doDrag = (e: MouseEvent) => {
        const deltaX = e.clientX - startX
        const deltaY = e.clientY - startY
        let newLeft = startLeft + deltaX
        let newTop = startTop + deltaY

        const allPages = document.querySelectorAll('.editor-content')
        let targetPage: HTMLElement | null = null

        allPages.forEach((page: any) => {
          const rect = page.getBoundingClientRect()
          if (e.clientY >= rect.top && e.clientY <= rect.bottom) {
            targetPage = page
          }
        })

        if (targetPage && targetPage !== parent) {
          parent = targetPage
          img.remove()
          parent.appendChild(img)
          const parentRect = parent.getBoundingClientRect()
          newLeft = e.clientX - parentRect.left - img.offsetWidth / 2
          newTop = e.clientY - parentRect.top - img.offsetHeight / 2
        }

        const parentStyle = window.getComputedStyle(parent)
        const paddingRight = parseInt(parentStyle.paddingRight)

        newLeft = Math.max(0, Math.min(newLeft, parent.clientWidth - img.offsetWidth - paddingRight))
        newTop = Math.max(0, Math.min(newTop, parent.clientHeight - img.offsetHeight))

        img.style.left = newLeft + 'px'
        img.style.top = newTop + 'px'
      }

      const stopDrag = () => {
        document.removeEventListener('mousemove', doDrag)
        document.removeEventListener('mouseup', stopDrag)
        img.style.zIndex = '10'

        const currentPageIndex = parseInt(parent.getAttribute('data-page-index') || '0')
        if (currentPageIndex !== originalPageIndex) {
          this.pages[currentPageIndex].content = parent.innerHTML
          this.pages[originalPageIndex].content = originalParent.innerHTML
        } else {
          this.pages[currentPageIndex].content = parent.innerHTML
        }
      }

      document.addEventListener('mousemove', doDrag)
      document.addEventListener('mouseup', stopDrag)
    },
    startResize(e: MouseEvent, img: HTMLImageElement) {
      if (e.button !== 0) return
      e.preventDefault()
      e.stopPropagation()

      const startX = e.clientX
      const startWidth = img.offsetWidth
      const startHeight = img.offsetHeight
      const aspectRatio = startWidth / startHeight

      const parent = img.closest('.editor-content') as HTMLElement
      const maxWidth = parent ? parent.clientWidth * 0.9 : 800

      const doResize = (e: MouseEvent) => {
        const newWidth = startWidth + (e.clientX - startX)
        const constrainedWidth = Math.max(50, Math.min(newWidth, maxWidth))
        const constrainedHeight = constrainedWidth / aspectRatio

        img.style.width = constrainedWidth + 'px'
        img.style.height = constrainedHeight + 'px'
        img.style.borderColor = '#2196F3'
      }

      const stopResize = () => {
        document.removeEventListener('mousemove', doResize)
        document.removeEventListener('mouseup', stopResize)
        img.style.borderColor = 'transparent'

        if (parent) {
          const pageIndex = parseInt(parent.getAttribute('data-page-index') || '0')
          this.pages[pageIndex].content = parent.innerHTML
        }
      }

      document.addEventListener('mousemove', doResize)
      document.addEventListener('mouseup', stopResize)
    },
    startDragMargin(type: string, e: MouseEvent) {
      e.preventDefault()
      this.draggingMargin = type
      this.dragStartX = e.clientX
      this.dragStartY = e.clientY

      if (type === 'left') {
        this.dragStartValue = this.internalLeftMargin
      } else if (type === 'right') {
        this.dragStartValue = this.internalRightMargin
      } else if (type === 'top') {
        this.dragStartValue = this.internalTopMargin
      } else if (type === 'bottom') {
        this.dragStartValue = this.internalBottomMargin
      }

      document.addEventListener('mousemove', this.dragMargin)
      document.addEventListener('mouseup', this.stopDragMargin)
    },
    dragMargin(e: MouseEvent) {
      if (!this.draggingMargin) return

      const deltaX = e.clientX - this.dragStartX
      const deltaY = e.clientY - this.dragStartY
      const deltaCmX = deltaX / this.cmToPx
      const deltaCmY = deltaY / this.cmToPx

      if (this.draggingMargin === 'left') {
        const newValue = this.dragStartValue + deltaCmX
        const maxLeft = this.pageWidthCm - this.internalRightMargin - 1
        this.internalLeftMargin = Math.max(0, Math.min(maxLeft, newValue))

        const margins: any = {
          left: this.internalLeftMargin,
          right: this.internalRightMargin,
          top: this.internalTopMargin,
          bottom: this.internalBottomMargin
        }
        this.$emit('margin-change', margins)
      } else if (this.draggingMargin === 'right') {
        const newValue = this.dragStartValue - deltaCmX
        const maxRight = this.pageWidthCm - this.internalLeftMargin - 1
        this.internalRightMargin = Math.max(0, Math.min(maxRight, newValue))

        const margins: any = {
          left: this.internalLeftMargin,
          right: this.internalRightMargin,
          top: this.internalTopMargin,
          bottom: this.internalBottomMargin
        }
        this.$emit('margin-change', margins)
      } else if (this.draggingMargin === 'top') {
        const newValue = this.dragStartValue + deltaCmY
        const maxTop = this.pageHeightCm - this.internalBottomMargin - 1
        this.internalTopMargin = Math.max(0, Math.min(maxTop, newValue))

        const margins: any = {
          left: this.internalLeftMargin,
          right: this.internalRightMargin,
          top: this.internalTopMargin,
          bottom: this.internalBottomMargin
        }
        this.$emit('margin-change', margins)
      } else if (this.draggingMargin === 'bottom') {
        const newValue = this.dragStartValue - deltaCmY
        const maxBottom = this.pageHeightCm - this.internalTopMargin - 1
        this.internalBottomMargin = Math.max(0, Math.min(maxBottom, newValue))

        const margins: any = {
          left: this.internalLeftMargin,
          right: this.internalRightMargin,
          top: this.internalTopMargin,
          bottom: this.internalBottomMargin
        }
        this.$emit('margin-change', margins)
      }
    },
    stopDragMargin() {
      this.draggingMargin = null
      document.removeEventListener('mousemove', this.dragMargin)
      document.removeEventListener('mouseup', this.stopDragMargin)
    },
    handleScroll() {
      const scrollContainer = this.$refs.scrollContainer as HTMLElement
      if (scrollContainer) {
        this.scrollLeft = scrollContainer.scrollLeft
      }
    },
    updateRulerPosition() {
      this.$forceUpdate()
    },
    addPage(afterIndex: number) {
      this.pages.splice(afterIndex + 1, 0, {
        header: '',
        footer: '',
        content: '',
        showHeader: false,
        showFooter: false
      })
    },
    removePage(index: number) {
      if (this.pages.length > 1) {
        const confirmed = confirm(`Вы уверены, что хотите удалить страницу ${index + 1}?`)
        if (confirmed) {
          this.pages.splice(index, 1)
          this.$nextTick(() => {
            this.editorRefs = this.editorRefs.filter((_, i) => i !== index)
          })
        }
      }
    }
  },
  mounted() {
    this.$nextTick(() => {
      if (this.editorRefs && this.editorRefs[0]) {
        this.previousContentLength = this.editorRefs[0].innerHTML.length
      }
      this.updateRulerPosition()
    })

    window.addEventListener('resize', this.updateRulerPosition)
  },
  beforeUnmount() {
    document.removeEventListener('mousemove', this.dragMargin)
    document.removeEventListener('mouseup', this.stopDragMargin)
    window.removeEventListener('resize', this.updateRulerPosition)
    if (this.overflowCheckTimeout) {
      clearTimeout(this.overflowCheckTimeout)
    }
    if (this.pasteTimeout) {
      clearTimeout(this.pasteTimeout)
    }
  }
}
</script>

<style scoped>
.editor-wrapper {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background-color: #e8e8e8;
}

.ruler-horizontal {
  height: 35px;
  background-color: #f5f5f5;
  border-bottom: 2px solid #ccc;
  overflow-x: hidden;
  overflow-y: hidden;
  position: relative;
  user-select: none;
  display: flex;
  justify-content: center;
}

.ruler-content {
  position: relative;
  height: 100%;
}

.ruler-page-indicator {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background-color: #ffffff;
  border-left: 2px solid #999;
  border-right: 2px solid #999;
}

.ruler-mark {
  position: absolute;
  width: 2px;
  height: 10px;
  background-color: #666;
  bottom: 0;
}

.ruler-mark-major {
  height: 16px;
  width: 2px;
  background-color: #333;
}

.ruler-number {
  position: absolute;
  top: 0px;
  left: 3px;
  font-size: 13px;
  font-weight: 700;
  color: #000;
  text-shadow: 0 0 3px #fff, 0 0 5px #fff;
  line-height: 1;
}

.ruler-marker {
  position: absolute;
  width: 0;
  height: 0;
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-top: 12px solid #2196F3;
  bottom: 0;
  transform: translateX(-8px);
  cursor: ew-resize;
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.3));
  transition: transform 0.1s;
}

.ruler-marker:hover {
  transform: translateX(-8px) scale(1.1);
}

.ruler-marker-left {
  border-top-color: #4CAF50;
}

.ruler-marker-right {
  border-top-color: #F44336;
}

.editor-container {
  flex: 1;
  display: flex;
  overflow: hidden;
  position: relative;
}

.editor-scroll {
  flex: 1;
  overflow: auto;
  padding: 40px;
  background-color: #e8e8e8;
  user-select: text;
  -webkit-user-select: text;
  -moz-user-select: text;
  -ms-user-select: text;
}

.editor-pages-wrapper {
  outline: none;
  min-height: 100%;
}

.editor-pages-wrapper:focus {
  outline: none;
}

.editor-page-wrapper {
  display: flex;
  margin: 35px auto 30px auto;
  width: fit-content;
}

.editor-page {
  min-width: 800px;
  width: 800px;
  min-height: 1000px;
  height: 1000px;
  background-color: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  position: relative;
  display: flex;
  flex-direction: column;
}

.editor-page.landscape {
  min-width: 1000px;
  width: 1000px;
  min-height: 800px;
  height: 800px;
}

.ruler-vertical {
  width: 35px;
  background-color: #f5f5f5;
  border-right: 2px solid #ccc;
  position: relative;
  user-select: none;
  flex-shrink: 0;
  overflow: hidden;
}

.ruler-vertical-content {
  position: relative;
  width: 100%;
  height: 100%;
}

.ruler-page-indicator-vertical {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  background-color: #ffffff;
  border-top: 2px solid #999;
  border-bottom: 2px solid #999;
}

.ruler-mark-vertical {
  position: absolute;
  height: 2px;
  width: 10px;
  background-color: #666;
  left: 0;
}

.ruler-mark-major-vertical {
  width: 16px;
  height: 2px;
  background-color: #333;
}

.ruler-number-vertical {
  position: absolute;
  left: 18px;
  top: -8px;
  font-size: 11px;
  font-weight: 600;
  color: #333;
  text-shadow: 0 0 2px #fff;
  writing-mode: horizontal-tb;
}

.ruler-marker-vertical {
  position: absolute;
  height: 0;
  width: 0;
  border-top: 8px solid transparent;
  border-bottom: 8px solid transparent;
  border-left: 12px solid #2196F3;
  left: 0;
  transform: translateY(-8px);
  cursor: ns-resize;
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.3));
  transition: transform 0.1s;
}

.ruler-marker-vertical:hover {
  transform: translateY(-8px) scale(1.1);
}

.ruler-marker-top {
  border-left-color: #4CAF50;
}

.ruler-marker-bottom {
  border-left-color: #F44336;
}

.page-header-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 4px 8px;
  background-color: #f9f9f9;
  border-bottom: 1px solid #e0e0e0;
  user-select: none;
  position: absolute;
  top: -33px;
  left: 0;
  right: 0;
  height: 32px;
  z-index: 100;
}

.page-label {
  font-size: 12px;
  color: #666;
  font-weight: 500;
}

.page-action-buttons {
  display: flex;
  gap: 4px;
}

.page-action-btn {
  width: 24px;
  height: 24px;
  border: 1px solid #ccc;
  background-color: #fff;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  color: #333;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
  padding: 0;
}

.page-action-btn:hover {
  background-color: #e8f5e9;
  border-color: #4CAF50;
  color: #4CAF50;
}

.page-action-btn.remove:hover {
  background-color: #ffebee;
  border-color: #f44336;
  color: #f44336;
}

.page-controls {
  display: flex;
  gap: 8px;
  padding: 4px 8px;
  background-color: #f9f9f9;
  border-bottom: 1px solid #e0e0e0;
  user-select: none;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  z-index: 99;
}

.page-controls.bottom-controls {
  border-bottom: none;
  border-top: 1px solid #e0e0e0;
  top: auto;
  bottom: 0;
}

.control-btn {
  padding: 4px 12px;
  font-size: 11px;
  border: 1px solid #d0d0d0;
  background-color: #fff;
  border-radius: 3px;
  cursor: pointer;
  transition: all 0.2s;
  color: #333;
}

.control-btn:hover {
  background-color: #f0f0f0;
  border-color: #999;
}

.control-btn.remove-btn {
  background-color: #ffe0e0;
  border-color: #ffb0b0;
  color: #c00;
}

.control-btn.remove-btn:hover {
  background-color: #ffc0c0;
}

.page-header,
.page-footer {
  min-height: 30px;
  padding: 8px;
  border-bottom: 1px dashed #ccc;
  font-size: 12px;
  color: #666;
  outline: none;
  user-select: text;
}

.page-footer {
  border-bottom: none;
  border-top: 1px dashed #ccc;
}

.page-header:empty:before,
.page-footer:empty:before {
  content: attr(data-placeholder);
  color: #999;
  pointer-events: none;
}

.editor-content {
  flex: 1;
  width: 100%;
  outline: none;
  font-family: Arial, sans-serif;
  font-size: 16px;
  line-height: 1.5;
  color: #333;
  overflow: hidden;
  word-wrap: break-word;
  user-select: text;
}

.editor-content p,
.editor-content div {
  margin: 0;
  padding: 0;
}

.editor-content br {
  content: "";
  display: block;
  margin: 0;
}

.editor-content:focus {
  outline: none;
}

.editor-content img {
  max-width: 100%;
  cursor: move;
  border: 2px solid transparent;
  position: relative;
  display: inline-block;
  z-index: 10;
}

.editor-content img:hover {
  border-color: #2196F3;
  box-shadow: 0 0 8px rgba(33, 150, 243, 0.3);
}

.editor-content img.selected {
  border-color: #2196F3;
}

.editor-content table {
  cursor: pointer;
}

.editor-content table:hover {
  outline: 2px solid #e0e0e0;
}

.editor-content ul,
.editor-content ol {
  padding-left: 40px;
  margin: 8px 0;
}

.editor-content li {
  margin: 4px 0;
}

@media print {
  .editor-wrapper {
    background-color: white;
  }

  .ruler-horizontal,
  .ruler-vertical {
    display: none !important;
  }

  .editor-container {
    overflow: visible !important;
  }

  .editor-page-wrapper {
    display: block !important;
  }

  .editor-scroll {
    overflow: visible !important;
    padding: 0 !important;
    background-color: white !important;
  }

  .editor-page {
    box-shadow: none !important;
    margin: 0 !important;
    page-break-after: always;
    page-break-inside: avoid;
    transform: none !important;
  }

  .editor-page:last-child {
    page-break-after: auto;
  }

  .page-label,
  .page-controls,
  .page-header-row,
  .page-action-buttons {
    display: none !important;
  }

  .page-header,
  .page-footer {
    border: none !important;
  }

  .editor-content {
    overflow: visible !important;
  }

  @page {
    margin: 0;
    size: A4 portrait;
  }

  @page :first {
    margin-top: 0;
  }

  .editor-page.landscape {
    width: 297mm;
    height: 210mm;
  }

  @page landscape {
    size: A4 landscape;
  }
}
</style>
