<template>
  <div class="image-annotator">
    <button class="help-button" @click="showHelp = true">
      <span class="help-icon">?</span>
    </button>
    
    <div class="help-modal" v-if="showHelp" @click.self="showHelp = false">
      <div class="help-content">
        <h2>标注工具使用说明</h2>
        <div class="help-sections">
          <section>
            <h3>基本操作</h3>
            <ul>
              <li>点击"添加图片"上传一张或多张图片</li>
              <li>使用左右方向键或点击缩略图切换图片</li>
              <li>使用鼠标滚轮或缩放按钮调整图片大小</li>
            </ul>
          </section>
          
          <section>
            <h3>标注步骤</h3>
            <ol>
              <li>在左侧添加需要标注的类别</li>
              <li>从下拉菜单选择要标注的类别</li>
              <li>点击"开始绘制"按钮</li>
              <li>在图片上拖动鼠标绘制标注框</li>
              <li>标注框会显示在下方列表中</li>
            </ol>
          </section>
          
          <section>
            <h3>快捷键</h3>
            <ul>
              <li>← ：切换到上一张图片</li>
              <li>→ ：切换到下一张图片</li>
            </ul>
          </section>
          
          <section>
            <h3>数据导出</h3>
            <ul>
              <li>点击"导出标注"按钮</li>
              <li>将生成包含以下文件的压缩包：</li>
              <li>- classes.txt：类别名称列表</li>
              <li>- *.txt：每张图片对应的标注文件</li>
              <li>- *.jpg：原始图片文件</li>
            </ul>
          </section>
        </div>
        <button class="close-help" @click="showHelp = false">关闭</button>
      </div>
    </div>

    <div class="keyboard-shortcuts-hint">
      <span>快捷键：</span>
      <span>← 上一张图片</span>
      <span>→ 下一张图片</span>
    </div>
    <div class="sidebar">
      <div class="sidebar-section">
        <h3>标注类别管理</h3>
        <div class="class-input">
          <input 
            v-model="newClassName" 
            placeholder="输入新类别名称"
            @keyup.enter="addClass"
          />
          <button @click="addClass" class="add-btn">添加</button>
        </div>
        <div class="class-list">
          <div v-for="(id, name) in classIds" :key="name" class="class-item">
            <div class="class-info">
              <span class="color-dot" :style="{ backgroundColor: classColors[name] }"></span>
              <span>{{ name }}</span>
            </div>
            <button @click="deleteClass(name)" class="delete-btn">×</button>
          </div>
        </div>
      </div>
      
      <div class="sidebar-section">
        <select 
          v-model="currentClass"
          :class="{ 'required': !currentClass }"
        >
          <option value="">选择标注类别</option>
          <option v-for="(id, name) in classIds" :key="name" :value="name">
            {{ name }}
          </option>
        </select>
      </div>
      
      <div class="sidebar-section">
        <div class="tool-buttons">
          <button 
            @click="toggleDrawing" 
            :disabled="!currentClass"
            :class="{ active: isDrawing }"
          >
            {{ isDrawing ? '完成绘制' : '开始绘制' }}
          </button>
          <div class="zoom-controls">
            <button @click="zoomIn" title="放大">+</button>
            <button @click="zoomOut" title="缩小">-</button>
            <button @click="resetZoom" title="重置">重置</button>
            <span>{{ (scale * 100).toFixed(0) }}%</span>
          </div>
        </div>
      </div>
    </div>

    <div class="main-content">
      <div class="top-section">
        <div class="upload-controls">
          <input 
            type="file" 
            @change="handleImageUpload" 
            accept="image/*"
            ref="fileInput"
            multiple
            class="file-input"
          />
          <button @click="$refs.fileInput.click()" class="upload-btn">
            添加图片
          </button>
          <button 
            @click="showExportDialog = true" 
            class="export-btn"
            :disabled="!hasAnnotations"
          >
            导出数据集
          </button>
        </div>
        
        <div class="image-list">
          <div class="image-grid">
            <div 
              v-for="(img, index) in images" 
              :key="index"
              class="image-item"
              :class="{ active: currentImageIndex === index }"
              @click="switchImage(index)"
            >
              <img :src="img.url" :alt="img.name" />
              <div class="image-info">
                <span>{{ img.name }}</span>
                <button @click.stop="removeImage(index)" class="remove-btn">×</button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="annotation-area">
        <div class="canvas-container" @wheel="handleWheel">
          <div v-if="!currentImage" class="placeholder">
            请上传图片开始标注
          </div>
          <canvas 
            ref="canvas" 
            @mousedown="startDrawing"
            @click="handleCanvasClick"
          ></canvas>
        </div>

        <div class="annotations-list">
          <div 
            v-for="(box, index) in currentBoxes" 
            :key="index" 
            class="annotation-item"
            :class="{ selected: selectedBoxIndex === index }"
            @click="selectBox(index)"
          >
            <span>{{ box.class }}</span>
            <button @click="deleteBox(index)" class="delete-btn">×</button>
          </div>
        </div>
      </div>
    </div>

    <!-- 导出设置对话框 -->
    <div class="export-modal" v-if="showExportDialog" @click.self="showExportDialog = false">
      <div class="export-content">
        <h2>导出设置</h2>
        
        <div class="export-section">
          <h3>数据格式</h3>
          <div class="format-options">
            <label>
              <input type="radio" v-model="exportFormat" value="yolo">
              YOLO
            </label>
            <label>
              <input type="radio" v-model="exportFormat" value="coco">
              COCO
            </label>
            <label>
              <input type="radio" v-model="exportFormat" value="voc">
              VOC
            </label>
          </div>
        </div>
        
        <div class="export-section">
          <h3>导出内容</h3>
          <div class="content-options">
            <label>
              <input type="checkbox" v-model="exportOptions.includeImages">
              包含图片文件
            </label>
            <label>
              <input type="checkbox" v-model="exportOptions.includeClasses">
              包含类别文件
            </label>
          </div>
        </div>
        
        <div class="export-section">
          <h3>图片设置</h3>
          <div class="image-options" v-if="exportOptions.includeImages">
            <label>
              <input type="checkbox" v-model="exportOptions.compressImages">
              压缩图片
            </label>
            <div class="quality-slider" v-if="exportOptions.compressImages">
              <span>质量：{{ exportOptions.imageQuality }}%</span>
              <input 
                type="range" 
                v-model.number="exportOptions.imageQuality" 
                min="10" 
                max="100" 
                step="10"
              >
            </div>
          </div>
        </div>
        
        <div class="export-section">
          <h3>数据集划分</h3>
          <div class="split-options">
            <label>
              <input type="checkbox" v-model="exportOptions.splitDataset">
              划分训练/验证集
            </label>
            <div class="split-ratio" v-if="exportOptions.splitDataset">
              <span>训练集比例：{{ exportOptions.trainRatio }}%</span>
              <input 
                type="range" 
                v-model.number="exportOptions.trainRatio" 
                min="50" 
                max="90" 
                step="10"
              >
            </div>
          </div>
        </div>
        
        <div class="button-group">
          <button class="cancel-btn" @click="showExportDialog = false">取消</button>
          <button class="confirm-btn" @click="handleExport">确认导出</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import JSZip from 'jszip'

export default {
  name: 'ImageAnnotator',
  data() {
    return {
      isDrawing: false,
      startPos: null,
      isMouseDown: false,
      currentBox: null,
      currentClass: '',
      newClassName: '',
      classIds: {
        'person': 0,
        'car': 1,
        'dog': 2,
        'cat': 3
      },
      yoloAnnotations: null,
      scale: 1,
      minScale: 0.1,
      maxScale: 5,
      originalWidth: 0,
      originalHeight: 0,
      // 预定义一组醒目的颜色
      predefinedColors: [
        '#FF0000', '#00FF00', '#0000FF', '#FF00FF', // 基础色
        '#FFA500', '#800080', '#008080', '#FFD700', // 橙色、紫色、青色、金色
        '#FF1493', '#32CD32', '#4169E1', '#FF4500', // 粉色、绿色、蓝色、橙红
        '#8A2BE2', '#20B2AA', '#FF69B4', '#00CED1', // 紫蓝、浅绿、粉红、青蓝
        '#FF6347', '#7B68EE', '#3CB371', '#B22222'  // 番茄红、淡紫、绿色、深红
      ],
      classColors: {
        'person': '#FF0000',  // 红色
        'car': '#00FF00',     // 绿色
        'dog': '#0000FF',     // 蓝色
        'cat': '#FF00FF'      // 紫色
      },
      images: [], // 存储所有图片信息
      currentImageIndex: -1, // 当前显示的图片索引
      annotations: {}, // 存储每张图片的标注信息 {imageIndex: boxes}
      selectedBoxIndex: -1, // 当前选中的标注框索引
      history: [], // 操作历史
      historyIndex: -1, // 当前历史位置
      isMac: /macintosh|mac os x/i.test(navigator.userAgent),
      showHelp: false,
      showExportDialog: false,
      exportFormat: 'yolo',
      exportOptions: {
        includeImages: true,
        includeClasses: true,
        compressImages: false,
        imageQuality: 80,
        splitDataset: false,
        trainRatio: 80
      }
    }
  },
  computed: {
    currentImage() {
      return this.currentImageIndex >= 0 ? this.images[this.currentImageIndex] : null
    },
    currentBoxes() {
      return this.currentImageIndex >= 0 ? 
        (this.annotations[this.currentImageIndex.toString()] || []) : []
    },
    hasAnnotations() {
      return Object.values(this.annotations).some(boxes => boxes.length > 0)
    }
  },
  mounted() {
    this.initCanvas()
    // 添加键盘事件监听
    window.addEventListener('keydown', this.handleKeyDown)
  },
  beforeUnmount() {
    // 移除键盘事件监听
    window.removeEventListener('keydown', this.handleKeyDown)
  },
  methods: {
    async handleImageUpload(event) {
      const files = Array.from(event.target.files)
      for (const file of files) {
        const imageInfo = await this.loadImage(file)
        this.images.push(imageInfo)
        if (this.currentImageIndex === -1) {
          this.switchImage(0)
        }
      }
    },
    
    loadImage(file) {
      return new Promise((resolve) => {
        const reader = new FileReader()
        reader.onload = (e) => {
          const img = new Image()
          img.onload = () => {
            resolve({
              file,
              name: file.name,
              url: e.target.result,
              width: img.width,
              height: img.height,
              image: img
            })
          }
          img.src = e.target.result
        }
        reader.readAsDataURL(file)
      })
    },
    
    switchImage(index) {
      if (index === this.currentImageIndex) return
      this.isDrawing = false
      this.startPos = null
      this.isMouseDown = false
      
      this.currentImageIndex = index
      this.originalWidth = this.currentImage.width
      this.originalHeight = this.currentImage.height
      
      // 设置画布初始大小为图片原始大小
      this.canvas.width = this.originalWidth
      this.canvas.height = this.originalHeight
      
      // 重置缩放
      this.resetZoom()
    },
    
    removeImage(index) {
      if (confirm('确定要删除这张图片吗？')) {
        this.images.splice(index, 1)
        // 删除对应的标注数据
        delete this.annotations[index.toString()]
        
        // 重新排序annotations
        const newAnnotations = {}
        Object.entries(this.annotations).forEach(([key, value]) => {
          const keyNum = parseInt(key)
          if (keyNum > index) {
            newAnnotations[(keyNum - 1).toString()] = value
          } else if (keyNum < index) {
            newAnnotations[key] = value
          }
        })
        this.annotations = newAnnotations
        
        // 更新当前图片索引
        if (this.images.length === 0) {
          this.currentImageIndex = -1
        } else if (this.currentImageIndex >= this.images.length) {
          this.currentImageIndex = this.images.length - 1
        }
        this.redrawCanvas()
      }
    },
    
    initCanvas() {
      this.canvas = this.$refs.canvas
      this.ctx = this.canvas.getContext('2d')
      this.canvas.width = 800
      this.canvas.height = 600
    },
    
    toggleDrawing() {
      this.isDrawing = !this.isDrawing
      if (!this.isDrawing) {
        this.isMouseDown = false
        this.startPos = null
        this.redrawCanvas()
      }
    },
    
    startDrawing(event) {
      if (!this.currentClass) {
        alert('请先选择标注类别！')
        return
      }
      // 确保是在画布上按下鼠标
      const target = event.target
      if (!this.isDrawing || target !== this.canvas) return
      
      this.isMouseDown = true
      const rect = this.canvas.getBoundingClientRect()
      
      // 计算鼠标在画布上的实际位置
      this.startPos = {
        x: (event.clientX - rect.left) * (this.originalWidth / rect.width),
        y: (event.clientY - rect.top) * (this.originalHeight / rect.height)
      }
      
      document.addEventListener('mousemove', this.draw)
      document.addEventListener('mouseup', this.endDrawing)
    },
    
    draw(event) {
      if (!this.isMouseDown || !this.startPos || !this.isDrawing) return
      
      const rect = this.canvas.getBoundingClientRect()
      
      const currentPos = {
        x: (event.clientX - rect.left) * (this.originalWidth / rect.width),
        y: (event.clientY - rect.top) * (this.originalHeight / rect.height)
      }
      
      this.redrawCanvas()
      
      // 绘制当前正在绘制的框
      this.ctx.strokeStyle = this.getClassColor(this.currentClass)
      this.ctx.lineWidth = 2
      this.ctx.strokeRect(
        this.startPos.x * this.scale,
        this.startPos.y * this.scale,
        (currentPos.x - this.startPos.x) * this.scale,
        (currentPos.y - this.startPos.y) * this.scale
      )
    },
    
    endDrawing(event) {
      if (!this.isMouseDown || !this.startPos || !this.isDrawing) return
      this.isMouseDown = false
      
      const rect = this.canvas.getBoundingClientRect()
      
      const endPos = {
        x: (event.clientX - rect.left) * (this.originalWidth / rect.width),
        y: (event.clientY - rect.top) * (this.originalHeight / rect.height)
      }
      
      // 确保宽度和高度都大于0
      const width = Math.abs(endPos.x - this.startPos.x)
      const height = Math.abs(endPos.y - this.startPos.y)
      
      if (width > 0 && height > 0) {
        const newBox = {
          x: Math.min(this.startPos.x, endPos.x),
          y: Math.min(this.startPos.y, endPos.y),
          width: width,
          height: height,
          class: this.currentClass,
          confidence: 1.0
        }
        
        // 确保当前图片的标注数组已初始化
        if (!this.annotations[this.currentImageIndex.toString()]) {
          this.annotations[this.currentImageIndex.toString()] = []
        }
        
        // 添加新的标注
        this.annotations[this.currentImageIndex.toString()].push(newBox)
      }
      
      this.startPos = null
      // 移除事件监听
      document.removeEventListener('mousemove', this.draw)
      document.removeEventListener('mouseup', this.endDrawing)
    },
    
    drawBox(box, index) {
      const color = this.getClassColor(box.class)
      this.ctx.strokeStyle = color
      this.ctx.lineWidth = index === this.selectedBoxIndex ? 3 : 2
      
      const scaledBox = {
        x: box.x * this.scale,
        y: box.y * this.scale,
        width: box.width * this.scale,
        height: box.height * this.scale
      }
      
      this.ctx.strokeRect(
        scaledBox.x,
        scaledBox.y,
        scaledBox.width,
        scaledBox.height
      )
      
      if (index === this.selectedBoxIndex) {
        this.ctx.fillStyle = `${color}33`  // 添加20%透明度
        this.ctx.fillRect(
          scaledBox.x,
          scaledBox.y,
          scaledBox.width,
          scaledBox.height
        )
      }
      
      // 绘制标签
      this.ctx.fillStyle = color
      this.ctx.font = '14px Arial'
      this.ctx.fillText(
        box.class,
        scaledBox.x,
        scaledBox.y - 5
      )
    },
    
    deleteBox(index) {
      if (index === -1) return
      
      // 添加到历史记录
      this.addHistory('delete')
      
      const currentAnnotations = this.annotations[this.currentImageIndex.toString()]
      if (currentAnnotations) {
        currentAnnotations.splice(index, 1)
        if (currentAnnotations.length === 0) {
          delete this.annotations[this.currentImageIndex.toString()]
        }
      }
      this.selectedBoxIndex = -1
      this.redrawCanvas()
    },
    
    redrawCanvas() {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)
      if (this.currentImage) {
        this.ctx.drawImage(
          this.currentImage.image,
          0,
          0,
          this.canvas.width,
          this.canvas.height
        )
      }
      this.currentBoxes.forEach((box, index) => this.drawBox(box, index))
    },
    
    convertToYoloFormat(box) {
      // 将坐标转换为YOLO格式（归一化的中心点坐标和宽高）
      // 使用原始图片尺寸进行归一化
      const centerX = (box.x + box.width / 2) / (this.originalWidth * this.scale)
      const centerY = (box.y + box.height / 2) / (this.originalHeight * this.scale)
      const width = box.width / (this.originalWidth * this.scale)
      const height = box.height / (this.originalHeight * this.scale)
      
      // 确保值在0-1之间
      const clamp = (num) => Math.min(Math.max(num, 0), 1)
      return {
        classId: this.classIds[box.class],
        centerX: clamp(centerX),
        centerY: clamp(centerY),
        width: clamp(width),
        height: clamp(height)
      }
    },
    
    async handleExport() {
      this.showExportDialog = false
      const zip = new JSZip()
      
      // 添加类别文件
      if (this.exportOptions.includeClasses) {
        if (this.exportFormat === 'yolo') {
          const sortedClasses = Object.entries(this.classIds)
            .sort((a, b) => a[1] - b[1])
            .map(([name]) => name)
          zip.file('classes.txt', sortedClasses.join('\n'))
        }
      }
      
      // 准备数据集
      let images = [...this.images]
      if (this.exportOptions.splitDataset) {
        images = this.shuffleArray(images)
        const trainCount = Math.floor(images.length * this.exportOptions.trainRatio / 100)
        
        const trainDir = zip.folder('train')
        const valDir = zip.folder('val')
        
        await this.processDataset(images.slice(0, trainCount), trainDir)
        await this.processDataset(images.slice(trainCount), valDir)
      } else {
        await this.processDataset(images, zip)
      }
      
      try {
        const content = await zip.generateAsync({ type: 'blob' })
        const url = URL.createObjectURL(content)
        const link = document.createElement('a')
        link.href = url
        link.download = 'dataset.zip'
        document.body.appendChild(link)
        link.click()
        document.body.removeChild(link)
        URL.revokeObjectURL(url)
      } catch (error) {
        console.error('Export failed:', error)
        alert('导出失败，请重试')
      }
    },
    
    async processDataset(images, zipFolder) {
      for (let i = 0; i < images.length; i++) {
        const img = images[i]
        const annotations = this.annotations[i.toString()] || []
        
        // 添加图片
        if (this.exportOptions.includeImages) {
          if (this.exportOptions.compressImages) {
            const compressedImage = await this.compressImage(
              img.file, 
              this.exportOptions.imageQuality
            )
            zipFolder.file(img.name, compressedImage)
          } else {
            zipFolder.file(img.name, img.file)
          }
        }
        
        // 根据不同格式生成标注文件
        if (this.exportFormat === 'yolo') {
          const txtFileName = img.name.replace(/\.[^/.]+$/, '.txt')
          zipFolder.file(txtFileName, this.generateYOLOAnnotations(annotations, img))
        } else if (this.exportFormat === 'voc') {
          const xmlFileName = img.name.replace(/\.[^/.]+$/, '.xml')
          zipFolder.file(xmlFileName, this.generateVOCAnnotation(img, annotations))
        }
      }
      
      // COCO格式需要生成单个json文件
      if (this.exportFormat === 'coco') {
        zipFolder.file('annotations.json', this.generateCOCOAnnotations())
      }
    },
    
    shuffleArray(array) {
      const newArray = [...array]
      for (let i = newArray.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [newArray[i], newArray[j]] = [newArray[j], newArray[i]]
      }
      return newArray
    },
    
    async compressImage(file, quality) {
      return new Promise((resolve) => {
        const img = new Image()
        img.onload = () => {
          const canvas = document.createElement('canvas')
          canvas.width = img.width
          canvas.height = img.height
          const ctx = canvas.getContext('2d')
          ctx.drawImage(img, 0, 0)
          canvas.toBlob(
            (blob) => resolve(blob),
            'image/jpeg',
            quality / 100
          )
        }
        img.src = URL.createObjectURL(file)
      })
    },
    
    addClass() {
      if (!this.newClassName.trim()) return
      
      if (this.classIds.hasOwnProperty(this.newClassName)) {
        alert('该类别已存在！')
        return
      }
      
      const newId = Object.values(this.classIds).length > 0 
        ? Math.max(...Object.values(this.classIds)) + 1 
        : 0
      
      this.classIds = {
        ...this.classIds,
        [this.newClassName]: newId
      }
      this.classColors[this.newClassName] = this.generateUniqueColor()
      this.newClassName = ''
    },
    
    deleteClass(className) {
      // 检查所有图片的标注
      if (Object.values(this.annotations).some(boxes => 
        boxes.some(box => box.class === className)
      )) {
        alert('该类别已被使用，无法删除！')
        return
      }
      
      if (this.currentClass === className) {
        this.currentClass = ''
      }
      
      const { [className]: removed, ...rest } = this.classIds
      this.classIds = rest
      delete this.classColors[className]
    },
    
    updateCanvasSize() {
      if (!this.currentImage) return
      
      this.canvas.width = this.originalWidth * this.scale
      this.canvas.height = this.originalHeight * this.scale
      
      this.redrawCanvas()
    },
    
    zoomIn() {
      this.scale = Math.min(this.maxScale, this.scale * 1.2)
      this.updateCanvasSize()
    },
    
    zoomOut() {
      this.scale = Math.max(this.minScale, this.scale / 1.2)
      this.updateCanvasSize()
    },
    
    resetZoom() {
      const containerWidth = 800
      const containerHeight = 600
      const scaleX = containerWidth / this.originalWidth
      const scaleY = containerHeight / this.originalHeight
      this.scale = Math.min(1, scaleX, scaleY)
      this.updateCanvasSize()
    },
    
    handleWheel(event) {
      if (!this.currentImage) return
      event.preventDefault()
      if (event.deltaY < 0) {
        this.zoomIn()
      } else {
        this.zoomOut()
      }
    },
    
    getClassColor(className) {
      // 如果没有预设颜色，则生成一个随机颜色
      if (!this.classColors[className]) {
        this.classColors[className] = this.generateUniqueColor()
      }
      return this.classColors[className]
    },
    
    generateUniqueColor() {
      // 首先尝试使用预定义颜色
      const usedColors = Object.values(this.classColors)
      const availableColor = this.predefinedColors.find(color => !usedColors.includes(color))
      
      if (availableColor) {
        return availableColor
      }
      
      // 如果预定义颜色都用完了，生成随机颜色
      let newColor
      do {
        // 生成明亮的颜色
        const hue = Math.random() * 360
        const saturation = 70 + Math.random() * 30 // 70-100%
        const lightness = 45 + Math.random() * 10  // 45-55%
        newColor = `hsl(${hue}, ${saturation}%, ${lightness}%)`
      } while (usedColors.includes(newColor))
      
      return newColor
    },
    
    handleKeyDown(event) {
      // 如果正在输入，不处理快捷键
      if (event.target.tagName === 'INPUT') return
      
      // 左右方向键切换图片
      if (event.key === 'ArrowLeft') {
        event.preventDefault()
        if (this.currentImageIndex > 0) {
          this.switchImage(this.currentImageIndex - 1)
        }
      } else if (event.key === 'ArrowRight') {
        event.preventDefault()
        if (this.currentImageIndex < this.images.length - 1) {
          this.switchImage(this.currentImageIndex + 1)
        }
      }
    },
    
    generateYOLOAnnotations(annotations, image) {
      return annotations.map(box => {
        const classId = this.classIds[box.class]
        // 计算归一化的中心点坐标和宽高
        const centerX = (box.x + box.width / 2) / this.originalWidth
        const centerY = (box.y + box.height / 2) / this.originalHeight
        const width = box.width / this.originalWidth
        const height = box.height / this.originalHeight
        return `${classId} ${centerX.toFixed(6)} ${centerY.toFixed(6)} ${width.toFixed(6)} ${height.toFixed(6)}`
      }).join('\n')
    },
    
    generateCOCOAnnotations() {
      const coco = {
        info: {
          year: new Date().getFullYear(),
          version: '1.0',
          description: 'Exported from YOLO Annotator',
          date_created: new Date().toISOString()
        },
        images: [],
        annotations: [],
        categories: []
      }
      
      // 添加类别信息
      Object.entries(this.classIds).forEach(([name, id]) => {
        coco.categories.push({
          id: id + 1, // COCO的类别ID从1开始
          name: name,
          supercategory: 'object'
        })
      })
      
      let annotationId = 1
      
      // 添加图片和标注信息
      this.images.forEach((img, imageId) => {
        coco.images.push({
          id: imageId + 1,
          file_name: img.name,
          width: this.originalWidth,
          height: this.originalHeight
        })
        
        const annotations = this.annotations[imageId.toString()] || []
        annotations.forEach(box => {
          coco.annotations.push({
            id: annotationId++,
            image_id: imageId + 1,
            category_id: this.classIds[box.class] + 1,
            bbox: [
              box.x,
              box.y,
              box.width,
              box.height
            ],
            area: box.width * box.height,
            segmentation: [],
            iscrowd: 0
          })
        })
      })
      
      return JSON.stringify(coco, null, 2)
    },
    
    generateVOCAnnotation(imageInfo, annotations) {
      const xml = []
      xml.push('<?xml version="1.0" encoding="UTF-8"?>')
      xml.push('<annotation>')
      xml.push('  <folder>images</folder>')
      xml.push(`  <filename>${imageInfo.name}</filename>`)
      xml.push('  <source>')
      xml.push('    <database>Unknown</database>')
      xml.push('  </source>')
      xml.push('  <size>')
      xml.push(`    <width>${this.originalWidth}</width>`)
      xml.push(`    <height>${this.originalHeight}</height>`)
      xml.push('    <depth>3</depth>')
      xml.push('  </size>')
      xml.push('  <segmented>0</segmented>')
      
      annotations.forEach(box => {
        xml.push('  <object>')
        xml.push(`    <name>${box.class}</name>`)
        xml.push('    <pose>Unspecified</pose>')
        xml.push('    <truncated>0</truncated>')
        xml.push('    <difficult>0</difficult>')
        xml.push('    <bndbox>')
        xml.push(`      <xmin>${Math.round(box.x)}</xmin>`)
        xml.push(`      <ymin>${Math.round(box.y)}</ymin>`)
        xml.push(`      <xmax>${Math.round(box.x + box.width)}</xmax>`)
        xml.push(`      <ymax>${Math.round(box.y + box.height)}</ymax>`)
        xml.push('    </bndbox>')
        xml.push('  </object>')
      })
      
      xml.push('</annotation>')
      return xml.join('\n')
    },
  }
}
</script>

<style scoped>
.image-annotator {
  display: grid;
  grid-template-columns: 250px 1fr;
  gap: 1rem;
  padding: 1rem;
  height: 100vh;
  background-color: #f0f2f5;
}

.sidebar {
  background-color: white;
  padding: 1rem;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  max-height: calc(100vh - 2rem);
  overflow-y: auto;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.sidebar-section {
  background-color: white;
  padding: 0.8rem;
  border-radius: 6px;
  border: 1px solid #e8e8e8;
  transition: all 0.3s ease;
}

.sidebar-section:hover {
  box-shadow: 0 2px 12px rgba(0,0,0,0.1);
}

.sidebar-section h3 {
  color: #1a1a1a;
  font-size: 1rem;
  margin-bottom: 1rem;
  font-weight: 600;
}

.class-input input {
  flex: 1;
  padding: 8px;
  border: 2px solid #e8e8e8;
  border-radius: 4px;
  transition: border-color 0.3s ease;
  font-size: 0.9rem;
}

.class-input input:focus {
  outline: none;
  border-color: #4CAF50;
}

.add-btn {
  padding: 8px 16px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
}

.add-btn:hover {
  background-color: #45a049;
  transform: translateY(-1px);
}

.class-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem;
  background-color: white;
  border: 1px solid #e8e8e8;
  border-radius: 4px;
  transition: all 0.3s ease;
}

.class-item:hover {
  background-color: #f8f9fa;
  transform: translateX(2px);
}

.color-dot {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  display: inline-block;
  margin-right: 8px;
  border: 2px solid rgba(255,255,255,0.8);
  box-shadow: 0 0 4px rgba(0,0,0,0.2);
}

.main-content {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  background-color: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.top-section {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  max-height: 180px;
  padding-bottom: 1rem;
  border-bottom: 2px solid #f0f2f5;
}

.image-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 120px));
  gap: 1rem;
  max-height: 200px;
  overflow-y: auto;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 8px;
  border: 1px solid #e8e8e8;
}

.image-item {
  position: relative;
  border: none;
  border-radius: 6px;
  overflow: hidden;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0 2px 6px rgba(0,0,0,0.1);
  width: 120px;
  height: 90px;
}

.image-item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  object-position: center;
}

.image-item.active {
  box-shadow: 0 0 0 2px #4CAF50, 0 2px 8px rgba(0,0,0,0.2);
  transform: translateY(-2px);
}

.image-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

.canvas-container {
  border: 2px solid #e8e8e8;
  position: relative;
  width: 100%;
  height: 600px;
  min-height: 400px;
  overflow: auto;
  border-radius: 8px;
  background-color: #f8f9fa;
}

.annotations-list {
  background-color: white;
  padding: 1rem;
  border-radius: 8px;
  border: 1px solid #e8e8e8;
  max-height: 200px;
  overflow-y: auto;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.annotation-item {
  display: flex;
  justify-content: space-between;
  padding: 0.5rem;
  border-bottom: 1px solid #f0f2f5;
  transition: all 0.3s ease;
}

.annotation-item:hover {
  background-color: #f8f9fa;
  padding-left: 1rem;
}

.zoom-controls button {
  padding: 4px 8px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
}

.zoom-controls button:hover {
  background-color: #45a049;
  transform: translateY(-1px);
}

/* 自定义滚动条样式 */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 4px;
}

::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
  background: #555;
}

.image-list {
  margin-bottom: 1rem;
}

.image-info {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 4px;
  background: rgba(0,0,0,0.7);
  color: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.image-info span {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 8px;
}

.remove-btn {
  background: none;
  border: none;
  color: white;
  cursor: pointer;
  padding: 0 4px;
  font-size: 16px;
  flex-shrink: 0;
}

.remove-btn:hover {
  color: #ff4444;
}

.top-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  background-color: white;
  border-radius: 4px;
  border: 1px solid #ddd;
}

.keyboard-shortcuts-hint {
  position: fixed;
  bottom: 1rem;
  left: 50%;
  transform: translateX(-50%);
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  font-size: 0.9rem;
  display: flex;
  gap: 1rem;
  z-index: 1000;
}

.keyboard-shortcuts-hint span {
  white-space: nowrap;
}

.keyboard-shortcuts-hint span:not(:first-child) {
  padding-left: 1rem;
  border-left: 1px solid rgba(255, 255, 255, 0.3);
}

.annotation-item.selected {
  background-color: #e3f2fd;
  border-left: 3px solid #1976d2;
}

.annotation-item:hover {
  background-color: #f5f5f5;
}

.annotation-item.selected:hover {
  background-color: #e3f2fd;
}

.help-button {
  position: fixed;
  top: 1rem;
  right: 1rem;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background-color: #4CAF50;
  color: white;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  font-weight: bold;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
  z-index: 1000;
  transition: all 0.3s ease;
}

.help-button:hover {
  background-color: #45a049;
  transform: scale(1.1);
}

.help-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1001;
}

.help-content {
  background-color: white;
  border-radius: 8px;
  padding: 2rem;
  max-width: 600px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 4px 12px rgba(0,0,0,0.2);
}

.help-content h2 {
  margin: 0 0 1.5rem;
  color: #1a1a1a;
  font-size: 1.5rem;
  text-align: center;
}

.help-sections {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.help-sections section {
  background-color: #f8f9fa;
  padding: 1rem;
  border-radius: 6px;
  border: 1px solid #e8e8e8;
}

.help-sections h3 {
  color: #2c3e50;
  margin: 0 0 0.8rem;
  font-size: 1.1rem;
}

.help-sections ul,
.help-sections ol {
  margin: 0;
  padding-left: 1.2rem;
}

.help-sections li {
  margin: 0.5rem 0;
  color: #4a4a4a;
  line-height: 1.4;
}

.close-help {
  display: block;
  margin: 1.5rem auto 0;
  padding: 0.5rem 2rem;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.3s ease;
}

.close-help:hover {
  background-color: #45a049;
  transform: translateY(-1px);
}

.export-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1001;
}

.export-content {
  background-color: white;
  border-radius: 8px;
  padding: 2rem;
  width: 500px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 4px 12px rgba(0,0,0,0.2);
}

.export-section {
  margin-bottom: 1.5rem;
}

.export-section h3 {
  color: #2c3e50;
  margin-bottom: 1rem;
  font-size: 1.1rem;
}

.format-options,
.content-options,
.image-options,
.split-options {
  display: flex;
  flex-direction: column;
  gap: 0.8rem;
}

.quality-slider,
.split-ratio {
  margin-left: 1.5rem;
  margin-top: 0.5rem;
}

.button-group {
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
  margin-top: 2rem;
}

.cancel-btn,
.confirm-btn {
  padding: 0.5rem 2rem;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.3s ease;
}

.cancel-btn {
  background-color: #f8f9fa;
  border: 1px solid #ddd;
  color: #666;
}

.confirm-btn {
  background-color: #4CAF50;
  border: none;
  color: white;
}

.cancel-btn:hover {
  background-color: #e9ecef;
}

.confirm-btn:hover {
  background-color: #45a049;
}

.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.disabled input {
  cursor: not-allowed;
}
</style> 