<template>
  <div class="w-full h-screen bg-black flex flex-col items-center justify-center text-white relative">
    <canvas
        ref="gameCanvas"
        :width="canvasWidth"
        :height="canvasHeight"
        class="border border-white"
    ></canvas>

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
      <h1 class="text-4xl text-pink-400 mb-4">🎆 2 Mayıs 🎆</h1>
      <p class="text-xl">Tanıştığımız Gün!<br />Seni çok seviyorum 💖</p>
    </div>
  </div>
</template>


<script setup>
import {ref, onMounted, onBeforeUnmount, computed} from 'vue'
import heartImgUrl from '../assets/heart.png'
import boyImgUrl from '../assets/boy-character.png'
import girlImgUrl from '../assets/justgirl.png'
import birdImgUrl from '../assets/birds.png'

const canvasWidth = 640
const canvasHeight = 480
const heartCount = 8
const showBoy = ref(false)
const hearts = ref([])
const collectedCount = ref(0)
const gameOver = ref(false)
const gameCanvas = ref(null)
const fireworksCanvas = ref(null)
const birds = []


const heartImage = new Image()
const boyImage = new Image()
const girlImage = new Image()
const birdImage = new Image()
girlImage.src = girlImgUrl
boyImage.src = boyImgUrl
heartImage.src = heartImgUrl
birdImage.src = birdImgUrl
const currentDate = computed(() => {
  const date = new Date(2024, 3, 25 + collectedCount.value) // Nisan = 3. ay
  return date.toLocaleDateString('tr-TR', { day: 'numeric', month: 'long' })
})

let ctx
let animationFrame
let fireworksCtx
let bgAudio

const prison = {
  x: canvasWidth - 1000,
  y: canvasHeight - 1000,
  size: 60,
}

const player = {
  x: 50,
  y: 50,
  size: 60,
  speed: 5,
  dx: 0,
  dy: 0,
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
  ctx.filter = 'none' // diğer çizimler etkilenmesin
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

  fireworksCtx.drawImage(boyImage, 120, canvasHeight - 120, 64, 64)

  fireworksCtx.drawImage(girlImage, 190, canvasHeight - 120, 64, 64)

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
  fireworksCtx.fillText("— 2 Mayıs 2025", canvasWidth / 2, 120)
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

  // 1. Patlama efekti oluştur
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

  // 2. Patlamayı frame frame işle
  function animateFireworks() {
    drawFinalScene()

    // Arkayı hafifçe karart (yavaşça sönen efekt)
    fireworksCtx.fillStyle = "rgba(0, 0, 0, 0.2)"
    fireworksCtx.fillRect(0, 0, canvasWidth, canvasHeight)

    fireworks.forEach(fx => {
      fx.x += fx.dx
      fx.y += fx.dy
      fx.dy += 0.002 // yerçekimi
      fx.life -= 0.005 // daha yavaş sönme

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

function spawnBirds() {
  birds.length = 0
  for (let i = 0; i < 3; i++) {
    birds.push({
      x: -60 - i * 100,
      y: 180 + Math.random() * 30,  // ⬅️ Yazının altına indirildi
      speed: 2 + Math.random(),
    })
  }
}



function drawBirds() {
  const birdImage = new Image()
  birdImage.src = birdImgUrl


  function animateBirds() {
    drawFinalScene()

    birds.forEach((b) => {
      b.x += b.speed
      fireworksCtx.drawImage(birdImage, b.x, b.y, 32, 32)
    })

    requestAnimationFrame(animateBirds)
  }
  animateBirds()
}

function endGame() {
  gameOver.value = true
  cancelAnimationFrame(animationFrame)
  showBoy.value = true

  drawFinalScene()
  showFireworks()
  playFinalMusic()
  spawnBirds()
  drawBirds()

}



function collectHearts() {
  hearts.value.forEach((heart) => {
    if (!heart.collected && checkCollision(player, heart)) {
      heart.collected = true
      collectedCount.value++

      console.log(`💘 Gün ${collectedCount.value + 24} Nisan / Mayıs kalbi toplandı!`)

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
  window.addEventListener('keydown', movePlayer)
  window.addEventListener('keyup', stopPlayer)
  update()
})


onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrame)
  window.removeEventListener('keydown', movePlayer)
  window.removeEventListener('keyup', stopPlayer)
})
</script>
