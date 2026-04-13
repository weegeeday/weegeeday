<script setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import popupImage from './assets/BCA.svg'
import soundFile from './assets/sound.mp3'
import sound2File from './assets/sound2.mp3'
import chickenImage1 from './assets/chicken/Layer 1.png'
import chickenImage2 from './assets/chicken/Layer 2.png'

const popupIntervalMs = 5000
const linearFrictionPerSecond = 0.08
const angularFrictionPerSecond = 0.55

const isPopupVisible = ref(false)
const popupCycleKey = ref(0)
const viewportWidth = ref(window.innerWidth)
const viewportHeight = ref(window.innerHeight)
const chickens = ref([])
const areCollisionsEnabled = ref(true)
const isPerformanceMode = ref(false)
const isMenuOpen = ref(false)
const chickensPerPopup = ref(1)
const upgradeCost = ref(2)
const chickenCap = ref(300)
const isUnlimitedCap = ref(false)
const useRenderLimit = ref(false)
const renderLimit = ref(120)
const fpsCounter = ref(0)
const totalChickenCount = ref(0)

let popupTimerId = null
let animationFrameId = null
let previousTimestamp = 0
let chickenIdSeed = 0
let activeDragId = null
let lowFpsDurationMs = 0
let smoothedFps = 60

const popupAudio = new Audio(soundFile)
const chickenAudio = new Audio(sound2File)
const chickenImages = [chickenImage1, chickenImage2]

popupAudio.preload = 'auto'
chickenAudio.preload = 'auto'

const chickenCount = computed(() => totalChickenCount.value)
const canAffordUpgrade = computed(() => chickenCount.value >= upgradeCost.value)
const effectiveRenderLimit = computed(() => {
  if (!useRenderLimit.value) {
    return totalChickenCount.value
  }

  return Math.min(totalChickenCount.value, normalizeRenderLimit(renderLimit.value))
})

const randomBetween = (minimum, maximum) => {
  return minimum + Math.random() * (maximum - minimum)
}

const capSpeed = (value, maxMagnitude) => {
  if (Math.abs(value) <= maxMagnitude) {
    return value
  }

  return value > 0 ? maxMagnitude : -maxMagnitude
}

const getChickenSize = () => {
  const baseSize = Math.min(viewportWidth.value, viewportHeight.value) * 0.12
  return Math.max(56, Math.min(128, Math.round(baseSize)))
}

const normalizeCapValue = (value) => {
  const safeValue = Number.isFinite(value) ? value : 300
  return Math.max(25, Math.min(2000, Math.round(safeValue)))
}

const normalizeRenderLimit = (value) => {
  const safeValue = Number.isFinite(value) ? value : 120
  return Math.max(10, Math.min(1000, Math.round(safeValue)))
}

const enforceChickenCap = () => {
  if (isUnlimitedCap.value) {
    return
  }

  if (totalChickenCount.value > chickenCap.value) {
    totalChickenCount.value = chickenCap.value
  }
}

const clampPosition = (chicken) => {
  const maxX = Math.max(0, viewportWidth.value - chicken.size)
  const maxY = Math.max(0, viewportHeight.value - chicken.size)
  chicken.x = Math.max(0, Math.min(chicken.x, maxX))
  chicken.y = Math.max(0, Math.min(chicken.y, maxY))
}

const playPopupAudio = async () => {
  try {
    popupAudio.muted = false
    popupAudio.currentTime = 0
    await popupAudio.play()
  } catch (error) {
    void error
  }
}

const playChickenAudio = async () => {
  try {
    chickenAudio.currentTime = 0
    await chickenAudio.play()
  } catch (error) {
    void error
  }
}

const updateViewport = () => {
  viewportWidth.value = window.innerWidth
  viewportHeight.value = window.innerHeight
  chickens.value.forEach((chicken) => {
    clampPosition(chicken)
  })
}

const spawnPopup = () => {
  isPopupVisible.value = true
  popupCycleKey.value += 1
  void playPopupAudio()
}

const createChicken = () => {
  const size = getChickenSize()
  const maxX = Math.max(0, viewportWidth.value - size)
  const maxY = Math.max(0, viewportHeight.value - size)

  return {
    id: ++chickenIdSeed,
    src: chickenImages[Math.floor(Math.random() * chickenImages.length)],
    size,
    x: randomBetween(0, maxX),
    y: randomBetween(0, maxY),
    velocityX: randomBetween(-240, 240) || 160,
    velocityY: randomBetween(-240, 240) || -160,
    dragOffsetX: 0,
    dragOffsetY: 0,
    pointerLastX: 0,
    pointerLastY: 0,
    pointerLastTime: 0,
    dragVelocityX: 0,
    dragVelocityY: 0,
    dragAngularVelocity: 0,
    angle: randomBetween(0, 360),
    angularVelocity: randomBetween(-180, 180),
    movedWhileDragging: false,
    isDragging: false,
  }
}

const syncRenderedChickens = () => {
  const targetCount = effectiveRenderLimit.value

  while (chickens.value.length > targetCount) {
    chickens.value.shift()
  }

  while (chickens.value.length < targetCount) {
    chickens.value.push(createChicken())
  }
}

const handlePopupClick = () => {
  void playChickenAudio()

  const requested = chickensPerPopup.value
  const allowedByCap = isUnlimitedCap.value
    ? requested
    : Math.max(0, chickenCap.value - totalChickenCount.value)
  const toAdd = Math.min(requested, allowedByCap)

  if (toAdd > 0) {
    totalChickenCount.value += toAdd
    syncRenderedChickens()
  }

  isPopupVisible.value = false

  if (popupTimerId !== null) {
    window.clearTimeout(popupTimerId)
  }

  popupTimerId = window.setTimeout(spawnPopup, popupIntervalMs)
}

const toggleMenu = () => {
  isMenuOpen.value = !isMenuOpen.value
}

const upgradeChickenSpawn = () => {
  if (!canAffordUpgrade.value) {
    return
  }

  totalChickenCount.value -= upgradeCost.value
  syncRenderedChickens()
  chickensPerPopup.value *= 2
  upgradeCost.value = Math.max(1, Math.ceil(upgradeCost.value * 1.3))
}

const applyChickenCap = () => {
  chickenCap.value = normalizeCapValue(chickenCap.value)
  enforceChickenCap()
  syncRenderedChickens()
}

const toggleUnlimitedCap = () => {
  if (!isUnlimitedCap.value) {
    applyChickenCap()
  }
}

const applyRenderLimit = () => {
  renderLimit.value = normalizeRenderLimit(renderLimit.value)
  syncRenderedChickens()
}

const findChickenById = (id) => {
  return chickens.value.find((chicken) => chicken.id === id) || null
}

const handleChickenPointerDown = (chicken, event) => {
  activeDragId = chicken.id
  chicken.isDragging = true
  chicken.movedWhileDragging = false
  chicken.dragOffsetX = event.clientX - chicken.x
  chicken.dragOffsetY = event.clientY - chicken.y
  chicken.pointerLastX = event.clientX
  chicken.pointerLastY = event.clientY
  chicken.pointerLastTime = event.timeStamp || performance.now()
  chicken.dragVelocityX = 0
  chicken.dragVelocityY = 0
  chicken.dragAngularVelocity = 0

  if (event.target && typeof event.target.setPointerCapture === 'function') {
    event.target.setPointerCapture(event.pointerId)
  }
}

const handlePointerMove = (event) => {
  if (activeDragId === null) {
    return
  }

  const chicken = findChickenById(activeDragId)
  if (!chicken) {
    activeDragId = null
    return
  }

  const nextX = event.clientX - chicken.dragOffsetX
  const nextY = event.clientY - chicken.dragOffsetY
  const movedDistance = Math.abs(nextX - chicken.x) + Math.abs(nextY - chicken.y)
  const eventTime = event.timeStamp || performance.now()
  const elapsedSeconds = Math.max(0.001, (eventTime - chicken.pointerLastTime) / 1000)
  const pointerDeltaX = event.clientX - chicken.pointerLastX
  const pointerDeltaY = event.clientY - chicken.pointerLastY

  const sampledVelocityX = pointerDeltaX / elapsedSeconds
  const sampledVelocityY = pointerDeltaY / elapsedSeconds
  const sampledAngularVelocity = ((pointerDeltaX - pointerDeltaY) / elapsedSeconds) * 0.08

  chicken.dragVelocityX = chicken.dragVelocityX * 0.35 + sampledVelocityX * 0.65
  chicken.dragVelocityY = chicken.dragVelocityY * 0.35 + sampledVelocityY * 0.65
  chicken.dragAngularVelocity = chicken.dragAngularVelocity * 0.45 + sampledAngularVelocity * 0.55

  chicken.pointerLastX = event.clientX
  chicken.pointerLastY = event.clientY
  chicken.pointerLastTime = eventTime

  if (movedDistance > 3) {
    chicken.movedWhileDragging = true
  }

  chicken.x = nextX
  chicken.y = nextY
  clampPosition(chicken)
}

const handlePointerUp = (event) => {
  if (activeDragId === null) {
    return
  }

  const chicken = findChickenById(activeDragId)
  if (chicken) {
    const releaseTime = event?.timeStamp || performance.now()
    const elapsedSeconds = Math.max(0.001, (releaseTime - chicken.pointerLastTime) / 1000)
    const releaseDeltaX = (event?.clientX ?? chicken.pointerLastX) - chicken.pointerLastX
    const releaseDeltaY = (event?.clientY ?? chicken.pointerLastY) - chicken.pointerLastY

    if (releaseDeltaX !== 0 || releaseDeltaY !== 0) {
      const releaseVelocityX = releaseDeltaX / elapsedSeconds
      const releaseVelocityY = releaseDeltaY / elapsedSeconds
      const releaseAngularVelocity = ((releaseDeltaX - releaseDeltaY) / elapsedSeconds) * 0.08

      chicken.dragVelocityX = chicken.dragVelocityX * 0.5 + releaseVelocityX * 0.5
      chicken.dragVelocityY = chicken.dragVelocityY * 0.5 + releaseVelocityY * 0.5
      chicken.dragAngularVelocity = chicken.dragAngularVelocity * 0.5 + releaseAngularVelocity * 0.5
    }

    chicken.isDragging = false

    if (chicken.movedWhileDragging) {
      chicken.velocityX = capSpeed(chicken.dragVelocityX, 420)
      chicken.velocityY = capSpeed(chicken.dragVelocityY, 420)
      chicken.angularVelocity = capSpeed(chicken.dragAngularVelocity, 540)
    } else {
      if (Math.abs(chicken.velocityX) < 60) {
        chicken.velocityX = chicken.velocityX >= 0 ? 120 : -120
      }
      if (Math.abs(chicken.velocityY) < 60) {
        chicken.velocityY = chicken.velocityY >= 0 ? 120 : -120
      }
      if (Math.abs(chicken.angularVelocity) < 30) {
        chicken.angularVelocity = chicken.angularVelocity >= 0 ? 80 : -80
      }
    }

    chicken.pointerLastTime = releaseTime
  }

  activeDragId = null
}

const handleChickenClick = async (chicken) => {
  if (chicken.movedWhileDragging) {
    chicken.movedWhileDragging = false
    return
  }

  await playChickenAudio()
}

const resolveChickenCollisions = () => {
  const chickenList = chickens.value

  for (let firstIndex = 0; firstIndex < chickenList.length; firstIndex += 1) {
    const firstChicken = chickenList[firstIndex]
    if (firstChicken.isDragging) {
      continue
    }

    for (let secondIndex = firstIndex + 1; secondIndex < chickenList.length; secondIndex += 1) {
      const secondChicken = chickenList[secondIndex]
      if (secondChicken.isDragging) {
        continue
      }

      const firstCenterX = firstChicken.x + firstChicken.size / 2
      const firstCenterY = firstChicken.y + firstChicken.size / 2
      const secondCenterX = secondChicken.x + secondChicken.size / 2
      const secondCenterY = secondChicken.y + secondChicken.size / 2

      let deltaX = secondCenterX - firstCenterX
      let deltaY = secondCenterY - firstCenterY
      let distanceSquared = deltaX * deltaX + deltaY * deltaY

      if (distanceSquared === 0) {
        deltaX = randomBetween(-1, 1)
        deltaY = randomBetween(-1, 1)
        distanceSquared = deltaX * deltaX + deltaY * deltaY
      }

      const firstRadius = firstChicken.size * 0.45
      const secondRadius = secondChicken.size * 0.45
      const minimumDistance = firstRadius + secondRadius

      if (distanceSquared >= minimumDistance * minimumDistance) {
        continue
      }

      const distance = Math.sqrt(distanceSquared)
      const normalX = deltaX / distance
      const normalY = deltaY / distance
      const overlap = minimumDistance - distance

      firstChicken.x -= normalX * overlap * 0.5
      firstChicken.y -= normalY * overlap * 0.5
      secondChicken.x += normalX * overlap * 0.5
      secondChicken.y += normalY * overlap * 0.5

      clampPosition(firstChicken)
      clampPosition(secondChicken)

      const relativeVelocityX = secondChicken.velocityX - firstChicken.velocityX
      const relativeVelocityY = secondChicken.velocityY - firstChicken.velocityY
      const velocityAlongNormal = relativeVelocityX * normalX + relativeVelocityY * normalY

      if (velocityAlongNormal >= 0) {
        continue
      }

      const bounceFactor = 0.95
      const impulse = (-(1 + bounceFactor) * velocityAlongNormal) / 2

      firstChicken.velocityX -= impulse * normalX
      firstChicken.velocityY -= impulse * normalY
      secondChicken.velocityX += impulse * normalX
      secondChicken.velocityY += impulse * normalY

      firstChicken.velocityX = capSpeed(firstChicken.velocityX, 320)
      firstChicken.velocityY = capSpeed(firstChicken.velocityY, 320)
      secondChicken.velocityX = capSpeed(secondChicken.velocityX, 320)
      secondChicken.velocityY = capSpeed(secondChicken.velocityY, 320)

      firstChicken.angularVelocity = capSpeed(firstChicken.angularVelocity - impulse * 0.08, 540)
      secondChicken.angularVelocity = capSpeed(secondChicken.angularVelocity + impulse * 0.08, 540)
    }
  }
}

const animateChickens = (timestamp) => {
  if (previousTimestamp === 0) {
    previousTimestamp = timestamp
  }

  const frameMs = Math.max(1, timestamp - previousTimestamp)
  const deltaSeconds = Math.min(0.032, frameMs / 1000)
  previousTimestamp = timestamp
  const instantaneousFps = 1000 / frameMs

  smoothedFps = smoothedFps * 0.9 + instantaneousFps * 0.1
  fpsCounter.value = Math.round(smoothedFps)
  const linearDamping = Math.exp(-linearFrictionPerSecond * deltaSeconds)
  const angularDamping = Math.exp(-angularFrictionPerSecond * deltaSeconds)

  if (!document.hidden && areCollisionsEnabled.value) {
    if (instantaneousFps < 26) {
      lowFpsDurationMs += frameMs
    } else {
      lowFpsDurationMs = Math.max(0, lowFpsDurationMs - frameMs)
    }

    if (lowFpsDurationMs >= 2000 && chickens.value.length >= 8) {
      areCollisionsEnabled.value = false
      isPerformanceMode.value = true
    }
  }

  chickens.value.forEach((chicken) => {
    if (chicken.isDragging) {
      return
    }

    chicken.velocityX *= linearDamping
    chicken.velocityY *= linearDamping
    chicken.angularVelocity *= angularDamping

    if (Math.abs(chicken.velocityX) < 0.5) {
      chicken.velocityX = 0
    }
    if (Math.abs(chicken.velocityY) < 0.5) {
      chicken.velocityY = 0
    }
    if (Math.abs(chicken.angularVelocity) < 1.2) {
      chicken.angularVelocity = 0
    }

    chicken.x += chicken.velocityX * deltaSeconds
    chicken.y += chicken.velocityY * deltaSeconds
    chicken.angle = (chicken.angle + chicken.angularVelocity * deltaSeconds) % 360

    const maxX = Math.max(0, viewportWidth.value - chicken.size)
    const maxY = Math.max(0, viewportHeight.value - chicken.size)

    if (chicken.x <= 0) {
      chicken.x = 0
      chicken.velocityX = Math.abs(chicken.velocityX)
    } else if (chicken.x >= maxX) {
      chicken.x = maxX
      chicken.velocityX = -Math.abs(chicken.velocityX)
    }

    if (chicken.y <= 0) {
      chicken.y = 0
      chicken.velocityY = Math.abs(chicken.velocityY)
    } else if (chicken.y >= maxY) {
      chicken.y = maxY
      chicken.velocityY = -Math.abs(chicken.velocityY)
    }
  })

  if (areCollisionsEnabled.value) {
    resolveChickenCollisions()
  }

  animationFrameId = window.requestAnimationFrame(animateChickens)
}

onMounted(() => {
  document.title = 'Barbaque Chicken Alert'
  updateViewport()
  syncRenderedChickens()

  popupTimerId = window.setTimeout(spawnPopup, popupIntervalMs)

  window.addEventListener('resize', updateViewport)
  window.addEventListener('pointermove', handlePointerMove)
  window.addEventListener('pointerup', handlePointerUp)
  window.addEventListener('pointercancel', handlePointerUp)

  animationFrameId = window.requestAnimationFrame(animateChickens)
})

onUnmounted(() => {
  if (popupTimerId !== null) {
    window.clearTimeout(popupTimerId)
  }
  if (animationFrameId !== null) {
    window.cancelAnimationFrame(animationFrameId)
  }

  window.removeEventListener('resize', updateViewport)
  window.removeEventListener('pointermove', handlePointerMove)
  window.removeEventListener('pointerup', handlePointerUp)
  window.removeEventListener('pointercancel', handlePointerUp)

  popupAudio.pause()
  popupAudio.currentTime = 0
  chickenAudio.pause()
  chickenAudio.currentTime = 0
})

watch(useRenderLimit, () => {
  syncRenderedChickens()
})
</script>

<template>
  <div class="black-screen">
    <div class="fps-hud">FPS: {{ fpsCounter }}</div>

    <div class="hud">
      <button class="menu-button" type="button" aria-label="Open menu" @click="toggleMenu">☰</button>
      <div class="counter">Chicken: {{ chickenCount }}</div>
    </div>

    <div v-if="isMenuOpen" class="menu-panel">
      <div class="menu-title">Upgrades</div>
      <button
        type="button"
        class="upgrade-button"
        :disabled="!canAffordUpgrade"
        @click="upgradeChickenSpawn"
      >
        Double chickens per popup (x{{ chickensPerPopup }}) — Cost: {{ upgradeCost }}
      </button>

      <label class="menu-label" for="chicken-cap-input">Chicken cap</label>
      <input
        id="chicken-cap-input"
        v-model.number="chickenCap"
        class="menu-input"
        type="number"
        min="25"
        max="2000"
        step="5"
        :disabled="isUnlimitedCap"
        @change="applyChickenCap"
      >

      <label class="menu-toggle">
        <input
          v-model="isUnlimitedCap"
          type="checkbox"
          @change="toggleUnlimitedCap"
        >
        Unlimited cap
      </label>

      <label class="menu-toggle">
        <input v-model="useRenderLimit" type="checkbox">
        Limit rendered chickens
      </label>

      <label class="menu-label" for="render-limit-input">Rendered chicken limit</label>
      <input
        id="render-limit-input"
        v-model.number="renderLimit"
        class="menu-input"
        type="number"
        min="10"
        max="1000"
        step="5"
        :disabled="!useRenderLimit"
        @change="applyRenderLimit"
      >
    </div>

    <div v-if="isPerformanceMode" class="performance-notice">
      Collision physics disabled for performance.
    </div>

    <img
      v-for="chicken in chickens"
      :key="chicken.id"
      class="chicken"
      :src="chicken.src"
      alt="Barbecue chicken"
      :style="{
        left: `${chicken.x}px`,
        top: `${chicken.y}px`,
        width: `${chicken.size}px`,
        height: `${chicken.size}px`,
        transform: `rotate(${chicken.angle}deg)`,
      }"
      @pointerdown="handleChickenPointerDown(chicken, $event)"
      @click="handleChickenClick(chicken)"
      draggable="false"
    />

    <img
      v-if="isPopupVisible"
      :key="popupCycleKey"
      class="popup-image"
      :src="popupImage"
      alt="BCA popup"
      @click="handlePopupClick"
    />
  </div>
</template>

<style scoped>
.black-screen {
  position: relative;
  min-height: 100vh;
  width: 100%;
  overflow: hidden;
  background: #000;
}

.hud {
  position: fixed;
  top: 1rem;
  right: 1rem;
  z-index: 50;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.fps-hud {
  position: fixed;
  top: 1rem;
  left: 1rem;
  z-index: 50;
  border: 1px solid #2c2c2c;
  background: rgba(15, 15, 15, 0.92);
  color: #f1f1f1;
  border-radius: 999px;
  padding: 0.35rem 0.75rem;
  font-size: 0.9rem;
}

.menu-button,
.counter {
  border: 1px solid #2c2c2c;
  background: rgba(15, 15, 15, 0.92);
  color: #f1f1f1;
  border-radius: 999px;
  padding: 0.35rem 0.75rem;
  font-size: 0.9rem;
}

.menu-button {
  cursor: pointer;
}

.menu-panel {
  position: fixed;
  top: 3.8rem;
  right: 1rem;
  z-index: 55;
  width: min(320px, 88vw);
  border: 1px solid #2c2c2c;
  background: rgba(15, 15, 15, 0.96);
  border-radius: 0.9rem;
  padding: 0.75rem;
  display: grid;
  gap: 0.55rem;
}

.menu-title {
  color: #f1f1f1;
  font-size: 0.85rem;
}

.menu-label {
  color: #cfcfcf;
  font-size: 0.78rem;
}

.menu-input {
  border: 1px solid #4a4a4a;
  background: rgba(20, 20, 20, 0.95);
  color: #f7f7f7;
  border-radius: 0.6rem;
  padding: 0.45rem 0.55rem;
  font-size: 0.82rem;
}

.menu-toggle {
  display: flex;
  align-items: center;
  gap: 0.45rem;
  color: #d8d8d8;
  font-size: 0.8rem;
}

.upgrade-button {
  border: 1px solid #4a4a4a;
  background: rgba(32, 32, 32, 0.95);
  color: #f7f7f7;
  border-radius: 0.7rem;
  padding: 0.5rem 0.65rem;
  font-size: 0.82rem;
  text-align: left;
  cursor: pointer;
}

.upgrade-button:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.performance-notice {
  position: fixed;
  top: 3.75rem;
  right: 1rem;
  z-index: 50;
  border: 1px solid #634400;
  background: rgba(56, 36, 0, 0.9);
  color: #ffd88a;
  border-radius: 0.7rem;
  padding: 0.45rem 0.7rem;
  font-size: 0.78rem;
}

.chicken {
  position: fixed;
  z-index: 20;
  user-select: none;
  -webkit-user-drag: none;
  touch-action: none;
  cursor: grab;
}

.chicken:active {
  cursor: grabbing;
}

.popup-image {
  position: fixed;
  z-index: 40;
  right: clamp(1rem, 3vw, 2rem);
  bottom: clamp(1rem, 3vw, 2rem);
  width: clamp(220px, 23vw, 480px);
  max-width: 30vw;
  height: auto;
  cursor: pointer;
  animation: popup-in 220ms ease-out;
}

@media (max-width: 767px) {
  .popup-image {
    left: 50%;
    right: auto;
    transform: translateX(-50%);
    width: clamp(180px, 72vw, 420px);
    max-width: 86vw;
  }

  .hud {
    top: 0.75rem;
    right: 0.75rem;
  }

  .fps-hud {
    top: 0.75rem;
    left: 0.75rem;
  }

  .menu-panel {
    top: 3.45rem;
    right: 0.75rem;
  }

  .performance-notice {
    top: 3.4rem;
    right: 0.75rem;
    max-width: 70vw;
  }
}

@keyframes popup-in {
  from {
    opacity: 0;
    scale: 0.95;
  }

  to {
    opacity: 1;
    scale: 1;
  }
}
</style>
