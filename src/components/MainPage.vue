<template>
  <div class="welcome-text" v-on="handlers"></div>
  <div class="form">
    <n-slider v-model:value="cubeAmount" vertical class="custom-slider" />
  </div>
  <div class="switch-form-controlls">
    <n-switch v-model:value="pausing" size="large">
    </n-switch>
  </div>
  <div class="button-form">
    <n-button strong secondary round class="custom-button" @click="resetBlocks">
      Reset
    </n-button>
  </div>
  <div class="full-screen-button-form">
    <n-button v-if="fullScreen" strong secondary round class="custom-button" @click="exitFullScreen">
      Exit Full Screen
    </n-button>
    <n-button v-else strong secondary round class="custom-button" @click="requestFullscreen">
      Full Screen
    </n-button>
  </div>

  <div class="switch-form">
    <n-switch v-model:value="showMainContent" size="large">
    </n-switch>
  </div>
  <div v-if="isMobile()" class="mobile-main-content hide-mobile-main-content" id="mainContent" @touch="showMainContent = true">
    <h1>Hiya, my name is Cal</h1>

    <p>Welcome to my front page, I'm a full stack developer who works primarily with Vue on the frontend and a mix of node/serverless and PHP on the backend. In my free time I like to paint, ski, and play tennis.</p>
    <div class="main-content-footer"><div style="padding-right: 15px;"><a class="mail-link mail-link-hidden" id="mailLink" href="mailto:cal.macconnachie@gmail.com" target="_blank"><n-icon><mail-icon/></n-icon></a></div></div>
  </div>
  <div v-else class="main-content hide-main-content" id="mainContent" @click="showMainContent = true">
    <h1 v-if="!playSnake" style="background: none">Hiya, my name is Cal</h1>

    <p v-if="!playSnake" style="background: none">Welcome to my front page, I'm a full stack developer who works primarily with Vue on the frontend and a mix of node/serverless and PHP on the backend. In my free time I like to paint, ski, and play tennis. I plan to add more to this in the future... maybe...</p>
    <div v-if="!playSnake" class="play-button-container">
      <n-button strong round style="color: #ECE2D0" @click="startSnakeGame">
      Start Snake
      </n-button>
    </div>
    <div v-if="playSnake" class="snake-score">Score: {{score}} <n-button strong round style="color: #ECE2D0"</div>
    <div class="game-over-screen" ref="gameOverScreen">
      <div class="game-over-text">Game Over</div>
      <div class="game-over-text">Score: {{score}}</div>
      <div class="game-over-text">High Score: {{highScore}}</div>
      <div class="play-button-container">
        <n-button strong round style="color: #ECE2D0" @click="restartSnakeGame">
        Play Again?
        </n-button>
      </div>
    </div>
    <div class="game-container" ref="gameContainer" id="gameContainer"></div>
    <div class="main-content-footer"><div style="padding-right: 15px;"><a class="mail-link mail-link-hidden" id="mailLink" href="mailto:cal.macconnachie@gmail.com" target="_blank"><n-icon class="icon"><mail-icon/></n-icon></a></div></div>
  </div>
  <div class="container" ref="canvas"/>
</template>

<script>
// colors
// 0xA26769 copper rose
// 0xBFB5AF pale silver
// 0xECE2D0 bone
// 0xD5B9B2 silver pink
// 0x582C4D dark byzantium
import * as THREE from 'three'
import { EmailOutlined as MailIcon } from '@vicons/material'
export default {
  name: 'MainPage',
  components: {
    MailIcon
  },
  data: function () {
    return {
      cubeAmount: 100,
      show: false,
      dragging: false,
      cubeIds: [],
      rotationThreshold: 0.0001,
      directionThreshold: 0.06,
      slowDownFactor: 0.9,
      pausing: true,
      showMainContent: false,
      boxControlls: true,
      palletteNumber: 127,
      fullScreen: false,
      direction: 'up',
      playSnake: false,
      gameOverFlag: false,
      score: 0,
      highScore: 0,
      frameCounter: 0,
      difficulty: 0.35,
      cameraSpeed: 1,
      keyStates: {
        w: false,
        a: false,
        s: false,
        d: false
      },
      handlers: {
        mousedown: () => {
          this.dragging = true
        },
        mouseup: () => {
          this.dragging = false
        },
        mousemove: this.draggingScreen
      }
    }
  },
  created () {
    if (this.isMobile()) {
      this.cubeAmount = 25
      this.dragging = true
      this.handlers = {
        touchmove: this.dragCubes
      }
    }
  },
  mounted: function() {
    this.cubeRotations = {}
    this.origionalCubeRoations = {}
    this.cubeDirections = {}
    this.origionalCubeDirections = {}
    window.addEventListener( 'resize', this.onWindowResize, false )
    this.init()
    this.animate()
  },
  methods: {
    init() {
      this.container = this.$refs.canvas
      this.scene = new THREE.Scene()
      this.scene.background = new THREE.Color(0xBFB5AF)
      this.camera = new THREE.OrthographicCamera(window.innerWidth / -2, window.innerWidth / 2, window.innerHeight / 2, window.innerHeight / -2, 0.1, 21000)
      this.camera.position.set(0, 0, 21000)

      this.renderer = new THREE.WebGLRenderer({ antialias: true })
      this.renderer.setSize(window.innerWidth, window.innerHeight)
      this.container.appendChild(this.renderer.domElement)

      this.light = new THREE.DirectionalLight( 0xECE2D0, 1 )
      this.light.position.set( 0.5, 0, 20010 )
      this.light.target = new THREE.Object3D(0, 0, 0)
      this.scene.add( this.light )

      this.cubes = []
      this.createCubes(this.cubeAmount)

      this.renderer.render(this.scene, this.camera)
    },
    animate() {
      requestAnimationFrame(this.animate)
      this.cubes.forEach((cube) => {
        cube.rotation.x += this.cubeRotations[cube.uuid].x
        cube.rotation.y += this.cubeRotations[cube.uuid].y
        cube.rotation.z += this.cubeRotations[cube.uuid].z
        if (!this.cubeAtEdge(cube)) {
          cube.position.x += this.cubeDirections[cube.uuid].x
          cube.position.y += this.cubeDirections[cube.uuid].y
        } else {
          const resetPosition = this.calculateResetPosition(cube)
          cube.position.x = resetPosition.x
          cube.position.y = resetPosition.y
        }
        if (this.pausing) {
          if (Math.abs(this.cubeRotations[cube.uuid].x) > this.rotationThreshold) {
            this.cubeRotations[cube.uuid].x = this.cubeRotations[cube.uuid].x * this.slowDownFactor
          } else {
            this.cubeRotations[cube.uuid].x = 0
          }
          if (Math.abs(this.cubeRotations[cube.uuid].y) > this.rotationThreshold) {
            this.cubeRotations[cube.uuid].y = this.cubeRotations[cube.uuid].y * this.slowDownFactor
          } else {
            this.cubeRotations[cube.uuid].y = 0
          }
          if (Math.abs(this.cubeRotations[cube.uuid].z) > this.rotationThreshold) {
            this.cubeRotations[cube.uuid].z = this.cubeRotations[cube.uuid].z * this.slowDownFactor
          } else {
            this.cubeRotations[cube.uuid].z = 0
          }
          if (Math.abs(this.cubeDirections[cube.uuid].x) > this.directionThreshold) {
            this.cubeDirections[cube.uuid].x = this.cubeDirections[cube.uuid].x * this.slowDownFactor
          } else {
            this.cubeDirections[cube.uuid].x = 0
          }
          if (Math.abs(this.cubeDirections[cube.uuid].y) > this.directionThreshold) {
            this.cubeDirections[cube.uuid].y = this.cubeDirections[cube.uuid].y * this.slowDownFactor
          } else {
            this.cubeDirections[cube.uuid].y = 0
          }
        } else {
          this.cubeRotations[cube.uuid].x = this.origionalCubeRoations[cube.uuid].x
          this.cubeRotations[cube.uuid].y = this.origionalCubeRoations[cube.uuid].y
          this.cubeRotations[cube.uuid].z = this.origionalCubeRoations[cube.uuid].z
          this.cubeDirections[cube.uuid].x = this.origionalCubeDirections[cube.uuid].x
          this.cubeDirections[cube.uuid].y = this.origionalCubeDirections[cube.uuid].y
        }
      })
      this.renderer.render(this.scene, this.camera)
    },
    generatePastelColour () {
      if (Math.random() > 0.5) {
        return '#'+(0x1000000+Math.random()*0xffffff).toString(16).substr(1,6)
      } else {
        const r = (Math.round(Math.random() * this.palletteNumber) + this.palletteNumber).toString(16)
        const g = (Math.round(Math.random() * this.palletteNumber) + this.palletteNumber).toString(16)
        const b = (Math.round(Math.random() * this.palletteNumber) + this.palletteNumber).toString(16)
        return '#' + r + g + b
      }
    },
    createCubes(number) {
      const speedLimit = 5
      const cubeUpperLimit = 200
      const cubeLowerLimit = 25
      for (let i = 0; i < number; i++) {
        const boxX = Math.floor(Math.random() * (cubeUpperLimit - cubeLowerLimit + 1)) + cubeLowerLimit
        const boxY = Math.floor(Math.random() * (cubeUpperLimit - cubeLowerLimit + 1)) + cubeLowerLimit
        const boxZ = Math.floor(Math.random() * (cubeUpperLimit - cubeLowerLimit + 1)) + cubeLowerLimit
        const geometry = new THREE.BoxGeometry(boxX, boxY, boxZ)
        const material = new THREE.MeshPhongMaterial({ color: this.generatePastelColour() })
        this.cube = new THREE.Mesh(geometry, material)
        this.cube.position.x = Math.random() * (Math.random() >= 0.5 ? 1 : -1) * window.innerWidth / 2.25
        // this.cube.position.x = Math.random() * window.innerWidth / 2 - window.innerWidth / 4
        this.cube.position.y = Math.random() * window.innerHeight / 2 - window.innerHeight / 4
        this.cube.position.z = this.getUniqueZ()
        this.cube.rotation.x = Math.random() * 2 * Math.PI
        this.cube.rotation.y = Math.random() * 2 * Math.PI
        this.cube.rotation.z = Math.random() * 2 * Math.PI
        this.scene.add(this.cube)
        this.cubes.push(this.cube)
        this.cubeIds.push(this.cube.uuid)
      }

      for (const cube of this.cubes) {
        if (this.cubeRotations[cube.uuid] !== undefined) {
          continue
        }
        this.cubeRotations[cube.uuid] = {
          x: Math.random() / 100,
          y: Math.random() / 100,
          z: Math.random() / 100
        }
        this.origionalCubeRoations[cube.uuid] = {
          x: this.cubeRotations[cube.uuid].x,
          y: this.cubeRotations[cube.uuid].y,
          z: this.cubeRotations[cube.uuid].z
        }
        this.cubeDirections[cube.uuid] = {
          x: (Math.random() * speedLimit) * (Math.random() < 0.5 ? -1 : 1),
          y: (Math.random() * speedLimit) * (Math.random() < 0.5 ? -1 : 1)
        }
        this.origionalCubeDirections[cube.uuid] = {
          x: this.cubeDirections[cube.uuid].x,
          y: this.cubeDirections[cube.uuid].y
        }
      }
    },
    cubeAtEdge(cube) {
      return cube.position.x > window.innerWidth || cube.position.x < -window.innerWidth || cube.position.y > window.innerHeight || cube.position.y < -window.innerHeight
    },
    calculateResetPosition(cube) {
      const resetPosition = {
        x: cube.position.x,
        y: cube.position.y
      }
      if (cube.position.x > window.innerWidth) {
        resetPosition.x = -window.innerWidth
      } else if (cube.position.x < -window.innerWidth) {
        resetPosition.x = window.innerWidth
      }
      if (cube.position.y > window.innerHeight) {
        resetPosition.y = -window.innerHeight
      } else if (cube.position.y < -window.innerHeight) {
        resetPosition.y = window.innerHeight
      }
      return resetPosition
    },
    draggingScreen (event) {
      if (this.boxControlls) {
        this.updateDirections(event)
      }
    },
    updateDirections (event) {
      if (this.dragging) {
        const x = event.clientX
        const y = event.clientY
        const xDirection = x > window.innerWidth / 2 ? 1 : -1
        const yDirection = y > window.innerHeight / 2 ? 1 : -1
        const xSpeed = Math.abs(x - window.innerWidth / 2) / 100
        const ySpeed = Math.abs(y - window.innerHeight / 2) / 100
        for (const cube of this.cubes) {
          this.cubeDirections[cube.uuid].x = this.origionalCubeDirections[cube.uuid].x * xSpeed * xDirection
          this.cubeDirections[cube.uuid].y = this.origionalCubeDirections[cube.uuid].y * ySpeed * yDirection
        }
      } else {
        if (this.pausing) {
          for (const cube of this.cubes) {
            this.cubeDirections[cube.uuid].x = 0
            this.cubeDirections[cube.uuid].y = 0
          }
        } else {
          for (const cube of this.cubes) {
            this.cubeDirections[cube.uuid].x = this.origionalCubeDirections[cube.uuid].x
            this.cubeDirections[cube.uuid].y = this.origionalCubeDirections[cube.uuid].y
          }
        }
      }
    },
    getUniqueZ () {
      let z = 100
      const multiple = 200
      if (this.cubes.length === 0) {
        return z
      }
      for (const cube of this.cubes) {
        if (cube.position.z === z) {
          z += multiple
        }
      }
      return z
    },
    resetBlocks () {
      for (let i = 0; i < this.cubeAmount; i++) {
        const cube = this.cubes.pop()
        this.scene.remove(cube)
      }
      this.createCubes(this.cubeAmount)
    },
    startSnakeGame () {
      this.playSnake = true
      this.initGame()
    },
    initGame() {
      this.gameContainer = this.$refs.gameContainer
      this.gameScene = new THREE.Scene()
      this.gameScene.background = new THREE.Color(0x353431)
      // main-content width
      const mainContentWidth = document.getElementById('mainContent').offsetWidth
      const mainContentHeight = document.getElementById('mainContent').offsetHeight
      // this.gameCamera = new THREE.OrthographicCamera(mainContentWidth / -2, mainContentWidth / 2, mainContentHeight / 2, mainContentHeight / -2, 0.1, 100)
      const aspect = mainContentWidth / mainContentHeight
      this.usableWidth = 25 * aspect
      this.gameCamera = new THREE.OrthographicCamera(-this.usableWidth, this.usableWidth, 25, -25, 0.1, 100)
      this.gameCamera.position.set(0, 0, 100)
      this.gameCamera.up.set( 0, 0, 1 )
      this.gameRenderer = new THREE.WebGLRenderer({ antialias: true })
      this.gameRenderer.setSize(mainContentWidth, mainContentHeight - 25)
      this.gameContainer.appendChild(this.gameRenderer.domElement)


      this.gameLight = new THREE.DirectionalLight( 0xECE2D0, 1 )
      this.gameLight.position.set( 0.5, 0, 100 )
      this.gameLight.target = new THREE.Object3D(0, 0, 0)
      this.gameScene.add(this.gameLight)
      this.renderer.render(this.gameScene, this.gameCamera)
      this.setupGame()
      this.setupKeyboardListeners()
      this.animateGame()
    },
    restartSnakeGame () {
      this.playSnake = true
      this.gameOverFlag = false
      this.score = 0
      this.$refs.gameOverScreen.style.display = 'none'
      this.setupGame()
    },
    setupGame () {
      const geometry = new THREE.BoxGeometry(1, 1, 1)
      const material = new THREE.MeshPhongMaterial({ color: this.generatePastelColour() })
      this.head = new THREE.Mesh(geometry, material)
      this.head.position.set(0, 0, 9)
      this.gameScene.add(this.head)
      this.tail = []
      this.addNewSnakeSegment('#353431')
    },
    setupKeyboardListeners () {
      window.addEventListener('keydown', (event) => {
        if (event.key === 'ArrowUp') {
          if (this.direction === 'down') {
            return
          }
          this.direction = 'up'
        } else if (event.key === 'ArrowDown') {
          if (this.direction === 'up') {
            return
          }
          this.direction = 'down'
        } else if (event.key === 'ArrowLeft') {
          if (this.direction === 'right') {
            return
          }
          this.direction = 'left'
        } else if (event.key === 'ArrowRight') {
          if (this.direction === 'left') {
            return
          }
          this.direction = 'right'
        } else if (event.key === 'w') {
          this.keyStates.w = true
        } else if (event.key === 'a') {
          this.keyStates.a = true
        } else if (event.key === 's') {
          this.keyStates.s = true
        } else if (event.key === 'd') {
          this.keyStates.d = true
        }
      })
      window.addEventListener('keyup', (event) => {
        if (event.key === 'w') {
          this.keyStates.w = false
        } else if (event.key === 'a') {
          this.keyStates.a = false
        } else if (event.key === 's') {
          this.keyStates.s = false
        } else if (event.key === 'd') {
          this.keyStates.d = false
        }
      })
    },
    createFoodPellet () {
      const geometry = new THREE.BoxGeometry(1, 1, 1)
      const material = new THREE.MeshPhongMaterial({ color: this.generatePastelColour() })
      this.pellet = new THREE.Mesh(geometry, material)
      const mainContentWidth = 25
      const y = Math.floor(Math.random() * mainContentWidth) - mainContentWidth / 2
      const x = Math.floor(Math.random() * this.usableWidth) - this.usableWidth / 2
      this.pellet.position.set(x, y, 9)
      this.gameScene.add(this.pellet)
    },
    addNewSnakeSegment (color = '#A26769') {
      const geometry = new THREE.BoxGeometry(1, 1, 1)
      const material = new THREE.MeshPhongMaterial({ color })
      const newSegment = new THREE.Mesh(geometry, material)
      this.tail.push(newSegment)
      this.gameScene.add(this.tail[this.tail.length - 1])
    },
    moveSnake () {
      if (this.direction === 'up') {
        this.head.position.y += this.difficulty
      } else if (this.direction === 'down') {
        this.head.position.y -= this.difficulty
      } else if (this.direction === 'left') {
        this.head.position.x -= this.difficulty
      } else if (this.direction === 'right') {
        this.head.position.x += this.difficulty
      }
      for (let i = this.tail.length - 1; i >= 1; i--) {
        let newPos = new THREE.Vector2(this.tail[i - 1].position.x, this.tail[i - 1].position.y)
        this.tail[i].position.x = newPos.x
        this.tail[i].position.y = newPos.y
      }
      this.tail[0].position.x = this.head.position.x
      this.tail[0].position.y = this.head.position.y
      
      // loop player back to other side of screen
      const mainContentWidth = this.usableWidth
      const mainContentHeight = 25
      if (this.head.position.x > mainContentWidth) {
        this.head.position.x = -mainContentWidth
      } else if (this.head.position.x < -mainContentWidth) {
        this.head.position.x = mainContentWidth
      } else if (this.head.position.y > mainContentHeight) {
        this.head.position.y = -mainContentHeight
      } else if (this.head.position.y < -mainContentHeight) {
        this.head.position.y = mainContentHeight
      }
    },
    checkForCollision () {
      if (this.tail.length <= 15) {
        return
      }
      for (let i = 15; i < this.tail.length; i++) {
        const segment = this.tail[i]
        const acceptableDistance = 0.5
        if (Math.abs(this.head.position.x - segment.position.x) < acceptableDistance && Math.abs(this.head.position.y - segment.position.y) < acceptableDistance) {
          this.gameOver()
        }
      }
    },
    gameOver () {
      this.gameOverFlag = true
      this.$refs.gameOverScreen.style.display = 'flex'
      this.gameScene.remove(this.head)
      this.head = null
      this.tail.forEach((segment) => {
        this.gameScene.remove(segment)
      })
      this.tail = []
      this.gameScene.remove(this.pellet)
      this.pellet = null
      this.direction = 'up'
    },
    animateGame () {
      requestAnimationFrame(this.animateGame)
      if (!this.pellet && !this.gameOverFlag) {
        this.createFoodPellet()
      }
      this.checkForCollision()
      if (!this.gameOverFlag) {
        this.moveSnake()
        const acceptableDistance = 1
        if (Math.abs(this.head.position.x - this.pellet.position.x) < acceptableDistance && Math.abs(this.head.position.y - this.pellet.position.y) < acceptableDistance) {
          this.pelletColor = this.pellet.material.color.getHex()
          this.gameScene.remove(this.pellet)
          this.pellet = null
          this.score += 1
        }
      }
      // this.moveCamera()
      this.gameRenderer.render(this.gameScene, this.gameCamera)
    },
    moveCamera () {
      // sphere formual x^2 + y^2 + z^2 = r^2 where r is 100
      if (this.keyStates.w) {
        this.gameCamera.position.y -= 0.1
      } else if (this.keyStates.s) {
        this.gameCamera.position.y += 0.1
      }
      if (this.keyStates.a) {
        this.gameCamera.position.x -= 0.1
      } else if (this.keyStates.d) {
        this.gameCamera.position.x += 0.1
      }
      this.gameCamera.position.z = Math.sqrt(10000 - (this.gameCamera.position.x * this.gameCamera.position.x) - (this.gameCamera.position.y * this.gameCamera.position.y))
      this.gameCamera.lookAt(0, 0, 0)
    },
    onWindowResize() {
      this.camera.aspect = window.innerWidth / window.innerHeight
      this.camera.updateProjectionMatrix()
      this.renderer.setSize(window.innerWidth, window.innerHeight)
    },
    isMobile() {
      if(/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)) {
        return true
      } else {
        return false
      }
    },
    dragCubes (event) {
      // move cubes for moblie drag
      const x = event.touches[0].pageX
      const y = event.touches[0].pageY
      const xDirection = x > window.innerWidth / 2 ? 1 : -1
      const yDirection = y > window.innerHeight / 2 ? 1 : -1
      const xSpeed = Math.abs(x - window.innerWidth / 2) / 100
      const ySpeed = Math.abs(y - window.innerHeight / 2) / 100
      for (const cube of this.cubes) {
        this.cubeDirections[cube.uuid].x = this.origionalCubeDirections[cube.uuid].x * xSpeed * xDirection
        this.cubeDirections[cube.uuid].y = this.origionalCubeDirections[cube.uuid].y * ySpeed * yDirection
      }
    },
    requestFullscreen() {
      const element = document.documentElement
      if (element.requestFullscreen) {
        element.requestFullscreen()
      } else if (element.mozRequestFullScreen) {
        element.mozRequestFullScreen()
      } else if (element.webkitRequestFullscreen) {
        element.webkitRequestFullscreen()
      } else if (element.msRequestFullscreen) {
        element.msRequestFullscreen()
      }
      this.fullScreen = true
    },
    exitFullScreen() {
      if (document.exitFullscreen) {
        document.exitFullscreen()
      } else if (document.mozCancelFullScreen) {
        document.mozCancelFullScreen()
      } else if (document.webkitExitFullscreen) {
        document.webkitExitFullscreen()
      }
      this.fullScreen = false
    }
  },

  watch: {
    cubeAmount() {
      if (this.cubes === undefined) {
        return
      }
      const difference = this.cubeAmount - this.cubes.length
      if (difference > 0) {
        this.createCubes(difference)
      } else {
        for (let i = 0; i < Math.abs(difference); i++) {
          const cube = this.cubes.pop()
          this.scene.remove(cube)
        }
      }
    },
    showMainContent() {
      if (this.showMainContent) {
        if (this.isMobile()) {
          document.querySelector('#mainContent').classList.add('show-mobile-main-content')
          document.querySelector('#mainContent').classList.remove('hide-mobile-main-content')
        } else {
          document.querySelector('#mainContent').classList.add('show-main-content')
          document.querySelector('#mainContent').classList.remove('hide-main-content')
        }
        document.querySelector('#mailLink').classList.remove('mail-link-hidden')
        document.querySelector('#mailLink').classList.add('mail-link-visible')
      } else {
        if (this.isMobile()) {
          document.querySelector('#mainContent').classList.add('hide-mobile-main-content')
          document.querySelector('#mainContent').classList.remove('show-mobile-main-content')
        } else {
          document.querySelector('#mainContent').classList.add('hide-main-content')
          document.querySelector('#mainContent').classList.remove('show-main-content')
        }
        document.querySelector('#mailLink').classList.remove('mail-link-visible')
        document.querySelector('#mailLink').classList.add('mail-link-hidden')
      }
    },
    score() {
      if (this.gameOverFlag) {
        return
      }
      if (this.score > this.highScore) {
        this.highScore = this.score
      }
      // add to snake
      this.addNewSnakeSegment(this.pelletColor)
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="css">
.container {
  width: 100vw;
  height: 100vh;
  display: block;
  margin: 0;
  padding: 0;
}
.game-conatiner{
  width: 100%;
  height: 500px;
  display: block;
  margin: 15rem;
  padding: 15rem;
  background-color: aqua;
}
.snake-score {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 100;
  color: white;
  font-size: 2rem;
  padding: 1rem;
}
.welcome-text {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 100;
}
.form {
  position: absolute;
  background-color: rgba(53, 52, 49, 0.7);
  border-radius: 10px;
  top: 10px;
  left: 10px;
  width: 5%;
  height: 300px;
  z-index: 101;
  padding: 10px;
}
.switch-form {
  position: absolute;
  background-color: rgba(53, 52, 49, 0.7);
  border-radius: 10px;
  top: 10px;
  right: 10px;
  z-index: 101;
  padding: 10px;
}
.custom-slider{
  position: relative;
  left: 50%;
  transform: translate(-50%, 0%);
}
.button-form {
  position: absolute;
  /* background-color: #353431; */
  bottom: 10px;
  right: 10px;
  z-index: 101;
  padding: 10px;
}
.full-screen-button-form {
  position: absolute;
  /* background-color: #353431; */
  bottom: 10px;
  left: 50%;
  transform: translate(-50%, 0);
  z-index: 101;
  padding: 10px;
}
.custom-button {
  position: relative;
  left: 50%;
  transform: translate(-50%, 0%);
}
.switch-form-controlls {
  position: absolute;
  background-color: rgba(53, 52, 49, 0.7);
  border-radius: 10px;
  bottom: 10px;
  left: 10px;
  z-index: 101;
  padding: 10px;
}
.main-content {
  position: absolute;
  background-color: #353431;
  border-radius: 10px;
  top: 10px;
  left: 50%;
  transform: translate(-50%, 0);
  width: 75%;
  height: 95%;
  z-index: 101;
  padding: 10px;
  color: #ECE2D0;
  font-size: 1.5em;
  transition: all 0.5s ease-in-out;
  overflow: hidden;
  overflow-y: scroll;
}
.hide-main-content {
  height: 0;
  width: 0;
  top: 10px;
  transform: translate(-50%, 0);
  color: rgba(53, 52, 49, 1);
}
.show-main-content {
  height: 95%;
  width: 75%;
  top: 50%;
  transform: translate(-50%, -50%);
  color: #ECE2D0;
}

.mobile-main-content {
  position: absolute;
  background-color: rgba(53, 52, 49, 1);
  border-radius: 10px;
  top: 10px;
  left: 50%;
  transform: translate(-50%, 0);
  width: 65%;
  height: 70%;
  z-index: 101;
  padding: 10px;
  color: #ECE2D0;
  font-size: 1.5em;
  transition: all 0.5s ease-in-out;
  overflow: hidden;
  overflow-y: scroll;
}
.hide-mobile-main-content {
  height: 0;
  width: 0;
  top: 10px;
  transform: translate(-50%, 0);
  color: rgba(53, 52, 49, 1);
}
.show-mobile-main-content {
  height: 70%;
  width: 65%;
  top: 50%;
  transform: translate(-50%, -50%);
  color: #ECE2D0;
}
.main-content-footer {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 10%;
  z-index: 101;
  display: flex;
  justify-content: flex-end;
  align-items: center;
}
.mail-link {
  color: #ECE2D0;
  text-decoration: none;
  transition: all 0.5s ease-in-out;
}
.mail-link-hidden {
  color: #353431;
  pointer-events: none;

}
.mail-link-visible {
  color: #ECE2D0;
}
.game-over-screen {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 100;
  background-color: rgba(53, 52, 49, 0.7);
  display: none;
  justify-content: center;
  align-items: center;
  flex-direction: column;
}
</style>
