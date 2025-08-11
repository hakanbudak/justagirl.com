<template>
  <div class="w-full h-screen bg-black flex flex-col items-center justify-center text-white relative">
    <canvas
        ref="gameCanvas"
        :width="canvasWidth"
        :height="canvasHeight"
        class="w-full max-w-[640px] h-auto aspect-video border border-white"
    />

    <canvas
        ref="fireworksCanvas"
        :width="canvasWidth"
        :height="canvasHeight"
        class="absolute z-20 pointer-events-none"
    ></canvas>

    <div class="absolute top-4 left-4 text-xl font-bold bg-pink-500 px-4 py-2 rounded">
      {{ currentDate }}
    </div>

    <div v-if="gameOver" class="absolute inset-0 bg-black bg-opacity-80 flex flex-col items-center justify-center text-center">
      <h1 class="text-4xl text-pink-400 mb-4">🎆 29 Temmuz 🎆</h1>
      <p class="text-xl">Tanıştığımız Gün!<br />Seni çok seviyorum 💖</p>
    </div>
  </div>
</template>


<script setup>
import {ref, onMounted, onBeforeUnmount, computed} from 'vue'
import heartImgUrl from '../assets/heart.png'
import boyImgUrl from '../assets/boyImgUrl.png'
import girlImgUrl from '../assets/endgirl.png'
import birdImgUrl from '../assets/birds.png'

const canvasWidth = window.innerWidth < 768 ? window.innerWidth : 640
const canvasHeight = window.innerHeight < 768 ? window.innerHeight - 100 : 480
const heartCount = 8
const showBoy = ref(false)
const hearts = ref([])
const collectedCount = ref(0)
const gameOver = ref(false)
const gameCanvas = ref(null)
const fireworksCanvas = ref(null)
const meteors = []


const heartImage = new Image()
const boyImage = new Image()
const girlImage = new Image()
const birdImage = new Image()
girlImage.src = girlImgUrl
boyImage.src = boyImgUrl
heartImage.src = heartImgUrl
birdImage.src = birdImgUrl
const currentDate = computed(() => {
  const startDay = 29
  const startMonth = 6
  const date = new Date(2024, startMonth, startDay + collectedCount.value)
  return date.toLocaleDateString('tr-TR', { day: 'numeric', month: 'long' })
})

let ctx
let animationFrame
let fireworksCtx
let bgAudio
let touchStartX = 0
let touchStartY = 0
let meteorAnimId

const prison = {
  x: canvasWidth - 1000,
  y: canvasHeight - 1000,
  size: 60,
}

const player = {
  x: 50,
  y: 50,
  size: 90,
  speed: 5,
  dx: 0,
  dy: 0,
}

function handleTouchStart(e) {
  const touch = e.touches[0]
  touchStartX = touch.clientX
  touchStartY = touch.clientY
}

function handleTouchMove(e) {
  if (!e.changedTouches) return
  const touch = e.changedTouches[0]
  const dx = touch.clientX - touchStartX
  const dy = touch.clientY - touchStartY

  const absDx = Math.abs(dx)
  const absDy = Math.abs(dy)

  if (absDx > absDy) {
    // Yatay kaydırma
    player.dx = dx > 0 ? player.speed : -player.speed
    player.dy = 0
  } else {
    // Dikey kaydırma
    player.dy = dy > 0 ? player.speed : -player.speed
    player.dx = 0
  }
}

function handleTouchEnd() {
  player.dx = 0
  player.dy = 0
}

function playFinalMusic() {
  bgAudio = new Audio('../audio/belki.mp3')
  bgAudio.volume = 0.7
  bgAudio.play()
}

function drawPlayer() {
  const grayscaleLevel = Math.max(0, 100 - (collectedCount.value / heartCount) * 100)

  ctx.filter = `grayscale(${grayscaleLevel}%)`
  ctx.drawImage(girlImage, player.x, player.y, player.size, player.size)
  ctx.filter = 'none'
}

function drawBoy() {
  if (showBoy.value) {
    ctx.drawImage(boyImage, prison.x + 5, prison.y + 5, 30, 30, prison.size)
  }
}


function spawnHearts() {
  for (let i = 0; i < heartCount; i++) {
    hearts.value.push({
      x: Math.floor(Math.random() * (canvasWidth - 20)),
      y: Math.floor(Math.random() * (canvasHeight - 20)),
      size: 24,
      collected: i !== 0,
    })
  }
}


function checkCollision(a, b) {
  return (
      a.x < b.x + b.size &&
      a.x + a.size > b.x &&
      a.y < b.y + b.size &&
      a.y + a.size > b.y
  )
}

function drawPrison() {
  ctx.fillStyle = 'gray'
  ctx.fillRect(prison.x, prison.y, prison.size, prison.size)

  ctx.strokeStyle = 'black'
  for (let i = 1; i < 4; i++) {
    const gap = prison.size / 4
    ctx.beginPath()
    ctx.moveTo(prison.x + i * gap, prison.y)
    ctx.lineTo(prison.x + i * gap, prison.y + prison.size)
    ctx.stroke()
  }
}

function drawFinalScene() {
  fireworksCtx.clearRect(0, 0, canvasWidth, canvasHeight)

  fireworksCtx.fillStyle = 'black'
  fireworksCtx.fillRect(0, 0, canvasWidth, canvasHeight)

  fireworksCtx.drawImage(girlImage, 190, canvasHeight - 120, 94, 94)

  fireworksCtx.drawImage(boyImage, 120, canvasHeight - 120, 94, 94)


  fireworksCtx.fillStyle = 'white'
  fireworksCtx.font = 'bold 20px sans-serif'
  fireworksCtx.textAlign = 'center'

  const lines = [
    "Bugün sadece tanışmadık.",
    "Bugün, ‘olsun’ dediğimiz her şeyin başladığı gün. 💖"
  ]

  lines.forEach((line, i) => {
    fireworksCtx.fillText(line, canvasWidth / 2, 50 + i * 30)
  })

  fireworksCtx.font = 'italic 14px sans-serif'
  fireworksCtx.fillStyle = 'rgba(255,255,255,0.6)'
  fireworksCtx.fillText("— 29 Temmuz 2025", canvasWidth / 2, 120)
}

function createMeteor() {
  // Ekranın sol/üst kenarlarından 45° açıyla diyagonal kayan bir meteor
  const fromTop = Math.random() < 0.5
  const startX = fromTop ? Math.random() * canvasWidth * 0.7 : -50
  const startY = fromTop ? -50 : Math.random() * canvasHeight * 0.3

  const angle = Math.PI / 4 + (Math.random() - 0.5) * 0.2 // ~45°
  const speed = 6 + Math.random() * 4
  const length = 60 + Math.random() * 80

  return {
    x: startX,
    y: startY,
    vx: Math.cos(angle) * speed,
    vy: Math.sin(angle) * speed,
    length,
    life: 1,                // 0→yok ol
    decay: 0.003 + Math.random() * 0.004
  }
}

function spawnMeteors(count = 10) {
  meteors.length = 0
  for (let i = 0; i < count; i++) {
    meteors.push(createMeteor())
  }
}

function drawMeteor(m) {
  // Kuyruk
  const tailX = m.x - m.vx * (m.length / 10)
  const tailY = m.y - m.vy * (m.length / 10)

  fireworksCtx.save()
  fireworksCtx.globalCompositeOperation = 'lighter'
  fireworksCtx.globalAlpha = Math.max(0, m.life)

  // Kuyruk çizgisi
  fireworksCtx.beginPath()
  fireworksCtx.moveTo(m.x, m.y)
  fireworksCtx.lineTo(tailX, tailY)
  fireworksCtx.lineWidth = 2
  fireworksCtx.strokeStyle = 'rgba(255,255,255,0.9)'
  fireworksCtx.stroke()

  // Baş kısım (parlayan)
  const grd = fireworksCtx.createRadialGradient(m.x, m.y, 0, m.x, m.y, 6)
  grd.addColorStop(0, 'rgba(255,255,255,1)')
  grd.addColorStop(1, 'rgba(255,255,255,0)')
  fireworksCtx.fillStyle = grd
  fireworksCtx.beginPath()
  fireworksCtx.arc(m.x, m.y, 6, 0, Math.PI * 2)
  fireworksCtx.fill()

  fireworksCtx.restore()
}

function animateMeteorShower() {
  // Final sahne arkaplanı + metin (her karede tazeliyoruz)
  drawFinalScene()

  // Hafif motion blur / iz efekti
  fireworksCtx.fillStyle = 'rgba(0,0,0,0.20)'
  fireworksCtx.fillRect(0, 0, canvasWidth, canvasHeight)

  // Meteorları güncelle/çiz
  for (let i = 0; i < meteors.length; i++) {
    const m = meteors[i]
    m.x += m.vx
    m.y += m.vy
    m.life -= m.decay

    drawMeteor(m)

    // Ekranı terk ettiyse veya söndüyse yenile
    if (
        m.life <= 0 ||
        m.x > canvasWidth + 100 ||
        m.y > canvasHeight + 100
    ) {
      meteors[i] = createMeteor()
    }
  }

  meteorAnimId = requestAnimationFrame(animateMeteorShower)
}

function startMeteorShower() {
  cancelAnimationFrame(meteorAnimId)
  spawnMeteors(14) // 12 Ağustos’a selam: 12–16 arası güzel duruyor
  animateMeteorShower()
}


function drawHearts() {
  hearts.value.forEach((heart) => {
    if (!heart.collected) {
      ctx.drawImage(heartImage, heart.x, heart.y, heart.size, heart.size)
    }
  })
}

function showFireworks() {
  const fireworks = []

  for (let i = 0; i < 100; i++) {
    fireworks.push({
      x: Math.random() * canvasWidth,
      y: canvasHeight,
      radius: Math.random() * 2 + 2,
      color: `hsl(${Math.random() * 360}, 100%, 70%)`,
      dx: (Math.random() - 0.5) * 6,
      dy: Math.random() * -10 - 3,
      life: 1
    })
  }

  function animateFireworks() {
    drawFinalScene()

    fireworksCtx.fillStyle = "rgba(0, 0, 0, 0.2)"
    fireworksCtx.fillRect(0, 0, canvasWidth, canvasHeight)

    fireworks.forEach(fx => {
      fx.x += fx.dx
      fx.y += fx.dy
      fx.dy += 0.002
      fx.life -= 0.005

      if (fx.life > 0) {
        fireworksCtx.beginPath()
        fireworksCtx.arc(fx.x, fx.y, fx.radius, 0, Math.PI * 2)
        fireworksCtx.fillStyle = fx.color
        fireworksCtx.globalAlpha = fx.life
        fireworksCtx.fill()
      }
    })

    fireworksCtx.globalAlpha = 1

    if (fireworks.some(fx => fx.life > 0)) {
      requestAnimationFrame(animateFireworks)
    }
  }

  animateFireworks()
}

function endGame() {
  gameOver.value = true
  cancelAnimationFrame(animationFrame)
  showBoy.value = true

  drawFinalScene()
  showFireworks()
  playFinalMusic()
  startMeteorShower()
}



function collectHearts() {
  hearts.value.forEach((heart) => {
    if (!heart.collected && checkCollision(player, heart)) {
      heart.collected = true
      collectedCount.value++

      console.log(`💘 Gün ${collectedCount.value + 2} Temmuz / Temmuz kalbi toplandı!`)

      if (collectedCount.value === heartCount) {
        endGame()
      }
    }
    if (collectedCount.value < heartCount) {
      hearts.value[collectedCount.value].collected = false
    }

  })
}



function clearCanvas() {
  ctx.clearRect(0, 0, canvasWidth, canvasHeight)
}

function update() {
  player.x += player.dx
  player.y += player.dy

  clearCanvas()
  drawPlayer()
  drawHearts()
  collectHearts()
  drawPrison()
  drawBoy()

  animationFrame = requestAnimationFrame(update)
}


function movePlayer(e) {
  const key = e.key
  switch (key) {
    case 'ArrowUp':
      player.dy = -player.speed
      player.dx = 0
      break
    case 'ArrowDown':
      player.dy = player.speed
      player.dx = 0
      break
    case 'ArrowLeft':
      player.dx = -player.speed
      player.dy = 0
      break
    case 'ArrowRight':
      player.dx = player.speed
      player.dy = 0
      break
  }
}

function stopPlayer(e) {
  if (
      ['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'].includes(e.key)
  ) {
    player.dx = 0
    player.dy = 0
  }
}

onMounted(() => {
  ctx = gameCanvas.value.getContext('2d')
  spawnHearts()
  fireworksCtx = fireworksCanvas.value.getContext('2d')

  // Masaüstü için
  window.addEventListener('keydown', movePlayer)
  window.addEventListener('keyup', stopPlayer)

  // Mobil için
  gameCanvas.value.addEventListener('touchstart', handleTouchStart)
  gameCanvas.value.addEventListener('touchmove', handleTouchMove)
  gameCanvas.value.addEventListener('touchend', handleTouchEnd)

  update()
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrame)
  cancelAnimationFrame(meteorAnimId)
  window.removeEventListener('keydown', movePlayer)
  window.removeEventListener('keyup', stopPlayer)

  gameCanvas.value.removeEventListener('touchstart', handleTouchStart)
  gameCanvas.value.removeEventListener('touchmove', handleTouchMove)
  gameCanvas.value.removeEventListener('touchend', handleTouchEnd)
})

</script>
