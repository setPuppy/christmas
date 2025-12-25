<template>
  <div class="app">
    <!-- Aurora Background -->
    <div class="aurora-bg"></div>
    
    <!-- Loader -->
    <div id="loader" :class="{ hidden: initialized }">
      <div class="spinner"></div>
      <div class="loader-text">小张努力加载中...</div>
    </div>

    <!-- Title with Enhanced Glow -->
    <!-- <h1 :class="{ 'ui-hidden': uiHidden }">
      <span class="title-main">圣诞快乐</span>
      <span class="title-sparkle">✨</span>
    </h1> -->
    
    <!-- Mode Indicator -->
    <!-- <div class="mode-indicator" :class="{ 'ui-hidden': uiHidden }">
      MODE: {{ mode }}
    </div> -->

    <!-- Photo Counter -->
    <!-- <div class="photo-counter" :class="{ 'ui-hidden': uiHidden }" v-if="photos.length > 0">
      Photo {{ currentPhotoIndex + 1 }} / {{ photos.length }}
    </div> -->

    <!-- Upload Controls -->
    <!-- <div class="upload-wrapper" :class="{ 'ui-hidden': uiHidden }">
      <div class="btn-group">
        <label class="upload-btn" for="fileInput">
          <span class="btn-icon">🎁</span> 
          添加照片
        </label>
        <button class="manage-btn" @click="showPhotoManager = !showPhotoManager" v-if="photos.length > 0">
          📷 管理 ({{ photos.length }})
        </button>
      </div>
      <input type="file" id="fileInput" accept="image/*" multiple @change="handleFileUpload">
      <span class="hint-text">按 H 隐藏UI | N 切换照片 | 1-2-3 切换模式</span>
    </div> -->

    <!-- Focused Photo Overlay - 使用HTML渲染，不受Bloom影响 -->
    <div class="focused-photo-overlay" v-if="mode === 'FOCUS' && focusedPhotoIndex >= 0 && photoDataUrls[focusedPhotoIndex]">
      <div 
        class="focused-photo-container"
        :class="photoAnimating ? currentAnimation : ''"
        :style="photoAnimating ? {
          '--start-x': photoStartPos.x + 'px',
          '--start-y': photoStartPos.y + 'px',
          '--start-scale': photoStartPos.scale
        } : {}"
      >
        <div class="photo-frame">
          <img :src="photoDataUrls[focusedPhotoIndex]" alt="Focused Photo" />
          <div class="photo-shine"></div>
        </div>
        <div class="photo-info">
          <span class="photo-icon">📷</span>
          {{ focusedPhotoIndex + 1 }} / {{ photoDataUrls.length }}
        </div>
      </div>
      <!-- 粒子爆炸效果 -->
      <div class="particle-burst" v-if="photoAnimating">
        <span v-for="i in 24" :key="i" class="burst-particle"></span>
      </div>
      <!-- 光环效果 -->
      <div class="light-rings" v-if="photoAnimating">
        <span class="ring ring-1"></span>
        <span class="ring ring-2"></span>
        <span class="ring ring-3"></span>
      </div>
    </div>

    <!-- Photo Manager Panel -->
    <div class="photo-manager" v-if="showPhotoManager && !uiHidden">
      <div class="manager-header">
        <span>📷 照片管理 ({{ photos.length }}张)</span>
        <button class="close-btn" @click="showPhotoManager = false">✕</button>
      </div>
      <div class="photo-grid">
        <div v-for="(url, index) in photoDataUrls" :key="index" class="photo-item">
          <img :src="url" :alt="'Photo ' + (index + 1)" />
          <div class="photo-overlay">
            <button class="delete-btn" @click="deletePhoto(index + (photos.length - photoDataUrls.length))">🗑️</button>
            <span class="photo-num">{{ index + 1 }}</span>
          </div>
        </div>
      </div>
      <div class="manager-footer" v-if="photoDataUrls.length > 0">
        <button class="clear-all-btn" @click="clearAllPhotos">🗑️ 清空所有照片</button>
      </div>
      <div class="empty-hint" v-if="photoDataUrls.length === 0">
        暂无上传的照片
      </div>
    </div>

    <!-- Gesture Status & Hints - 手机端隐藏 -->
    <!-- <div class="gesture-panel" :class="{ 'ui-hidden': uiHidden }" v-if="!isMobile">
      <div class="gesture-status" :class="{ active: webcamActive }">
        {{ detectedGesture }}
      </div>
      <div class="gesture-hints" v-if="webcamActive">
        ✋ 张开手 → 散开模式<br>
        ✊ 握拳 → 圣诞树模式<br>
        🤏 捏合 → 聚焦照片
      </div>
      <div class="gesture-hints" v-else>
        👆 双击屏幕 → 切换模式<br>
        👆 滑动 → 旋转视角<br>
        ⌨️ 1/2/3 → 切换模式<br>
        ⌨️ N → 下一张照片
      </div>
    </div>
     -->
    <!-- 手机端简洁提示 -->
    <div class="mobile-hint" v-if="isMobile && !uiHidden">
      <span v-if="mode === 'FOCUS'">👆 点击切换照片 | 双击返回</span>
      <span v-else>👆 双击切换模式</span>
    </div>

    <!-- Webcam Container -->
    <div class="webcam-container">
      <video ref="webcamVideo" autoplay playsinline></video>
      <canvas ref="cvCanvas" width="160" height="120"></canvas>
    </div>

    <!-- Three.js Canvas -->
    <canvas ref="threeCanvas"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js'
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js'
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js'
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js'
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js'
import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js'
import { FilesetResolver, HandLandmarker } from '@mediapipe/tasks-vision'

// Refs
const threeCanvas = ref(null)
const webcamVideo = ref(null)
const cvCanvas = ref(null)

// State
const initialized = ref(false)
const uiHidden = ref(false)
const mode = ref('TREE')
const targetMode = ref('TREE')
const photos = ref([])
const photoDataUrls = ref([])  // 存储图片的DataURL用于localStorage
const currentPhotoIndex = ref(0)
const focusedPhotoIndex = ref(-1)
const detectedGesture = ref('等待手势...')
const webcamActive = ref(false)
const showPhotoManager = ref(false)  // 照片管理面板
const photoAnimating = ref(false)    // 照片动画状态
const photoStartPos = ref({ x: 0, y: 0, scale: 0.1 })  // 动画起始位置
const currentAnimation = ref('fly-in')  // 当前动画类型
const animationTypes = ['fly-in', 'spiral-in', 'flip-in', 'bounce-in', 'zoom-blur', 'slide-rotate']
const isMobile = ref(false)  // 是否是手机端

// Three.js objects
let renderer, camera, scene, composer
let mainGroup, starGroup, photoGroup, snowGroup, ribbonGroup
let pointLight, pointLight2
let particles = []
let starParticles = []
let snowParticles = []
let ribbons = []
let topStar = null
let clock
let mouseInteraction = { x: 0, y: 0 }
let handLandmarker = null
let handData = null

// Pinch zoom state for photo
let lastPinchDistance = null
let photoZoomScale = 5

// Gesture detection state
let currentGesture = 'NONE'
let previousGesture = 'NONE'  // 记录上一次的手势
let lastGestureTime = 0
const GESTURE_COOLDOWN = 300 // ms between gesture changes
let hasShownPhotoThisPinch = false  // 标记本次捏合是否已经显示过照片

// 手势稳定性检测 - 需要持续一段时间才确认手势
let pendingGesture = 'NONE'  // 待确认的手势
let pendingGestureStartTime = 0  // 待确认手势的开始时间
const GESTURE_CONFIRM_TIME = 600  // 手势需要持续600ms才确认（接近1秒，但体验更好）
let confirmedGesture = 'FIST'  // 已确认的手势（初始为FIST，对应初始的TREE模式）
let gestureConfirmProgress = 0  // 手势确认进度 (0-1)
let isFirstGestureDetection = true  // 是否是首次检测到手势

// ============================================
// PARTICLE CLASS - Enhanced
// ============================================
class Particle {
  constructor(mesh, type, index) {
    this.mesh = mesh
    this.type = type
    this.index = index
    this.targetPosition = new THREE.Vector3()
    this.rotationSpeed = new THREE.Vector3(
      (Math.random() - 0.5) * 0.08,
      (Math.random() - 0.5) * 0.08,
      (Math.random() - 0.5) * 0.08
    )
    this.treeT = Math.random()
    this.treeAngle = Math.random() * Math.PI * 2
    this.scatterRadius = 15 + Math.random() * 25
    this.scatterTheta = Math.random() * Math.PI * 2
    this.scatterPhi = Math.acos(2 * Math.random() - 1)
    this.originalScale = mesh.scale.clone()
    
    // Softer pulse effects
    this.pulsePhase = Math.random() * Math.PI * 2
    this.pulseSpeed = 1.5 + Math.random() * 4
    this.pulsePhase2 = Math.random() * Math.PI * 2
    this.pulseSpeed2 = 2 + Math.random() * 5
    this.pulsePhase3 = Math.random() * Math.PI * 2
    this.twinkleType = Math.random()
    this.baseEmissive = 0.6 + Math.random() * 0.4
    
    // Color shift
    this.colorPhase = Math.random() * Math.PI * 2
    this.colorSpeed = 0.5 + Math.random() * 1.5
    this.hasColorShift = Math.random() > 0.7
    
    // 烟花效果属性 - 约50%的粒子可以有烟花闪烁效果（实际同时爆发约15%）
    this.isFireworkParticle = Math.random() < 0.5
    this.fireworkPhase = Math.random() * Math.PI * 2
    this.fireworkSpeed = 5 + Math.random() * 12  // 闪烁速度（更快）
    this.fireworkIntensity = 2.0 + Math.random() * 3.0  // 闪烁强度（大幅提高）
    this.fireworkBurstTime = Math.random() * 2  // 初始随机延迟，让爆发更分散
    this.fireworkBurstDuration = 0  // 爆发持续时间
    this.isFireworkBursting = false  // 是否正在爆发
  }

  updateTreePosition(maxRadius = 10, height = 22) {
    const t = this.treeT
    const radius = maxRadius * (1 - t) * (0.7 + Math.random() * 0.3)
    const angle = this.treeAngle + t * 50 * Math.PI // More spiral
    const y = t * height - height * 0.35
    this.targetPosition.set(Math.cos(angle) * radius, y, Math.sin(angle) * radius)
  }

  updateScatterPosition() {
    const r = this.scatterRadius
    this.targetPosition.set(
      r * Math.sin(this.scatterPhi) * Math.cos(this.scatterTheta),
      r * Math.sin(this.scatterPhi) * Math.sin(this.scatterTheta),
      r * Math.cos(this.scatterPhi)
    )
  }

  update(currentMode, time) {
    this.mesh.position.lerp(this.targetPosition, 0.06)

    if (currentMode === 'SCATTER' || currentMode === 'FOCUS') {
      this.mesh.rotation.x += this.rotationSpeed.x
      this.mesh.rotation.y += this.rotationSpeed.y
      this.mesh.rotation.z += this.rotationSpeed.z
    }

    if (this.mesh.material && this.mesh.material.emissiveIntensity !== undefined) {
      let intensity
      
      // 散开模式下的烟花效果
      if ((currentMode === 'SCATTER' || currentMode === 'FOCUS') && this.isFireworkParticle) {
        // 检查是否需要触发新的爆发
        if (!this.isFireworkBursting && time > this.fireworkBurstTime) {
          // 高概率触发爆发（约30%概率每帧触发，确保同时有约15%的粒子在爆发）
          if (Math.random() < 0.08) {
            this.isFireworkBursting = true
            this.fireworkBurstDuration = 0.4 + Math.random() * 0.8  // 爆发持续0.4-1.2秒
            this.fireworkBurstTime = time + this.fireworkBurstDuration
          } else {
            this.fireworkBurstTime = time + 0.05  // 0.05秒后再次检查（更频繁）
          }
        }
        
        // 爆发状态下的烟花效果
        if (this.isFireworkBursting) {
          const burstProgress = 1 - (this.fireworkBurstTime - time) / this.fireworkBurstDuration
          
          // 快速闪烁 + 随机亮度变化，模拟烟花（大幅提高强度）
          const rapidFlash = Math.sin(time * this.fireworkSpeed * 20 + this.fireworkPhase)
          const randomBurst = Math.pow(Math.random(), 0.3) * this.fireworkIntensity  // 更高的随机亮度
          
          // 爆发开始时亮度上升，结束时下降
          const envelopeCurve = Math.sin(burstProgress * Math.PI)
          
          // 大幅提高曝光度：基础亮度 + 闪烁 + 随机爆发
          intensity = this.baseEmissive * (2 + (rapidFlash * 1.5 + randomBurst * 4) * envelopeCurve * 3)
          
          // 爆发时粒子明显变大
          if (this.type !== 'PHOTO') {
            const burstScale = 1 + envelopeCurve * 0.8
            this.mesh.scale.setScalar(this.originalScale.x * burstScale)
          }
          
          // 检查爆发是否结束
          if (time >= this.fireworkBurstTime) {
            this.isFireworkBursting = false
            this.fireworkBurstTime = time + 0.5 + Math.random() * 2  // 0.5-2.5秒后可能再次爆发（更频繁）
          }
        } else {
          // 非爆发状态，正常闪烁但稍微亮一点
          const gentleFlicker = Math.sin(time * this.fireworkSpeed + this.fireworkPhase)
          intensity = this.baseEmissive * (0.8 + gentleFlicker * 0.3)
          
          if (this.type !== 'PHOTO') {
            const scalePulse = 1 + Math.sin(time * this.pulseSpeed * 0.5 + this.pulsePhase) * 0.1
            this.mesh.scale.setScalar(this.originalScale.x * scalePulse)
          }
        }
      } else {
        // 正常模式下的闪烁效果
        // Softer twinkle patterns
        if (this.twinkleType < 0.25) {
          // Gentle on/off
          const wave = Math.sin(time * this.pulseSpeed + this.pulsePhase)
          intensity = wave > 0.3 ? this.baseEmissive * 1.5 : this.baseEmissive * 0.3
        } else if (this.twinkleType < 0.50) {
          // Double wave
          const wave1 = Math.sin(time * this.pulseSpeed + this.pulsePhase)
          const wave2 = Math.sin(time * this.pulseSpeed2 + this.pulsePhase2)
          intensity = this.baseEmissive * (0.5 + wave1 * 0.25 + wave2 * 0.25)
        } else if (this.twinkleType < 0.70) {
          // Soft flicker
          const flicker = Math.sin(time * 8 + this.pulsePhase) * Math.sin(time * 12 + this.pulsePhase2)
          intensity = this.baseEmissive * (0.6 + flicker * 0.4)
        } else {
          // Smooth breathing
          const breath = (Math.sin(time * this.pulseSpeed * 0.5 + this.pulsePhase) + 1) * 0.5
          intensity = this.baseEmissive * (0.5 + breath * 0.8)
        }
        
        if (this.type !== 'PHOTO') {
          const scalePulse = 1 + Math.sin(time * this.pulseSpeed * 0.5 + this.pulsePhase) * 0.1
          this.mesh.scale.setScalar(this.originalScale.x * scalePulse)
        }
      }
      
      this.mesh.material.emissiveIntensity = intensity
      
      // Color shift for some particles
      if (this.hasColorShift && this.mesh.material.emissive) {
        const hue = (Math.sin(time * this.colorSpeed + this.colorPhase) + 1) * 0.15 + 0.05
        this.mesh.material.emissive.setHSL(hue, 0.7, 0.4)
      }
    }
  }
}

// ============================================
// STAR PARTICLE CLASS - Enhanced
// ============================================
class StarParticle {
  constructor(mesh, index) {
    this.mesh = mesh
    this.index = index
    this.basePosition = new THREE.Vector3(0, 0, 0)
    this.scatterPosition = new THREE.Vector3()
    this.phase = Math.random() * Math.PI * 2
    this.phase2 = Math.random() * Math.PI * 2
    this.twinkleSpeed = 3 + Math.random() * 10
    this.twinkleSpeed2 = 5 + Math.random() * 12
    this.baseOpacity = 0.7 + Math.random() * 0.3
    this.twinkleType = Math.random()
    this.rotationSpeed = Math.random() * 0.02
    
    const radius = 35 + Math.random() * 50
    const theta = Math.random() * Math.PI * 2
    const phi = Math.acos(2 * Math.random() - 1)
    this.scatterPosition.set(
      radius * Math.sin(phi) * Math.cos(theta),
      radius * Math.sin(phi) * Math.sin(theta),
      radius * Math.cos(phi)
    )
    
    mesh.position.copy(this.basePosition)
    mesh.visible = false
  }

  update(time, currentMode) {
    if (currentMode === 'SCATTER' || currentMode === 'FOCUS') {
      this.mesh.position.lerp(this.scatterPosition, 0.04)
      this.mesh.rotation.z += this.rotationSpeed
      
      let twinkle
      if (this.twinkleType < 0.4) {
        const wave = Math.sin(time * this.twinkleSpeed + this.phase)
        twinkle = wave > 0.3 ? 1.0 : 0.15
      } else if (this.twinkleType < 0.7) {
        const wave1 = Math.sin(time * this.twinkleSpeed + this.phase)
        const wave2 = Math.sin(time * this.twinkleSpeed2 + this.phase2)
        twinkle = 0.2 + Math.max(0, wave1) * 0.5 + Math.max(0, wave2) * 0.3
      } else {
        // Sparkle burst
        const sparkle = Math.pow(Math.max(0, Math.sin(time * this.twinkleSpeed + this.phase)), 6)
        twinkle = 0.3 + sparkle * 0.7
      }
      this.mesh.material.opacity = this.baseOpacity * twinkle
      this.mesh.visible = true
    } else {
      this.mesh.position.lerp(this.basePosition, 0.1)
      this.mesh.material.opacity *= 0.92
      if (this.mesh.position.length() < 3) {
        this.mesh.visible = false
      }
    }
  }
}

// ============================================
// SNOWFLAKE CLASS - NEW
// ============================================
class Snowflake {
  constructor(mesh, index) {
    this.mesh = mesh
    this.index = index
    this.velocity = new THREE.Vector3(
      (Math.random() - 0.5) * 0.02,
      -0.05 - Math.random() * 0.1,
      (Math.random() - 0.5) * 0.02
    )
    this.rotationSpeed = new THREE.Vector3(
      (Math.random() - 0.5) * 0.02,
      (Math.random() - 0.5) * 0.02,
      (Math.random() - 0.5) * 0.01
    )
    this.swayPhase = Math.random() * Math.PI * 2
    this.swaySpeed = 1 + Math.random() * 2
    this.swayAmount = 0.02 + Math.random() * 0.03
    
    // Start position
    this.reset()
  }

  reset() {
    const spread = 60
    this.mesh.position.set(
      (Math.random() - 0.5) * spread,
      30 + Math.random() * 20,
      (Math.random() - 0.5) * spread
    )
  }

  update(time) {
    // Add sway
    const sway = Math.sin(time * this.swaySpeed + this.swayPhase) * this.swayAmount
    
    this.mesh.position.x += this.velocity.x + sway
    this.mesh.position.y += this.velocity.y
    this.mesh.position.z += this.velocity.z
    
    this.mesh.rotation.x += this.rotationSpeed.x
    this.mesh.rotation.y += this.rotationSpeed.y
    this.mesh.rotation.z += this.rotationSpeed.z

    // Reset if below ground
    if (this.mesh.position.y < -15) {
      this.reset()
    }
  }
}

// ============================================
// RIBBON CLASS - NEW (Light ribbons wrapping tree)
// ============================================
class LightRibbon {
  constructor(index, totalRibbons) {
    this.index = index
    this.points = []
    this.geometry = null
    this.mesh = null
    this.phaseOffset = (index / totalRibbons) * Math.PI * 2
    this.colorOffset = index / totalRibbons
    this.speed = 0.5 + Math.random() * 0.5
    this.create()
  }

  create() {
    const segments = 100
    const positions = []
    const colors = []
    
    for (let i = 0; i < segments; i++) {
      positions.push(0, 0, 0)
      colors.push(1, 1, 1)
    }

    this.geometry = new THREE.BufferGeometry()
    this.geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
    this.geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3))

    const material = new THREE.LineBasicMaterial({
      vertexColors: true,
      transparent: true,
      opacity: 0.8,
      blending: THREE.AdditiveBlending,
      linewidth: 2
    })

    this.mesh = new THREE.Line(this.geometry, material)
  }

  update(time) {
    const positions = this.geometry.attributes.position.array
    const colors = this.geometry.attributes.color.array
    const segments = positions.length / 3

    for (let i = 0; i < segments; i++) {
      const t = i / segments
      const height = 22
      const maxRadius = 10
      
      // Spiral around tree
      const y = t * height - height * 0.35
      const radius = maxRadius * (1 - t * 0.9) + 0.5
      const angle = t * Math.PI * 8 + time * this.speed + this.phaseOffset
      
      positions[i * 3] = Math.cos(angle) * radius
      positions[i * 3 + 1] = y
      positions[i * 3 + 2] = Math.sin(angle) * radius

      // Rainbow colors
      const hue = (t + time * 0.1 + this.colorOffset) % 1
      const color = new THREE.Color().setHSL(hue, 1.0, 0.6)
      colors[i * 3] = color.r
      colors[i * 3 + 1] = color.g
      colors[i * 3 + 2] = color.b
    }

    this.geometry.attributes.position.needsUpdate = true
    this.geometry.attributes.color.needsUpdate = true
  }
}

// ============================================
// INITIALIZATION
// ============================================
const init = async () => {
  // 检测是否是移动设备
  isMobile.value = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent) 
    || window.innerWidth < 768
  
  setupRenderer()
  setupCamera()
  setupScene()
  setupEnvironment()
  setupLights()
  setupPostProcessing()
  createTopStar()
  createGlowingParticles()
  createStarField()
  createSnowfall()
  createLightRibbons()
  
  // 先尝试加载保存的照片
  loadPhotosFromStorage()
  
  // 如果没有保存的照片，创建默认照片
  setTimeout(() => {
    if (photos.value.length === 0) {
      createDefaultPhoto()
    }
  }, 100)
  
  setupEventListeners()
  
  await setupMediaPipe()
  
  initialized.value = true
  animate()
}

const setupRenderer = () => {
  renderer = new THREE.WebGLRenderer({
    canvas: threeCanvas.value,
    antialias: true,
    alpha: true
  })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.8 // Reduced exposure for softer look
  renderer.outputColorSpace = THREE.SRGBColorSpace
}

const setupCamera = () => {
  camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 2, 50)
  camera.lookAt(0, 3, 0)
}

const setupScene = () => {
  scene = new THREE.Scene()
  mainGroup = new THREE.Group()
  starGroup = new THREE.Group()
  photoGroup = new THREE.Group()
  snowGroup = new THREE.Group()
  ribbonGroup = new THREE.Group()
  scene.add(mainGroup)
  scene.add(starGroup)
  scene.add(photoGroup)
  scene.add(snowGroup)
  mainGroup.add(ribbonGroup)
  clock = new THREE.Clock()
}

const setupEnvironment = () => {
  const pmremGenerator = new THREE.PMREMGenerator(renderer)
  const roomEnv = new RoomEnvironment()
  const envMap = pmremGenerator.fromScene(roomEnv).texture
  scene.environment = envMap
  pmremGenerator.dispose()
}

const setupLights = () => {
  const ambient = new THREE.AmbientLight(0xffffff, 0.25)
  scene.add(ambient)

  pointLight = new THREE.PointLight(0xffbb44, 3, 80)
  pointLight.position.set(0, 5, 0)
  mainGroup.add(pointLight)

  pointLight2 = new THREE.PointLight(0xffee66, 2, 60)
  pointLight2.position.set(0, 14, 0)
  mainGroup.add(pointLight2)

  const pointLight3 = new THREE.PointLight(0xffcc44, 1.5, 50)
  pointLight3.position.set(0, -8, 0)
  mainGroup.add(pointLight3)

  // Softer spotlights
  const spotGold = new THREE.SpotLight(0xffd700, 800)
  spotGold.position.set(30, 40, 40)
  spotGold.angle = Math.PI / 5
  spotGold.penumbra = 0.6
  scene.add(spotGold)

  const spotGold2 = new THREE.SpotLight(0xffbb00, 600)
  spotGold2.position.set(-25, 35, 35)
  spotGold2.angle = Math.PI / 5
  spotGold2.penumbra = 0.6
  scene.add(spotGold2)

  const spotRed = new THREE.SpotLight(0xff4444, 300)
  spotRed.position.set(20, 10, -30)
  spotRed.angle = Math.PI / 6
  spotRed.penumbra = 0.7
  scene.add(spotRed)

  const spotGreen = new THREE.SpotLight(0x44ff44, 250)
  spotGreen.position.set(-20, 10, -30)
  spotGreen.angle = Math.PI / 6
  spotGreen.penumbra = 0.7
  scene.add(spotGreen)

  const spotBlue = new THREE.SpotLight(0x4477ff, 200)
  spotBlue.position.set(-30, 20, -30)
  spotBlue.angle = Math.PI / 5
  spotBlue.penumbra = 0.6
  scene.add(spotBlue)
}

const setupPostProcessing = () => {
  composer = new EffectComposer(renderer)
  const renderPass = new RenderPass(scene, camera)
  composer.addPass(renderPass)

  // Bloom只影响发光物体，提高threshold避免影响照片
  const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    0.8,  // strength - softer
    0.5,  // radius
    0.6   // threshold - 提高阈值，只有高亮度物体才会bloom
  )
  composer.addPass(bloomPass)
}

// ============================================
// CREATE TOP STAR - NEW
// ============================================
const createTopStar = () => {
  const starShape = new THREE.Shape()
  const outerRadius = 1.2
  const innerRadius = 0.5
  const points = 5

  for (let i = 0; i < points * 2; i++) {
    const radius = i % 2 === 0 ? outerRadius : innerRadius
    const angle = (i / (points * 2)) * Math.PI * 2 - Math.PI / 2
    const x = Math.cos(angle) * radius
    const y = Math.sin(angle) * radius
    if (i === 0) starShape.moveTo(x, y)
    else starShape.lineTo(x, y)
  }
  starShape.closePath()

  const extrudeSettings = { depth: 0.3, bevelEnabled: true, bevelThickness: 0.1, bevelSize: 0.1 }
  const geometry = new THREE.ExtrudeGeometry(starShape, extrudeSettings)

  const material = new THREE.MeshPhysicalMaterial({
    color: 0xffd700,
    metalness: 0.9,
    roughness: 0.1,
    emissive: 0xcc8800,
    emissiveIntensity: 1.0,
    clearcoat: 0.8,
    clearcoatRoughness: 0.1
  })

  topStar = new THREE.Mesh(geometry, material)
  topStar.position.set(0, 15, 0)
  topStar.rotation.x = Math.PI / 2
  
  // Softer point light at star
  const starLight = new THREE.PointLight(0xffd700, 3, 25)
  topStar.add(starLight)
  
  mainGroup.add(topStar)
}

const createCandyCaneTexture = () => {
  const canvas = document.createElement('canvas')
  canvas.width = 256
  canvas.height = 256
  const ctx = canvas.getContext('2d')
  ctx.fillStyle = '#ffffff'
  ctx.fillRect(0, 0, 256, 256)
  ctx.strokeStyle = '#cc0000'
  ctx.lineWidth = 20
  for (let i = -256; i < 512; i += 40) {
    ctx.beginPath()
    ctx.moveTo(i, 0)
    ctx.lineTo(i + 256, 256)
    ctx.stroke()
  }
  const texture = new THREE.CanvasTexture(canvas)
  texture.wrapS = THREE.RepeatWrapping
  texture.wrapT = THREE.RepeatWrapping
  texture.repeat.set(1, 4)
  return texture
}

const createCandyCane = () => {
  const curve = new THREE.CatmullRomCurve3([
    new THREE.Vector3(0, 0, 0),
    new THREE.Vector3(0, 1.5, 0),
    new THREE.Vector3(0.2, 2.2, 0),
    new THREE.Vector3(0.6, 2.5, 0),
    new THREE.Vector3(1, 2.4, 0),
    new THREE.Vector3(1.2, 2.1, 0)
  ])
  const geometry = new THREE.TubeGeometry(curve, 32, 0.1, 8, false)
  const material = new THREE.MeshStandardMaterial({
    map: createCandyCaneTexture(),
    roughness: 0.3,
    metalness: 0.1
  })
  return new THREE.Mesh(geometry, material)
}

const createGlowingParticles = () => {
  const particleCount = 2000 // Balanced particle count

  // Materials with softer glow - more Christmas-like
  const goldGlowMaterial = new THREE.MeshStandardMaterial({
    color: 0xffd700, metalness: 0.8, roughness: 0.2,
    emissive: 0xcc9900, emissiveIntensity: 0.6
  })
  const brightGoldMaterial = new THREE.MeshStandardMaterial({
    color: 0xffdd00, metalness: 0.8, roughness: 0.2,
    emissive: 0xbb8800, emissiveIntensity: 0.8
  })
  const warmWhiteMaterial = new THREE.MeshStandardMaterial({
    color: 0xfffaf0, metalness: 0.7, roughness: 0.2,
    emissive: 0xffeedd, emissiveIntensity: 0.5
  })
  const orangeMaterial = new THREE.MeshStandardMaterial({
    color: 0xffaa00, metalness: 0.8, roughness: 0.2,
    emissive: 0xcc6600, emissiveIntensity: 0.6
  })
  const redMaterial = new THREE.MeshPhysicalMaterial({
    color: 0xff4444, metalness: 0.7, roughness: 0.15,
    clearcoat: 0.8, clearcoatRoughness: 0.1,
    emissive: 0xaa0000, emissiveIntensity: 0.5
  })
  const greenMaterial = new THREE.MeshPhysicalMaterial({
    color: 0x44dd44, metalness: 0.7, roughness: 0.15,
    clearcoat: 0.8, clearcoatRoughness: 0.1,
    emissive: 0x008800, emissiveIntensity: 0.4
  })
  const blueMaterial = new THREE.MeshStandardMaterial({
    color: 0x6699ff, metalness: 0.8, roughness: 0.2,
    emissive: 0x2244aa, emissiveIntensity: 0.5
  })
  const pinkMaterial = new THREE.MeshStandardMaterial({
    color: 0xff77aa, metalness: 0.8, roughness: 0.2,
    emissive: 0xcc2266, emissiveIntensity: 0.4
  })
  const purpleMaterial = new THREE.MeshStandardMaterial({
    color: 0xaa77ff, metalness: 0.8, roughness: 0.2,
    emissive: 0x6622aa, emissiveIntensity: 0.4
  })

  const tinySpereGeom = new THREE.SphereGeometry(0.06, 12, 12)
  const smallSpereGeom = new THREE.SphereGeometry(0.12, 16, 16)
  const mediumSpereGeom = new THREE.SphereGeometry(0.22, 16, 16)
  const largeSpereGeom = new THREE.SphereGeometry(0.35, 16, 16)

  for (let i = 0; i < particleCount; i++) {
    let mesh
    const type = Math.random()

    if (type < 0.25) {
      mesh = new THREE.Mesh(tinySpereGeom, goldGlowMaterial.clone())
    } else if (type < 0.40) {
      mesh = new THREE.Mesh(smallSpereGeom, brightGoldMaterial.clone())
    } else if (type < 0.52) {
      mesh = new THREE.Mesh(smallSpereGeom, warmWhiteMaterial.clone())
    } else if (type < 0.62) {
      mesh = new THREE.Mesh(mediumSpereGeom, orangeMaterial.clone())
    } else if (type < 0.72) {
      mesh = new THREE.Mesh(largeSpereGeom, redMaterial.clone())
    } else if (type < 0.80) {
      mesh = new THREE.Mesh(mediumSpereGeom, greenMaterial.clone())
    } else if (type < 0.86) {
      mesh = new THREE.Mesh(smallSpereGeom, blueMaterial.clone())
    } else if (type < 0.91) {
      mesh = new THREE.Mesh(smallSpereGeom, pinkMaterial.clone())
    } else if (type < 0.96) {
      mesh = new THREE.Mesh(smallSpereGeom, purpleMaterial.clone())
    } else {
      mesh = createCandyCane()
      mesh.scale.set(0.35, 0.35, 0.35)
    }

    mesh.rotation.set(Math.random() * Math.PI, Math.random() * Math.PI, Math.random() * Math.PI)
    const particle = new Particle(mesh, 'ORNAMENT', i)
    particle.updateTreePosition()
    mesh.position.copy(particle.targetPosition)
    particles.push(particle)
    mainGroup.add(mesh)
  }
}

const createStarField = () => {
  const starCount = 4000 // More stars
  const starGeomTiny = new THREE.SphereGeometry(0.02, 6, 6)
  const starGeomSmall = new THREE.SphereGeometry(0.05, 8, 8)
  const starGeomMedium = new THREE.SphereGeometry(0.09, 8, 8)

  // Add some cross-shaped stars
  const createStarShape = () => {
    const shape = new THREE.Shape()
    const size = 0.08
    shape.moveTo(0, size)
    shape.lineTo(size * 0.3, size * 0.3)
    shape.lineTo(size, 0)
    shape.lineTo(size * 0.3, -size * 0.3)
    shape.lineTo(0, -size)
    shape.lineTo(-size * 0.3, -size * 0.3)
    shape.lineTo(-size, 0)
    shape.lineTo(-size * 0.3, size * 0.3)
    shape.closePath()
    return new THREE.ShapeGeometry(shape)
  }
  const starShapeGeom = createStarShape()

  for (let i = 0; i < starCount; i++) {
    const size = Math.random()
    let geom
    if (size < 0.5) geom = starGeomTiny
    else if (size < 0.8) geom = starGeomSmall
    else if (size < 0.95) geom = starGeomMedium
    else geom = starShapeGeom

    const brightness = 0.7 + Math.random() * 0.3
    const hue = Math.random() < 0.7 ? 0.12 : (Math.random() < 0.5 ? 0.6 : 0.0)
    const color = new THREE.Color().setHSL(hue, 0.3, brightness)

    const material = new THREE.MeshBasicMaterial({
      color, transparent: true, opacity: 0,
      blending: THREE.AdditiveBlending
    })
    const mesh = new THREE.Mesh(geom, material)
    mesh.visible = false

    const star = new StarParticle(mesh, i)
    starParticles.push(star)
    starGroup.add(mesh)
  }
}

// ============================================
// CREATE SNOWFALL - NEW
// ============================================
const createSnowfall = () => {
  const snowCount = 800

  // Different snowflake geometries
  const snowGeomTiny = new THREE.SphereGeometry(0.03, 6, 6)
  const snowGeomSmall = new THREE.SphereGeometry(0.06, 8, 8)
  const snowGeomMedium = new THREE.SphereGeometry(0.1, 8, 8)

  // Create hexagonal snowflake shape
  const createHexFlake = () => {
    const group = new THREE.Group()
    const material = new THREE.MeshBasicMaterial({
      color: 0xffffff,
      transparent: true,
      opacity: 0.8,
      blending: THREE.AdditiveBlending
    })

    for (let i = 0; i < 6; i++) {
      const arm = new THREE.Mesh(
        new THREE.BoxGeometry(0.02, 0.15, 0.01),
        material
      )
      arm.rotation.z = (i / 6) * Math.PI * 2
      arm.position.y = 0.07
      group.add(arm)
    }
    return group
  }

  const snowMaterial = new THREE.MeshBasicMaterial({
    color: 0xffffff,
    transparent: true,
    opacity: 0.85,
    blending: THREE.AdditiveBlending
  })

  for (let i = 0; i < snowCount; i++) {
    let mesh
    const type = Math.random()

    if (type < 0.5) {
      mesh = new THREE.Mesh(snowGeomTiny, snowMaterial.clone())
    } else if (type < 0.8) {
      mesh = new THREE.Mesh(snowGeomSmall, snowMaterial.clone())
    } else if (type < 0.95) {
      mesh = new THREE.Mesh(snowGeomMedium, snowMaterial.clone())
    } else {
      mesh = createHexFlake()
    }

    const snowflake = new Snowflake(mesh, i)
    snowParticles.push(snowflake)
    snowGroup.add(mesh)
  }
}

// ============================================
// CREATE LIGHT RIBBONS - NEW
// ============================================
const createLightRibbons = () => {
  const ribbonCount = 4

  for (let i = 0; i < ribbonCount; i++) {
    const ribbon = new LightRibbon(i, ribbonCount)
    ribbons.push(ribbon)
    ribbonGroup.add(ribbon.mesh)
  }
}

const createPhotoWithFrame = (texture) => {
  const group = new THREE.Group()

  // Photo - 使用特殊设置保持原图效果
  const photoGeom = new THREE.PlaneGeometry(2, 2)
  const photoMat = new THREE.MeshBasicMaterial({
    map: texture,
    side: THREE.DoubleSide,
    toneMapped: false,  // 禁用色调映射
    transparent: false
  })
  const photo = new THREE.Mesh(photoGeom, photoMat)
  photo.renderOrder = 999  // 最后渲染
  group.add(photo)

  // Gold frame with glow
  const frameThickness = 0.1
  const frameDepth = 0.08
  const frameMaterial = new THREE.MeshStandardMaterial({
    color: 0xffd700, metalness: 1.0, roughness: 0.1,
    emissive: 0xaa7700, emissiveIntensity: 0.5
  })

  const frameGeomH = new THREE.BoxGeometry(2.25, frameThickness, frameDepth)
  const frameGeomV = new THREE.BoxGeometry(frameThickness, 2.25, frameDepth)

  const frameTop = new THREE.Mesh(frameGeomH, frameMaterial)
  frameTop.position.set(0, 1.08, 0)
  group.add(frameTop)

  const frameBottom = new THREE.Mesh(frameGeomH, frameMaterial)
  frameBottom.position.set(0, -1.08, 0)
  group.add(frameBottom)

  const frameLeft = new THREE.Mesh(frameGeomV, frameMaterial)
  frameLeft.position.set(-1.08, 0, 0)
  group.add(frameLeft)

  const frameRight = new THREE.Mesh(frameGeomV, frameMaterial)
  frameRight.position.set(1.08, 0, 0)
  group.add(frameRight)

  // Add corner decorations
  const cornerGeom = new THREE.SphereGeometry(0.08, 8, 8)
  const cornerMaterial = new THREE.MeshStandardMaterial({
    color: 0xffd700, metalness: 1.0, roughness: 0.0,
    emissive: 0xffaa00, emissiveIntensity: 1.0
  })

  const corners = [[-1.08, 1.08], [1.08, 1.08], [-1.08, -1.08], [1.08, -1.08]]
  corners.forEach(([x, y]) => {
    const corner = new THREE.Mesh(cornerGeom, cornerMaterial)
    corner.position.set(x, y, 0.04)
    group.add(corner)
  })

  group.userData.type = 'PHOTO'
  return group
}

const createPhotoTexture = (text) => {
  const canvas = document.createElement('canvas')
  canvas.width = 512
  canvas.height = 512
  const ctx = canvas.getContext('2d')

  const gradient = ctx.createLinearGradient(0, 0, 512, 512)
  gradient.addColorStop(0, '#1a0a2e')
  gradient.addColorStop(0.5, '#0d1a2d')
  gradient.addColorStop(1, '#0d0d0d')
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, 512, 512)

  // Decorative border
  ctx.strokeStyle = '#d4af37'
  ctx.lineWidth = 8
  ctx.strokeRect(20, 20, 472, 472)
  ctx.strokeStyle = 'rgba(212, 175, 55, 0.5)'
  ctx.lineWidth = 2
  ctx.strokeRect(35, 35, 442, 442)

  ctx.fillStyle = '#d4af37'
  ctx.font = 'bold 48px Cinzel, serif'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText(text, 256, 256)

  // More snowflakes
  ctx.font = '24px serif'
  ctx.fillStyle = 'rgba(252, 238, 167, 0.5)'
  for (let i = 0; i < 30; i++) {
    ctx.fillText('❄', Math.random() * 512, Math.random() * 512)
  }

  // Stars
  ctx.fillStyle = 'rgba(255, 215, 0, 0.4)'
  for (let i = 0; i < 15; i++) {
    ctx.fillText('✦', Math.random() * 512, Math.random() * 512)
  }

  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  return texture
}

const createDefaultPhoto = () => {
  // 创建默认照片的Canvas
  const canvas = document.createElement('canvas')
  canvas.width = 512
  canvas.height = 512
  const ctx = canvas.getContext('2d')

  const gradient = ctx.createLinearGradient(0, 0, 512, 512)
  gradient.addColorStop(0, '#1a0a2e')
  gradient.addColorStop(0.5, '#0d1a2d')
  gradient.addColorStop(1, '#0d0d0d')
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, 512, 512)

  ctx.strokeStyle = '#d4af37'
  ctx.lineWidth = 8
  ctx.strokeRect(20, 20, 472, 472)
  ctx.strokeStyle = 'rgba(212, 175, 55, 0.5)'
  ctx.lineWidth = 2
  ctx.strokeRect(35, 35, 442, 442)

  ctx.fillStyle = '#d4af37'
  ctx.font = 'bold 48px serif'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText('圣诞节快乐！', 256, 256)

  ctx.font = '24px serif'
  ctx.fillStyle = 'rgba(252, 238, 167, 0.5)'
  for (let i = 0; i < 30; i++) {
    ctx.fillText('❄', Math.random() * 512, Math.random() * 512)
  }

  const dataUrl = canvas.toDataURL('image/png')
  
  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  addPhotoToScene(texture, dataUrl, false)  // 默认照片不保存到storage，但有dataUrl用于显示
}


const addPhotoToScene = (texture, dataUrl = null, shouldSave = true) => {
  // 创建带相框的照片网格并设置缩放比例
  const photoGroupMesh = createPhotoWithFrame(texture)
  photoGroupMesh.scale.set(0.4, 0.4, 0.4)

  // 创建照片粒子对象并设置随机位置参数
  const particle = new Particle(photoGroupMesh, 'PHOTO', particles.length)
  particle.treeT = 0.2 + Math.random() * 0.5  // 树干高度比例
  particle.treeAngle = Math.random() * Math.PI * 2  // 树周角度
  particle.originalScale = photoGroupMesh.scale.clone()  // 保存原始缩放
  particle.updateTreePosition(8, 18)  // 更新在树上的位置
  photoGroupMesh.position.copy(particle.targetPosition)  // 应用目标位置

  // 将照片添加到粒子和照片数组，并添加到主组
  particles.push(particle)
  photos.value.push(particle)
  mainGroup.add(photoGroupMesh)
  
  // 保存照片数据URL到存储
  if (dataUrl) {
    photoDataUrls.value.push(dataUrl)
    if (shouldSave) {
      savePhotosToStorage()
    }
  }
  
  // 更新当前照片索引指向最新添加的照片
  currentPhotoIndex.value = photos.value.length - 1
}

// localStorage操作
const STORAGE_KEY = 'christmas-tree-photos'

const savePhotosToStorage = () => {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(photoDataUrls.value))
  } catch (e) {
    console.warn('Failed to save photos:', e)
  }
}

const loadPhotosFromStorage = () => {
  try {
    const saved = localStorage.getItem(STORAGE_KEY)
    if (saved) {
      const urls = JSON.parse(saved)
      urls.forEach(url => {
        new THREE.TextureLoader().load(url, (t) => {
          t.colorSpace = THREE.SRGBColorSpace
          addPhotoToScene(t, url, false) // false = 不重复保存
        })
      })
    }
  } catch (e) {
    console.warn('Failed to load photos:', e)
  }
}

const deletePhoto = (index) => {
  if (index < 0 || index >= photos.value.length) return
  
  // 移除3D场景中的照片
  const particle = photos.value[index]
  if (particle && particle.mesh) {
    mainGroup.remove(particle.mesh)
    photoGroup.remove(particle.mesh)
    // 从particles数组中移除
    const particleIndex = particles.indexOf(particle)
    if (particleIndex > -1) {
      particles.splice(particleIndex, 1)
    }
  }
  
  // 从数组中移除
  photos.value.splice(index, 1)
  photoDataUrls.value.splice(index, 1)
  
  // 更新索引
  if (photos.value.length === 0) {
    currentPhotoIndex.value = 0
    focusedPhotoIndex.value = -1
    if (mode.value === 'FOCUS') {
      targetMode.value = 'TREE'
    }
  } else {
    if (currentPhotoIndex.value >= photos.value.length) {
      currentPhotoIndex.value = photos.value.length - 1
    }
    if (focusedPhotoIndex.value >= photos.value.length) {
      focusedPhotoIndex.value = photos.value.length - 1
    }
  }
  
  savePhotosToStorage()
}

const clearAllPhotos = () => {
  // 移除所有照片
  photos.value.forEach((particle) => {
    if (particle && particle.mesh) {
      mainGroup.remove(particle.mesh)
      photoGroup.remove(particle.mesh)
      const particleIndex = particles.indexOf(particle)
      if (particleIndex > -1) {
        particles.splice(particleIndex, 1)
      }
    }
  })
  
  photos.value = []
  photoDataUrls.value = []
  currentPhotoIndex.value = 0
  focusedPhotoIndex.value = -1
  
  if (mode.value === 'FOCUS') {
    targetMode.value = 'TREE'
  }
  
  localStorage.removeItem(STORAGE_KEY)
}

// File upload handler
const handleFileUpload = (e) => {
  const files = e.target.files
  if (files && files.length > 0) {
    Array.from(files).forEach(file => {
      const reader = new FileReader()
      reader.onload = (ev) => {
        const dataUrl = ev.target.result
        new THREE.TextureLoader().load(dataUrl, (t) => {
          t.colorSpace = THREE.SRGBColorSpace
          addPhotoToScene(t, dataUrl, true)  // true = 保存到storage
        })
      }
      reader.readAsDataURL(file)
    })
  }
  e.target.value = ''
}

// 计算3D位置到屏幕2D位置
const getScreenPosition = (particle) => {
  if (!particle || !particle.mesh || !camera) {
    return { x: window.innerWidth / 2, y: window.innerHeight / 2 }
  }
  
  const vector = new THREE.Vector3()
  particle.mesh.getWorldPosition(vector)
  vector.project(camera)
  
  const x = (vector.x * 0.5 + 0.5) * window.innerWidth
  const y = (-(vector.y * 0.5) + 0.5) * window.innerHeight
  
  return { x, y }
}

// 触发照片动画 - 随机选择动画类型
const triggerPhotoAnimation = (photoIndex) => {
  const particle = photos.value[photoIndex]
  if (particle) {
    const pos = getScreenPosition(particle)
    photoStartPos.value = { 
      x: pos.x, 
      y: pos.y, 
      scale: 0.15
    }
    
    // 随机选择动画类型
    currentAnimation.value = animationTypes[Math.floor(Math.random() * animationTypes.length)]
    photoAnimating.value = true
    
    // 动画1秒后重置状态
    setTimeout(() => {
      photoAnimating.value = false
    }, 1000)
  }
}

// Switch to next photo
const nextPhoto = () => {
  if (photos.value.length > 0) {
    currentPhotoIndex.value = (currentPhotoIndex.value + 1) % photos.value.length
    focusedPhotoIndex.value = currentPhotoIndex.value
    photoZoomScale = 5 // Reset zoom
    
    // 如果在FOCUS模式，触发动画
    if (mode.value === 'FOCUS') {
      triggerPhotoAnimation(focusedPhotoIndex.value)
    }
  }
}

// MediaPipe setup
const setupMediaPipe = async () => {
  try {
    const vision = await FilesetResolver.forVisionTasks(
      "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.3/wasm"
    )
    handLandmarker = await HandLandmarker.createFromOptions(vision, {
      baseOptions: {
        modelAssetPath: "https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task",
        delegate: "GPU"
      },
      runningMode: "VIDEO",
      numHands: 1
    })
    await setupWebcam()
  } catch (error) {
    console.warn('MediaPipe initialization failed:', error)
  }
}

const setupWebcam = async () => {
  // 检查是否支持摄像头API
  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    console.warn('MediaDevices API not supported')
    detectedGesture.value = '❌ 浏览器不支持'
    webcamActive.value = false
    return
  }

  // 检查是否是安全上下文（HTTPS或localhost）
  const isSecure = window.isSecureContext
  const isLocalhost = location.hostname === 'localhost' || location.hostname === '127.0.0.1'
  
  if (!isSecure && !isLocalhost) {
    console.warn('Insecure context - camera requires HTTPS or localhost')
    detectedGesture.value = '⚠️ 需要HTTPS访问'
    webcamActive.value = false
    return
  }

  try {
    const stream = await navigator.mediaDevices.getUserMedia({
      video: { width: 320, height: 240, facingMode: 'user' }
    })
    webcamVideo.value.srcObject = stream
    webcamVideo.value.play()
    webcamVideo.value.addEventListener('loadeddata', () => {
      webcamActive.value = true
      detectedGesture.value = '✅ 摄像头已连接'
      processHands()
    })
  } catch (error) {
    console.warn('Webcam access denied:', error)
    if (error.name === 'NotAllowedError') {
      detectedGesture.value = '❌ 请允许摄像头权限'
    } else if (error.name === 'NotFoundError') {
      detectedGesture.value = '❌ 未找到摄像头'
    } else {
      detectedGesture.value = '❌ 摄像头未授权'
    }
    webcamActive.value = false
  }
}

const processHands = () => {
  if (!handLandmarker) return
  const video = webcamVideo.value
  const now = performance.now()

  if (video.readyState >= 2) {
    const results = handLandmarker.detectForVideo(video, now)
    if (results.landmarks && results.landmarks.length > 0) {
      processGestures(results.landmarks[0])
    }
  }
  requestAnimationFrame(processHands)
}

const processGestures = (landmarks) => {
  // Key landmarks - 21个手部关键点
  const wrist = landmarks[0]
  const thumbTip = landmarks[4]
  const thumbIP = landmarks[3]
  const thumbMCP = landmarks[2]
  const indexTip = landmarks[8]
  const indexDIP = landmarks[7]
  const indexPIP = landmarks[6]
  const indexMCP = landmarks[5]
  const middleTip = landmarks[12]
  const middleDIP = landmarks[11]
  const middlePIP = landmarks[10]
  const middleMCP = landmarks[9]
  const ringTip = landmarks[16]
  const ringDIP = landmarks[15]
  const ringPIP = landmarks[14]
  const ringMCP = landmarks[13]
  const pinkyTip = landmarks[20]
  const pinkyDIP = landmarks[19]
  const pinkyPIP = landmarks[18]
  const pinkyMCP = landmarks[17]

  // 计算手掌大小用于归一化（手腕到中指MCP的距离）
  const palmSize = Math.hypot(
    middleMCP.x - wrist.x,
    middleMCP.y - wrist.y,
    middleMCP.z - wrist.z
  )
  
  // 归一化距离计算
  const normalizedDist = (a, b) => {
    const d = Math.hypot(a.x - b.x, a.y - b.y, a.z - b.z)
    return d / palmSize
  }

  // 检查手指是否弯曲 - 使用指尖到MCP的距离与PIP到MCP的距离比较
  const isFingerCurled = (tip, dip, pip, mcp) => {
    const tipToMcp = normalizedDist(tip, mcp)
    const pipToMcp = normalizedDist(pip, mcp)
    // 如果指尖到MCP的距离小于PIP到MCP距离的1.5倍，认为是弯曲的
    return tipToMcp < pipToMcp * 1.6
  }
  
  // 检查手指是否伸展
  const isFingerExtended = (tip, dip, pip, mcp) => {
    const tipToMcp = normalizedDist(tip, mcp)
    const pipToMcp = normalizedDist(pip, mcp)
    // 伸展时指尖距离MCP应该明显大于PIP到MCP的距离
    return tipToMcp > pipToMcp * 1.8
  }

  // ===== 手势检测 =====
  
  // 1. 检测各手指状态
  const indexCurled = isFingerCurled(indexTip, indexDIP, indexPIP, indexMCP)
  const middleCurled = isFingerCurled(middleTip, middleDIP, middlePIP, middleMCP)
  const ringCurled = isFingerCurled(ringTip, ringDIP, ringPIP, ringMCP)
  const pinkyCurled = isFingerCurled(pinkyTip, pinkyDIP, pinkyPIP, pinkyMCP)
  
  const indexExtended = isFingerExtended(indexTip, indexDIP, indexPIP, indexMCP)
  const middleExtended = isFingerExtended(middleTip, middleDIP, middlePIP, middleMCP)
  const ringExtended = isFingerExtended(ringTip, ringDIP, ringPIP, ringMCP)
  const pinkyExtended = isFingerExtended(pinkyTip, pinkyDIP, pinkyPIP, pinkyMCP)
  
  const curledCount = [indexCurled, middleCurled, ringCurled, pinkyCurled].filter(Boolean).length
  const extendedCount = [indexExtended, middleExtended, ringExtended, pinkyExtended].filter(Boolean).length
  
  // 2. 捏合检测 - 拇指和食指尖靠近，但其他手指不能全部弯曲
  const thumbIndexDist = normalizedDist(thumbTip, indexTip)
  // 真正的捏合：拇指食指靠近 + (中指或无名指或小指至少有一个伸展)
  const isPinch = thumbIndexDist < 0.4 && (middleExtended || ringExtended || pinkyExtended)
  
  // 3. 握拳检测 - 所有四指弯曲，且指尖都靠近手掌
  const allFingersCurled = curledCount >= 3
  const tipsToPalm = (normalizedDist(indexTip, wrist) + normalizedDist(middleTip, wrist) + 
                      normalizedDist(ringTip, wrist) + normalizedDist(pinkyTip, wrist)) / 4
  const isFist = allFingersCurled && tipsToPalm < 1.2 && !isPinch
  
  // 4. 张开手检测 - 大部分手指伸展
  const isOpenHand = extendedCount >= 3 && !isPinch
  
  // 确定手势（优先级：捏合 > 握拳 > 张开手）
  let newGesture = 'NONE'
  const now = Date.now()
  
  if (isPinch) {
    newGesture = 'PINCH'
  } else if (isFist) {
    newGesture = 'FIST'
  } else if (isOpenHand) {
    newGesture = 'OPEN'
  }
  
  // ============================================
  // 手势稳定性检测 - 需要持续一段时间才确认
  // ============================================
  
  // 如果检测到新手势
  if (newGesture !== 'NONE') {
    // 首次检测到手势时，初始化待确认手势
    if (isFirstGestureDetection) {
      pendingGesture = newGesture
      pendingGestureStartTime = now
      isFirstGestureDetection = false
    }
    
    // 如果与待确认手势相同，继续计时
    if (newGesture === pendingGesture) {
      const holdTime = now - pendingGestureStartTime
      gestureConfirmProgress = Math.min(1, holdTime / GESTURE_CONFIRM_TIME)
      
      // 更新显示，显示确认进度
      const progressBar = '█'.repeat(Math.floor(gestureConfirmProgress * 5)) + '░'.repeat(5 - Math.floor(gestureConfirmProgress * 5))
      
      if (newGesture === 'PINCH') {
        if (confirmedGesture === 'PINCH') {
          detectedGesture.value = `🤏 捏合中 - 照片 ${focusedPhotoIndex.value + 1}/${photos.value.length}`
        } else {
          detectedGesture.value = `🤏 捏合中 [${progressBar}]`
        }
      } else if (newGesture === 'FIST') {
        if (confirmedGesture === 'FIST') {
          detectedGesture.value = `✊ 握拳 → 圣诞树模式`
        } else {
          detectedGesture.value = `✊ 握拳中 [${progressBar}]`
        }
      } else if (newGesture === 'OPEN') {
        if (confirmedGesture === 'OPEN') {
          detectedGesture.value = `✋ 张开手 → 散开模式`
        } else {
          detectedGesture.value = `✋ 张开手 [${progressBar}]`
        }
      }
      
      // 如果持续时间达到确认阈值，确认手势
      if (holdTime >= GESTURE_CONFIRM_TIME && confirmedGesture !== newGesture) {
        // 手势确认！执行模式切换
        previousGesture = confirmedGesture
        confirmedGesture = newGesture
        lastGestureTime = now
        
        // 重置计时器，准备下一次手势切换
        pendingGestureStartTime = now
        
        if (newGesture === 'PINCH') {
          if (photos.value.length > 0) {
            targetMode.value = 'FOCUS'
            
            // 只有从其他手势切换到捏合时，才切换到下一张照片
            if (previousGesture !== 'PINCH' && !hasShownPhotoThisPinch) {
              if (previousGesture === 'OPEN' || previousGesture === 'FIST' || previousGesture === 'NONE') {
                focusedPhotoIndex.value = (focusedPhotoIndex.value + 1) % photos.value.length
              }
              currentPhotoIndex.value = focusedPhotoIndex.value
              hasShownPhotoThisPinch = true
              triggerPhotoAnimation(focusedPhotoIndex.value)
            }
          }
        } else if (newGesture === 'FIST') {
          targetMode.value = 'TREE'
          lastPinchDistance = null
          hasShownPhotoThisPinch = false
        } else if (newGesture === 'OPEN') {
          targetMode.value = 'SCATTER'
          lastPinchDistance = null
          hasShownPhotoThisPinch = false
        }
        
        // 更新currentGesture用于其他逻辑
        currentGesture = newGesture
      } else if (confirmedGesture === newGesture) {
        // 已确认的手势，保持计时器更新以便切换检测
        pendingGestureStartTime = now
      }
    } else {
      // 检测到不同的手势，重新开始计时
      pendingGesture = newGesture
      pendingGestureStartTime = now
      gestureConfirmProgress = 0
      
      // 显示新检测到的手势
      if (newGesture === 'PINCH') {
        detectedGesture.value = `🤏 检测到捏合...`
      } else if (newGesture === 'FIST') {
        detectedGesture.value = `✊ 检测到握拳...`
      } else if (newGesture === 'OPEN') {
        detectedGesture.value = `✋ 检测到张开手...`
      }
    }
  } else {
    // 没有检测到明确手势时，保持一定的容错性
    // 给用户一点时间调整手势
    if (pendingGesture !== 'NONE' && gestureConfirmProgress > 0) {
      // 如果正在确认中，允许短暂中断
      gestureConfirmProgress = Math.max(0, gestureConfirmProgress - 0.05)
      if (gestureConfirmProgress <= 0) {
        pendingGesture = 'NONE'
        pendingGestureStartTime = 0
      }
      detectedGesture.value = `🖐️ 识别中...`
    } else {
      // 重置状态
      pendingGesture = 'NONE'
      pendingGestureStartTime = 0
      gestureConfirmProgress = 0
      detectedGesture.value = `🖐️ 识别中...`
    }
  }
  
  // 持续捏合时保持FOCUS模式（只有已确认的捏合才生效）
  if (confirmedGesture === 'PINCH' && photos.value.length > 0) {
    targetMode.value = 'FOCUS'
  }
  
  // FOCUS模式下的缩放 - 只有确认的捏合手势才触发缩放
  if (mode.value === 'FOCUS' && confirmedGesture === 'PINCH' && isPinch) {
    const thumbMiddleDist = normalizedDist(thumbTip, middleTip)
    if (lastPinchDistance !== null) {
      const zoomDelta = (thumbMiddleDist - lastPinchDistance) * 15
      photoZoomScale = Math.max(3, Math.min(12, photoZoomScale + zoomDelta))
    }
    lastPinchDistance = thumbMiddleDist
  } else if (confirmedGesture !== 'PINCH') {
    lastPinchDistance = null
  }

  // 更新手部旋转数据
  const palmCenter = landmarks[9]
  handData = {
    rotationX: (palmCenter.y - 0.5) * Math.PI * 0.4,
    rotationY: (palmCenter.x - 0.5) * Math.PI * 0.6
  }
}

// Event listeners
const setupEventListeners = () => {
  window.addEventListener('resize', onResize)
  document.addEventListener('keydown', onKeyDown)
  document.addEventListener('mousemove', onMouseMove)
  document.addEventListener('wheel', onWheel)
  
  // 触摸控制支持（手机端）
  document.addEventListener('touchstart', onTouchStart, { passive: false })
  document.addEventListener('touchmove', onTouchMove, { passive: false })
  document.addEventListener('touchend', onTouchEnd)
}

// 触摸状态
let touchStartX = 0
let touchStartY = 0
let touchStartTime = 0
let lastTapTime = 0

const onTouchStart = (e) => {
  if (e.target.closest('.upload-wrapper, .photo-manager, .gesture-panel')) return
  
  touchStartX = e.touches[0].clientX
  touchStartY = e.touches[0].clientY
  touchStartTime = Date.now()
}

const onTouchMove = (e) => {
  if (e.target.closest('.upload-wrapper, .photo-manager, .gesture-panel')) return
  
  const deltaX = e.touches[0].clientX - touchStartX
  const deltaY = e.touches[0].clientY - touchStartY
  
  // 更新视角
  mouseInteraction.x = deltaX * 0.002
  mouseInteraction.y = deltaY * 0.002
}

const onTouchEnd = (e) => {
  if (e.target.closest('.upload-wrapper, .photo-manager, .gesture-panel, .mobile-hint')) return
  
  const touchDuration = Date.now() - touchStartTime
  const now = Date.now()
  const deltaX = Math.abs(e.changedTouches[0].clientX - touchStartX)
  const deltaY = Math.abs(e.changedTouches[0].clientY - touchStartY)
  const isSwipe = deltaX > 30 || deltaY > 30
  
  // 如果是滑动，不处理点击
  if (isSwipe) {
    lastTapTime = 0
    return
  }
  
  // 双击检测
  const isDoubleTap = now - lastTapTime < 300
  
  if (mode.value === 'FOCUS') {
    // FOCUS模式下
    if (isDoubleTap) {
      // 双击返回圣诞树模式
      targetMode.value = 'TREE'
      lastTapTime = 0
    } else if (touchDuration < 300) {
      // 单击切换下一张照片
      if (photos.value.length > 1) {
        focusedPhotoIndex.value = (focusedPhotoIndex.value + 1) % photos.value.length
        currentPhotoIndex.value = focusedPhotoIndex.value
        triggerPhotoAnimation(focusedPhotoIndex.value)
      }
      lastTapTime = now
    }
  } else {
    // 非FOCUS模式：双击切换模式
    if (isDoubleTap) {
      if (mode.value === 'TREE') {
        targetMode.value = 'SCATTER'
      } else if (mode.value === 'SCATTER') {
        if (photos.value.length > 0) {
          targetMode.value = 'FOCUS'
          focusedPhotoIndex.value = (focusedPhotoIndex.value + 1) % photos.value.length
          currentPhotoIndex.value = focusedPhotoIndex.value
          triggerPhotoAnimation(focusedPhotoIndex.value)
        }
      }
      lastTapTime = 0
    } else if (touchDuration < 200) {
      lastTapTime = now
    }
  }
}

const onResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
  composer.setSize(window.innerWidth, window.innerHeight)
}

const onKeyDown = (e) => {
  if (e.key.toLowerCase() === 'h') {
    uiHidden.value = !uiHidden.value
  }
  if (e.key === '1') {
    targetMode.value = 'TREE'
  }
  if (e.key === '2') {
    targetMode.value = 'SCATTER'
  }
  if (e.key === '3') {
    if (photos.value.length > 0) {
      targetMode.value = 'FOCUS'
      // 聚焦下一张照片
      focusedPhotoIndex.value = (focusedPhotoIndex.value + 1) % photos.value.length
      currentPhotoIndex.value = focusedPhotoIndex.value
      
      // 触发飞入动画
      triggerPhotoAnimation(focusedPhotoIndex.value)
    }
  }
  if (e.key.toLowerCase() === 'n') {
    nextPhoto()
  }
  // Arrow keys for zoom in FOCUS mode
  if (mode.value === 'FOCUS') {
    if (e.key === 'ArrowUp' || e.key === '+' || e.key === '=') {
      photoZoomScale = Math.min(15, photoZoomScale + 0.5)
    }
    if (e.key === 'ArrowDown' || e.key === '-') {
      photoZoomScale = Math.max(2, photoZoomScale - 0.5)
    }
  }
}

const onMouseMove = (e) => {
  mouseInteraction.x = (e.clientX / window.innerWidth - 0.5) * 0.5
  mouseInteraction.y = (e.clientY / window.innerHeight - 0.5) * 0.3
}

const onWheel = (e) => {
  if (mode.value === 'FOCUS') {
    const delta = e.deltaY > 0 ? -0.3 : 0.3
    photoZoomScale = Math.max(2, Math.min(15, photoZoomScale + delta))
  }
}

// Animation loop
const animate = () => {
  requestAnimationFrame(animate)
  const time = clock.getElapsedTime()

  // Mode transition
  if (mode.value !== targetMode.value) {
    mode.value = targetMode.value
  }

  // Update all elements
  updateParticles(time)
  updateStars(time)
  updateSnow(time)
  updateRibbons(time)
  updateTopStar(time)

  // Scene rotation
  if (handData) {
    mainGroup.rotation.y += (handData.rotationY - mainGroup.rotation.y) * 0.06
    mainGroup.rotation.x += (handData.rotationX - mainGroup.rotation.x) * 0.06
    starGroup.rotation.y = mainGroup.rotation.y * 0.3
    starGroup.rotation.x = mainGroup.rotation.x * 0.3
  } else {
    mainGroup.rotation.y += 0.002
    mainGroup.rotation.y += (mouseInteraction.x * 2.5 - mainGroup.rotation.y) * 0.025
    mainGroup.rotation.x += (mouseInteraction.y - mainGroup.rotation.x) * 0.025
    starGroup.rotation.y = mainGroup.rotation.y * 0.25
  }

  // Softer light animation
  const pulse1 = Math.sin(time * 2) * 0.3 + Math.sin(time * 3.5) * 0.2
  const pulse2 = Math.sin(time * 2.5) * 0.25 + Math.sin(time * 4) * 0.15
  pointLight.intensity = 3 + pulse1 * 1
  pointLight2.intensity = 2 + pulse2 * 0.8

  composer.render()
}

const updateParticles = (time) => {
  const currentMode = mode.value

  particles.forEach((particle) => {
    if (currentMode === 'TREE') {
      particle.updateTreePosition()
      particle.mesh.scale.lerp(particle.originalScale, 0.07)
      
      // 恢复照片可见性
      if (particle.type === 'PHOTO') {
        particle.mesh.visible = true
        if (particle.mesh.parent !== mainGroup) {
          photoGroup.remove(particle.mesh)
          mainGroup.add(particle.mesh)
        }
      }
    } else if (currentMode === 'SCATTER') {
      particle.updateScatterPosition()
      particle.mesh.scale.lerp(particle.originalScale, 0.07)

      // 恢复照片可见性
      if (particle.type === 'PHOTO') {
        particle.mesh.visible = true
        if (particle.mesh.parent !== mainGroup) {
          photoGroup.remove(particle.mesh)
          mainGroup.add(particle.mesh)
        }
      }
    } else if (currentMode === 'FOCUS') {
      // FOCUS模式：粒子散开，照片通过HTML覆盖层显示（不受Bloom影响）
      particle.updateScatterPosition()
      particle.mesh.scale.lerp(particle.originalScale, 0.07)
      
      // 所有照片粒子在FOCUS模式下隐藏（使用HTML显示）
      if (particle.type === 'PHOTO') {
        particle.mesh.visible = false
      }
    }
    particle.update(currentMode, time)
  })
}

const updateStars = (time) => {
  starParticles.forEach(star => star.update(time, mode.value))
}

const updateSnow = (time) => {
  snowParticles.forEach(snow => snow.update(time))
}

const updateRibbons = (time) => {
  // Only show ribbons in TREE mode
  const targetOpacity = mode.value === 'TREE' ? 0.8 : 0
  ribbons.forEach(ribbon => {
    ribbon.update(time)
    ribbon.mesh.material.opacity += (targetOpacity - ribbon.mesh.material.opacity) * 0.05
  })
}

const updateTopStar = (time) => {
  if (topStar) {
    // Gentle rotation
    topStar.rotation.z = time * 0.2
    
    // Softer pulsing glow
    const pulse = Math.sin(time * 2) * 0.3 + Math.sin(time * 3.5) * 0.2
    topStar.material.emissiveIntensity = 1.0 + pulse * 0.5
    
    // Subtle scale pulse
    const scalePulse = 1 + Math.sin(time * 1.5) * 0.05
    topStar.scale.setScalar(scalePulse)
  }
}

// Cleanup
const cleanup = () => {
  window.removeEventListener('resize', onResize)
  document.removeEventListener('keydown', onKeyDown)
  document.removeEventListener('mousemove', onMouseMove)
  document.removeEventListener('wheel', onWheel)
  document.removeEventListener('touchstart', onTouchStart)
  document.removeEventListener('touchmove', onTouchMove)
  document.removeEventListener('touchend', onTouchEnd)
  renderer?.dispose()
}

onMounted(init)
onUnmounted(cleanup)
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Cinzel:wght@400;700&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  background: #000000;
  overflow: hidden;
  font-family: 'Times New Roman', serif;
  color: #fceea7;
}

.app {
  width: 100%;
  height: 100vh;
  position: relative;
}

/* Aurora Background */
.aurora-bg {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(
    180deg,
    #000510 0%,
    #001020 20%,
    #000818 40%,
    #000510 60%,
    #000205 80%,
    #000000 100%
  );
  z-index: -2;
}

.aurora-bg::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 60%;
  background: 
    radial-gradient(ellipse 80% 50% at 20% 20%, rgba(0, 100, 80, 0.15) 0%, transparent 50%),
    radial-gradient(ellipse 60% 40% at 70% 30%, rgba(0, 50, 100, 0.1) 0%, transparent 50%),
    radial-gradient(ellipse 70% 50% at 50% 10%, rgba(100, 50, 150, 0.08) 0%, transparent 50%);
  animation: aurora 15s ease-in-out infinite;
}

@keyframes aurora {
  0%, 100% { opacity: 0.5; transform: translateX(0) scale(1); }
  50% { opacity: 0.8; transform: translateX(5%) scale(1.1); }
}

#loader {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: #000000;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  transition: opacity 1s ease-out;
}

#loader.hidden {
  opacity: 0;
  pointer-events: none;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 2px solid transparent;
  border-top: 2px solid #d4af37;
  border-right: 2px solid rgba(212, 175, 55, 0.5);
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loader-text {
  margin-top: 25px;
  font-family: 'Cinzel', serif;
  font-size: 14px;
  letter-spacing: 5px;
  color: #d4af37;
  animation: pulse 2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 0.5; }
  50% { opacity: 1; }
}

h1 {
  position: fixed;
  top: 25px;
  left: 50%;
  transform: translateX(-50%);
  font-family: 'Cinzel', serif;
  font-size: 60px;
  font-weight: 700;
  z-index: 100;
  transition: opacity 0.5s ease;
  text-align: center;
}

.title-main {
  background: linear-gradient(180deg, #ffffff 0%, #ffd700 50%, #d4af37 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  filter: drop-shadow(0 0 30px rgba(212, 175, 55, 0.8)) drop-shadow(0 0 60px rgba(255, 200, 50, 0.4));
  animation: titleGlow 3s ease-in-out infinite;
}

.title-sparkle {
  font-size: 40px;
  animation: sparkle 1.5s ease-in-out infinite;
  display: inline-block;
  margin-left: 10px;
}

@keyframes titleGlow {
  0%, 100% { filter: drop-shadow(0 0 30px rgba(212, 175, 55, 0.8)) drop-shadow(0 0 60px rgba(255, 200, 50, 0.4)); }
  50% { filter: drop-shadow(0 0 40px rgba(212, 175, 55, 1)) drop-shadow(0 0 80px rgba(255, 200, 50, 0.6)); }
}

@keyframes sparkle {
  0%, 100% { transform: scale(1) rotate(0deg); opacity: 1; }
  50% { transform: scale(1.3) rotate(180deg); opacity: 0.7; }
}

.upload-wrapper {
  position: fixed;
  bottom: 35px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
  z-index: 100;
  transition: opacity 0.5s ease;
}

.upload-btn {
  padding: 16px 30px;
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(15px);
  -webkit-backdrop-filter: blur(15px);
  border: 1px solid rgba(212, 175, 55, 0.6);
  border-radius: 35px;
  color: #d4af37;
  font-family: 'Cinzel', serif;
  font-size: 14px;
  letter-spacing: 2px;
  cursor: pointer;
  transition: all 0.4s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  min-width: 160px;
  text-align: center;
}

.upload-btn:hover {
  background: rgba(212, 175, 55, 0.25);
  box-shadow: 0 0 20px rgba(212, 175, 55, 0.4);
  border-color: #ffd700;
}

.btn-icon {
  font-size: 18px;
  animation: bounce 2s ease-in-out infinite;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-3px); }
}

.hint-text {
  font-size: 12px;
  color: rgba(252, 238, 167, 0.5);
  letter-spacing: 2px;
}

#fileInput {
  display: none;
}

.ui-hidden {
  opacity: 0 !important;
  pointer-events: none !important;
}

.mode-indicator {
  position: fixed;
  top: 110px;
  left: 50%;
  transform: translateX(-50%);
  font-family: 'Cinzel', serif;
  font-size: 16px;
  letter-spacing: 4px;
  color: #d4af37;
  opacity: 0.8;
  z-index: 100;
  transition: opacity 0.5s ease;
  text-shadow: 0 0 10px rgba(212, 175, 55, 0.5);
}

.photo-counter {
  position: fixed;
  top: 140px;
  left: 50%;
  transform: translateX(-50%);
  font-family: 'Cinzel', serif;
  font-size: 14px;
  letter-spacing: 2px;
  color: #fceea7;
  opacity: 0.6;
  z-index: 100;
  transition: opacity 0.5s ease;
}

.webcam-container {
  position: fixed;
  right: 20px;
  bottom: 120px;
  border-radius: 12px;
  overflow: hidden;
  border: 2px solid rgba(212, 175, 55, 0.4);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5);
}

.webcam-container video {
  width: 160px;
  height: 120px;
  transform: scaleX(-1);
  display: block;
}

.webcam-container canvas {
  display: none;
}

canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

/* Focused Photo Overlay - 居中显示，不受Bloom影响 */
.focused-photo-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 50;
  pointer-events: none;
  perspective: 1000px;
}

.focused-photo-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  transform-style: preserve-3d;
}

/* 动画1: 飞入动画 */
.focused-photo-container.fly-in {
  animation: flyInAnimation 1s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;
}

@keyframes flyInAnimation {
  0% {
    transform: translate(calc(var(--start-x) - 50vw), calc(var(--start-y) - 50vh)) scale(0.1) rotateY(180deg) rotateX(30deg);
    opacity: 0;
    filter: blur(15px);
  }
  40% {
    opacity: 1;
    filter: blur(3px);
  }
  70% {
    transform: translate(0, -15px) scale(1.1) rotateY(10deg) rotateX(-5deg);
    filter: blur(0);
  }
  100% {
    transform: translate(0, 0) scale(1) rotateY(0deg) rotateX(0deg);
    opacity: 1;
  }
}

/* 动画2: 螺旋进入 */
.focused-photo-container.spiral-in {
  animation: spiralInAnimation 1s cubic-bezier(0.25, 0.46, 0.45, 0.94) forwards;
}

@keyframes spiralInAnimation {
  0% {
    transform: translate(calc(var(--start-x) - 50vw), calc(var(--start-y) - 50vh)) scale(0) rotate(540deg);
    opacity: 0;
    filter: blur(10px);
  }
  50% {
    opacity: 1;
    filter: blur(2px);
  }
  75% {
    transform: translate(10px, -10px) scale(1.08) rotate(15deg);
    filter: blur(0);
  }
  100% {
    transform: translate(0, 0) scale(1) rotate(0deg);
    opacity: 1;
  }
}

/* 动画3: 翻转进入 */
.focused-photo-container.flip-in {
  animation: flipInAnimation 1s cubic-bezier(0.68, -0.55, 0.265, 1.55) forwards;
}

@keyframes flipInAnimation {
  0% {
    transform: perspective(1000px) rotateX(-90deg) rotateY(45deg) scale(0.3);
    opacity: 0;
    filter: blur(8px);
  }
  50% {
    transform: perspective(1000px) rotateX(15deg) rotateY(-8deg) scale(1.05);
    opacity: 1;
    filter: blur(0);
  }
  75% {
    transform: perspective(1000px) rotateX(-5deg) rotateY(3deg) scale(0.98);
  }
  100% {
    transform: perspective(1000px) rotateX(0deg) rotateY(0deg) scale(1);
    opacity: 1;
  }
}

/* 动画4: 弹跳进入 */
.focused-photo-container.bounce-in {
  animation: bounceInAnimation 1s cubic-bezier(0.36, 0.07, 0.19, 0.97) forwards;
}

@keyframes bounceInAnimation {
  0% {
    transform: translateY(-100vh) scale(0.5);
    opacity: 0;
  }
  40% {
    transform: translateY(20px) scale(1.1);
    opacity: 1;
  }
  60% {
    transform: translateY(-25px) scale(0.95);
  }
  80% {
    transform: translateY(8px) scale(1.02);
  }
  100% {
    transform: translateY(0) scale(1);
    opacity: 1;
  }
}

/* 动画5: 缩放模糊 */
.focused-photo-container.zoom-blur {
  animation: zoomBlurAnimation 1s cubic-bezier(0.4, 0, 0.2, 1) forwards;
}

@keyframes zoomBlurAnimation {
  0% {
    transform: scale(4);
    opacity: 0;
    filter: blur(30px) brightness(2);
  }
  50% {
    transform: scale(0.9);
    opacity: 1;
    filter: blur(3px) brightness(1.1);
  }
  75% {
    transform: scale(1.05);
    filter: blur(0) brightness(1);
  }
  100% {
    transform: scale(1);
    opacity: 1;
    filter: blur(0) brightness(1);
  }
}

/* 动画6: 滑动旋转 */
.focused-photo-container.slide-rotate {
  animation: slideRotateAnimation 1s cubic-bezier(0.25, 0.46, 0.45, 0.94) forwards;
}

@keyframes slideRotateAnimation {
  0% {
    transform: translateX(-100vw) rotate(-90deg) scale(0.5);
    opacity: 0;
    filter: blur(8px);
  }
  50% {
    transform: translateX(5vw) rotate(10deg) scale(1.05);
    opacity: 1;
    filter: blur(0);
  }
  75% {
    transform: translateX(-2vw) rotate(-3deg) scale(0.98);
  }
  100% {
    transform: translateX(0) rotate(0deg) scale(1);
    opacity: 1;
  }
}

.photo-frame {
  position: relative;
  padding: 12px;
  background: linear-gradient(145deg, #d4af37, #b8960c, #d4af37, #f0d050, #d4af37);
  border-radius: 8px;
  box-shadow: 
    0 0 40px rgba(212, 175, 55, 0.6),
    0 0 80px rgba(212, 175, 55, 0.3),
    0 20px 60px rgba(0, 0, 0, 0.5),
    inset 0 0 20px rgba(255, 255, 255, 0.3);
  transform-style: preserve-3d;
  overflow: hidden;
}

.photo-frame::before {
  content: '';
  position: absolute;
  inset: -2px;
  background: linear-gradient(45deg, 
    transparent 30%, 
    rgba(255, 255, 255, 0.3) 50%, 
    transparent 70%
  );
  animation: frameShine 3s ease-in-out infinite;
}

@keyframes frameShine {
  0%, 100% { transform: translateX(-100%) rotate(45deg); }
  50% { transform: translateX(100%) rotate(45deg); }
}

.photo-frame img {
  display: block;
  max-width: 65vw;
  max-height: 60vh;
  width: auto;
  height: auto;
  border-radius: 4px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5);
  position: relative;
  z-index: 1;
}

.photo-shine {
  position: absolute;
  top: 0;
  left: -100%;
  width: 50%;
  height: 100%;
  background: linear-gradient(
    90deg,
    transparent,
    rgba(255, 255, 255, 0.4),
    transparent
  );
  animation: photoShine 2s ease-in-out infinite;
  z-index: 2;
  pointer-events: none;
}

@keyframes photoShine {
  0% { left: -50%; }
  100% { left: 150%; }
}

.photo-info {
  margin-top: 20px;
  padding: 10px 25px;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(15px);
  border-radius: 25px;
  color: #d4af37;
  font-family: 'Cinzel', serif;
  font-size: 14px;
  letter-spacing: 3px;
  border: 1px solid rgba(212, 175, 55, 0.3);
  display: flex;
  align-items: center;
  gap: 10px;
}

.photo-icon {
  font-size: 16px;
  animation: iconPulse 1.5s ease-in-out infinite;
}

@keyframes iconPulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.2); }
}

/* 粒子爆炸效果 */
.particle-burst {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  pointer-events: none;
}

.burst-particle {
  position: absolute;
  width: 10px;
  height: 10px;
  background: radial-gradient(circle, #ffd700, #ff6b00);
  border-radius: 50%;
  box-shadow: 0 0 12px #ffd700, 0 0 25px #ff8c00;
  animation: burstOutFixed 0.8s cubic-bezier(0.25, 0.46, 0.45, 0.94) forwards;
}

.burst-particle:nth-child(1) { --tx: 180px; --ty: 0; --delay: 0s; }
.burst-particle:nth-child(2) { --tx: 165px; --ty: 70px; --delay: 0.02s; }
.burst-particle:nth-child(3) { --tx: 125px; --ty: 125px; --delay: 0.04s; }
.burst-particle:nth-child(4) { --tx: 70px; --ty: 165px; --delay: 0.06s; }
.burst-particle:nth-child(5) { --tx: 0; --ty: 180px; --delay: 0.08s; }
.burst-particle:nth-child(6) { --tx: -70px; --ty: 165px; --delay: 0.1s; }
.burst-particle:nth-child(7) { --tx: -125px; --ty: 125px; --delay: 0.12s; }
.burst-particle:nth-child(8) { --tx: -165px; --ty: 70px; --delay: 0.14s; }
.burst-particle:nth-child(9) { --tx: -180px; --ty: 0; --delay: 0.16s; }
.burst-particle:nth-child(10) { --tx: -165px; --ty: -70px; --delay: 0.18s; }
.burst-particle:nth-child(11) { --tx: -125px; --ty: -125px; --delay: 0.02s; }
.burst-particle:nth-child(12) { --tx: -70px; --ty: -165px; --delay: 0.04s; }
.burst-particle:nth-child(13) { --tx: 0; --ty: -180px; --delay: 0.06s; }
.burst-particle:nth-child(14) { --tx: 70px; --ty: -165px; --delay: 0.08s; }
.burst-particle:nth-child(15) { --tx: 125px; --ty: -125px; --delay: 0.1s; }
.burst-particle:nth-child(16) { --tx: 165px; --ty: -70px; --delay: 0.12s; }
.burst-particle:nth-child(17) { --tx: 220px; --ty: 40px; --delay: 0s; }
.burst-particle:nth-child(18) { --tx: -220px; --ty: -40px; --delay: 0.05s; }
.burst-particle:nth-child(19) { --tx: 40px; --ty: 220px; --delay: 0.1s; }
.burst-particle:nth-child(20) { --tx: -40px; --ty: -220px; --delay: 0.15s; }
.burst-particle:nth-child(21) { --tx: 240px; --ty: 0; --delay: 0s; }
.burst-particle:nth-child(22) { --tx: -240px; --ty: 0; --delay: 0.05s; }
.burst-particle:nth-child(23) { --tx: 0; --ty: 240px; --delay: 0.1s; }
.burst-particle:nth-child(24) { --tx: 0; --ty: -240px; --delay: 0.15s; }

.burst-particle {
  animation-delay: var(--delay, 0s);
}

@keyframes burstOutFixed {
  0% {
    transform: translate(-50%, -50%) scale(0);
    opacity: 1;
  }
  20% {
    transform: translate(-50%, -50%) scale(2);
    opacity: 1;
  }
  100% {
    transform: translate(calc(-50% + var(--tx)), calc(-50% + var(--ty))) scale(0);
    opacity: 0;
  }
}

/* 光环效果 */
.light-rings {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  pointer-events: none;
}

.ring {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  border: 3px solid rgba(212, 175, 55, 0.8);
  border-radius: 50%;
  animation: ringExpand 0.8s ease-out forwards;
}

.ring-1 {
  animation-delay: 0s;
}
.ring-2 {
  animation-delay: 0.1s;
}
.ring-3 {
  animation-delay: 0.2s;
}

@keyframes ringExpand {
  0% {
    width: 50px;
    height: 50px;
    opacity: 1;
    border-width: 3px;
  }
  100% {
    width: 500px;
    height: 500px;
    opacity: 0;
    border-width: 1px;
  }
}

.btn-group {
  display: flex;
  gap: 12px;
  align-items: center;
  justify-content: center;
}

.manage-btn {
  padding: 16px 30px;
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(15px);
  -webkit-backdrop-filter: blur(15px);
  border: 1px solid rgba(100, 200, 100, 0.5);
  border-radius: 35px;
  color: #88dd88;
  font-family: 'Cinzel', serif;
  font-size: 14px;
  letter-spacing: 2px;
  cursor: pointer;
  transition: all 0.4s ease;
  min-width: 160px;
  text-align: center;
}

.manage-btn:hover {
  background: rgba(100, 200, 100, 0.2);
  border-color: #88ff88;
  box-shadow: 0 0 20px rgba(100, 200, 100, 0.3);
}

.photo-manager {
  position: fixed;
  right: 20px;
  top: 50%;
  transform: translateY(-50%);
  width: 280px;
  max-height: 70vh;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(212, 175, 55, 0.4);
  border-radius: 20px;
  padding: 20px;
  z-index: 200;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.manager-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
  color: #d4af37;
  font-size: 14px;
  font-weight: bold;
}

.close-btn {
  background: none;
  border: none;
  color: #ff6666;
  font-size: 18px;
  cursor: pointer;
  padding: 5px;
  transition: transform 0.2s;
}

.close-btn:hover {
  transform: scale(1.2);
}

.photo-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
  overflow-y: auto;
  max-height: 50vh;
  padding-right: 5px;
}

.photo-grid::-webkit-scrollbar {
  width: 4px;
}

.photo-grid::-webkit-scrollbar-thumb {
  background: rgba(212, 175, 55, 0.5);
  border-radius: 2px;
}

.photo-item {
  position: relative;
  aspect-ratio: 1;
  border-radius: 10px;
  overflow: hidden;
  border: 2px solid rgba(212, 175, 55, 0.3);
  transition: all 0.3s ease;
}

.photo-item:hover {
  border-color: #d4af37;
  transform: scale(1.05);
}

.photo-item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.photo-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  opacity: 0;
  transition: opacity 0.3s;
  display: flex;
  justify-content: center;
  align-items: center;
}

.photo-item:hover .photo-overlay {
  opacity: 1;
}

.delete-btn {
  background: rgba(255, 50, 50, 0.8);
  border: none;
  border-radius: 50%;
  width: 36px;
  height: 36px;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.2s;
}

.delete-btn:hover {
  background: #ff3333;
  transform: scale(1.1);
}

.photo-num {
  position: absolute;
  bottom: 5px;
  right: 5px;
  background: rgba(0, 0, 0, 0.7);
  color: #fff;
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 10px;
}

.manager-footer {
  margin-top: 15px;
  padding-top: 15px;
  border-top: 1px solid rgba(212, 175, 55, 0.2);
}

.clear-all-btn {
  width: 100%;
  padding: 10px;
  background: rgba(255, 50, 50, 0.2);
  border: 1px solid rgba(255, 50, 50, 0.5);
  border-radius: 10px;
  color: #ff6666;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.3s;
}

.clear-all-btn:hover {
  background: rgba(255, 50, 50, 0.4);
}

.empty-hint {
  text-align: center;
  color: rgba(255, 255, 255, 0.5);
  padding: 20px;
  font-size: 12px;
}

.gesture-panel {
  position: fixed;
  left: 25px;
  bottom: 35px;
  z-index: 100;
  transition: opacity 0.5s ease;
  background: rgba(0, 0, 0, 0.5);
  padding: 15px 20px;
  border-radius: 15px;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(212, 175, 55, 0.3);
  min-width: 200px;
}

.gesture-status {
  font-size: 14px;
  color: #ff9999;
  margin-bottom: 12px;
  padding: 8px 12px;
  background: rgba(255, 100, 100, 0.1);
  border-radius: 8px;
  text-align: center;
  border: 1px solid rgba(255, 100, 100, 0.2);
  transition: all 0.3s ease;
}

.gesture-status.active {
  color: #99ff99;
  background: rgba(100, 255, 100, 0.1);
  border-color: rgba(100, 255, 100, 0.3);
}

.gesture-hints {
  font-size: 11px;
  color: rgba(252, 238, 167, 0.6);
  line-height: 2;
}

/* 手机端简洁提示 */
.mobile-hint {
  position: fixed;
  left: 50%;
  bottom: 100px;
  transform: translateX(-50%);
  padding: 8px 16px;
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  color: rgba(255, 255, 255, 0.7);
  font-size: 12px;
  z-index: 100;
  pointer-events: none;
  animation: fadeInOut 3s ease-in-out infinite;
}

@keyframes fadeInOut {
  0%, 100% { opacity: 0.5; }
  50% { opacity: 1; }
}
</style>
