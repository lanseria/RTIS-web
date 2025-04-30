<script setup>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
// DecalGeometry is needed to 'project' tiles onto the curved surface
import { DecalGeometry } from 'three/examples/jsm/geometries/DecalGeometry.js'

// --- Configuration ---
const config = reactive({
  rows: 6,
  cols: 40,
  radius: 4.0,
  height: 3.0,
  tileOpacity: 0.6,
  tileColor: 0x0077FF, // Blueish color for tiles
  highlightColor: 0xFFFF00, // Yellow for highlight
  backgroundColor: 0x1A2B4A, // Dark blue background
  ambientLightColor: 0x406080,
  directionalLightColor: 0xFFFFFF,
})

// --- Refs ---
const container = ref(null) // To mount the canvas
const selectedTileId = ref(null) // To display the selected tile ID

// --- Three.js Variables ---
let scene, camera, renderer, controls
let baseCylinderMesh
const tileMeshes = [] // Store tile meshes for interaction
let raycaster, mouse
let selectedTileMesh = null // Keep track of the highlighted tile
let baseTileMaterial, highlightTileMaterial // Materials for tiles
let animationFrameId

// --- Lifecycle Hooks ---
onMounted(() => {
  if (container.value) {
    initThree()
    createTank()
    createTiles()
    setupInteraction()
    animate()
    window.addEventListener('resize', onWindowResize)
  }
})
onUnmounted(() => {
  window.removeEventListener('resize', onWindowResize)
  if (renderer) {
    renderer.domElement.removeEventListener('pointerdown', onPointerClick)
  }
  cancelAnimationFrame(animationFrameId)

  if (scene) {
    scene.traverse((object) => {
      if (object.geometry)
        object.geometry.dispose()
      if (object.material) {
        if (Array.isArray(object.material)) {
          object.material.forEach(material => disposeMaterial(material))
        }
        else {
          // Check if it's the text material and dispose its map (CanvasTexture)
          if (object.material.map && object.material.map.isCanvasTexture) {
            object.material.map.dispose()
          }
          disposeMaterial(object.material) // Dispose the material itself
        }
      }
      // Remove text meshes explicitly if not handled by scene traversal (though it should be)
      // if (object.userData.textMesh) {
      //     scene.remove(object.userData.textMesh);
      // }
    })
    // Clear arrays holding references
    tileMeshes.length = 0
  }
  if (renderer) {
    renderer.dispose()
    const canvas = renderer.domElement
    if (canvas && canvas.parentNode) {
      canvas.parentNode.removeChild(canvas)
    }
  }
  // Clear references
  scene = null
  camera = null
  renderer = null
  controls = null
  raycaster = null
  selectedTileMesh = null
  baseCylinderMesh = null
  baseTileMaterial = null
  highlightTileMaterial = null
})
// --- Helper Functions ---
// Make sure disposeMaterial handles texture disposal (it should already)
function disposeMaterial(material) {
  material.dispose()
  // Dispose textures
  for (const key of Object.keys(material)) {
    const value = material[key]
    if (value && typeof value === 'object' && value.isTexture) {
      // Check if it's a CanvasTexture before disposing (though base dispose should work)
      // console.log("Disposing texture:", value);
      value.dispose()
    }
  }
}

function generateTileId(row, col) {
  const rowLetter = String.fromCharCode(65 + row) // A, B, C...
  const colNumber = col + 1
  return `${rowLetter}${colNumber}`
}

// --- Initialization ---
function initThree() {
  const containerElement = container.value
  if (!containerElement)
    return

  // Scene
  scene = new THREE.Scene()
  scene.background = new THREE.Color(config.backgroundColor)
  scene.fog = new THREE.Fog(config.backgroundColor, 10, 25) // Add fog for depth

  // Camera
  const aspect = containerElement.clientWidth / containerElement.clientHeight
  camera = new THREE.PerspectiveCamera(50, aspect, 0.1, 100)
  camera.position.set(config.radius * 1.5, config.height * 1.2, config.radius * 1.5) // Adjust initial view
  camera.lookAt(0, 0, 0)

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(containerElement.clientWidth, containerElement.clientHeight)
  renderer.setPixelRatio(window.devicePixelRatio)
  containerElement.appendChild(renderer.domElement)

  // Lighting
  const ambientLight = new THREE.AmbientLight(config.ambientLightColor, 0.8)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(config.directionalLightColor, 1.0)
  directionalLight.position.set(5, 10, 7.5)
  scene.add(directionalLight)

  // Controls
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.1
  controls.screenSpacePanning = false
  controls.minDistance = config.radius * 1.1
  controls.maxDistance = config.radius * 5
  controls.target.set(0, 0, 0) // Look at the center of the tank base
  controls.maxPolarAngle = Math.PI // Allow looking from below

  // Raycaster for interaction
  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()
}

// --- Object Creation ---
function createTank() {
  const geometry = new THREE.CylinderGeometry(
    config.radius, // radiusTop
    config.radius, // radiusBottom
    config.height, // height
    config.cols * 2, // radialSegments (more for smoother texture mapping)
    config.rows * 2, // heightSegments (more for smoother texture mapping)
    true, // openEnded
  )

  const textureLoader = new THREE.TextureLoader()
  const wallTexture = textureLoader.load('/textures/rusty_metal_grid_ao_4k.jpg', // Make sure this path is correct
    (texture) => {
      texture.wrapS = THREE.RepeatWrapping
      texture.wrapT = THREE.RepeatWrapping
      texture.repeat.set(4, 1) // Adjust texture tiling if needed
      texture.needsUpdate = true
    }, undefined, // onProgress callback (optional)
    (err) => {
      console.error('Error loading texture:', err)
      // Use a fallback color if texture fails
      const fallbackMaterial = new THREE.MeshStandardMaterial({
        color: 0x607D8B, // Blue-grey fallback
        metalness: 0.6,
        roughness: 0.4,
        side: THREE.DoubleSide,
      })
      baseCylinderMesh.material = fallbackMaterial
    })

  const material = new THREE.MeshStandardMaterial({
    map: wallTexture,
    metalness: 0.7,
    roughness: 0.3,
    side: THREE.DoubleSide, // Render inside and outside
    // color: 0x88aaff, // Optional base color tint
  })

  baseCylinderMesh = new THREE.Mesh(geometry, material)
  baseCylinderMesh.rotation.x = 0 // Rotate to stand upright if needed (depends on geometry orientation)
  baseCylinderMesh.position.y = 0 // Center vertically at origin (adjust if needed)
  scene.add(baseCylinderMesh)
}
// Rename and modify the function
function createCanvasTextureForText(message, parameters = {}) {
  const fontface = parameters.fontface || 'Arial'
  const fontsize = parameters.fontsize || 32 // Increase fontsize for better texture quality
  const borderThickness = parameters.borderThickness || 0 // No border needed usually
  const backgroundColor = parameters.backgroundColor || { r: 0, g: 0, b: 0, a: 0.0 } // Transparent background
  const textColor = parameters.textColor || { r: 255, g: 255, b: 255, a: 1.0 } // White text

  const canvas = document.createElement('canvas')
  const context = canvas.getContext('2d')
  context.font = `Bold ${fontsize}px ${fontface}`

  // Measure text
  const metrics = context.measureText(message)
  const textWidth = metrics.width
  const textHeight = fontsize * 1.2 // Approximate height

  // Use calculated dimensions, maybe slightly padded
  const padding = fontsize * 0.2
  canvas.width = textWidth + padding * 2
  canvas.height = textHeight + padding * 2 // Make slightly taller for better vertical centering

  // Recalculate font size based on canvas size isn't strictly needed here if we size the plane later

  // Set styles AFTER setting canvas size
  context.font = `Bold ${fontsize}px ${fontface}`
  context.fillStyle = `rgba(${backgroundColor.r}, ${backgroundColor.g}, ${backgroundColor.b}, ${backgroundColor.a})`
  // context.strokeStyle = ... // Border not needed
  // context.lineWidth = ...

  // Draw background (transparent)
  context.fillRect(0, 0, canvas.width, canvas.height)

  // Draw text (centered)
  context.fillStyle = `rgba(${textColor.r}, ${textColor.g}, ${textColor.b}, ${textColor.a})`
  context.textAlign = 'center'
  context.textBaseline = 'middle'
  context.fillText(message, canvas.width / 2, canvas.height / 2)

  const texture = new THREE.CanvasTexture(canvas)
  texture.needsUpdate = true
  // texture.minFilter = THREE.LinearFilter; // Smoother texture
  // texture.magFilter = THREE.LinearFilter;

  return {
    texture,
    aspect: canvas.width / canvas.height,
  }
}
function createTiles() {
  // Create reusable materials
  baseTileMaterial = new THREE.MeshStandardMaterial({
    color: config.tileColor,
    opacity: config.tileOpacity,
    transparent: true,
    polygonOffset: true,
    polygonOffsetFactor: -2, // Push decals slightly away from base mesh
    depthTest: true,
    depthWrite: false, // Allow underlying texture to show through better
    side: THREE.DoubleSide,
  })

  highlightTileMaterial = new THREE.MeshStandardMaterial({
    color: config.highlightColor,
    opacity: config.tileOpacity + 0.2, // Slightly more opaque when highlighted
    transparent: true,
    polygonOffset: true,
    polygonOffsetFactor: -3, // Push highlighted further
    depthTest: true,
    depthWrite: false,
    emissive: config.highlightColor, // Make it glow slightly
    emissiveIntensity: 0.5,
    side: THREE.DoubleSide,
  })
  const tileHeight = config.height / config.rows
  const angleStep = (2 * Math.PI) / config.cols
  const tileCircumferenceWidth = (2 * Math.PI * config.radius) / config.cols

  const decalWidth = tileCircumferenceWidth * 0.95
  const decalHeight = tileHeight * 0.95
  const decalSize = new THREE.Vector3(decalWidth, decalHeight, 0.1)
  const orientation = new THREE.Euler()
  const position = new THREE.Vector3()
  const check = new THREE.Vector3(1, 1, 1) // Used for DecalGeometry check parameter

  const dummy = new THREE.Object3D()

  // Reusable text material
  const textMaterial = new THREE.MeshBasicMaterial({
    // map: set per instance,
    transparent: true,
    depthTest: true, // Test against other objects
    depthWrite: false, // Don't obscure things behind it unnecessarily
    side: THREE.DoubleSide, // Render both sides
    polygonOffset: true,
    polygonOffsetFactor: -4, // Render slightly in front of decal
  })

  for (let row = 0; row < config.rows; row++) {
    for (let col = 0; col < config.cols; col++) {
      const tileId = generateTileId(row, col)

      // --- Tile Decal ---
      const angle = col * angleStep + angleStep / 2
      const y = (config.height / 2) - (row * tileHeight + tileHeight / 2)

      position.set(
        config.radius * Math.cos(angle),
        y,
        config.radius * Math.sin(angle),
      )

      // Orientation for Decal (normal vector direction)
      dummy.position.copy(position)
      dummy.lookAt(0, y, 0) // Look towards the center axis
      orientation.copy(dummy.rotation)
      // DecalGeometry check parameter needs to be non-zero, often related to size
      check.set(decalWidth, decalHeight, 0.1).multiplyScalar(0.5)

      const decalGeom = new DecalGeometry(baseCylinderMesh, position, orientation, decalSize, check) // Added check
      const tileMesh = new THREE.Mesh(decalGeom, baseTileMaterial.clone()) // Clone needed if we change properties individually later
      tileMesh.userData.id = tileId
      tileMesh.userData.isTile = true
      scene.add(tileMesh)
      tileMeshes.push(tileMesh)

      // --- Add Text Plane ---
      const { texture: textTexture, aspect: textAspect } = createCanvasTextureForText(tileId)

      // Size the plane based on the text texture aspect ratio
      // Make the plane size relative to the decal size for consistency
      const textPlaneHeight = decalHeight * 0.3 // Adjust scale factor as needed
      const textPlaneWidth = textPlaneHeight * textAspect

      const textPlaneGeometry = new THREE.PlaneGeometry(textPlaneWidth, textPlaneHeight)
      const textPlaneMaterial = textMaterial.clone() // Clone to set unique map
      textPlaneMaterial.map = textTexture

      const textMesh = new THREE.Mesh(textPlaneGeometry, textPlaneMaterial)

      // Position the text plane slightly outside the decal along the normal
      const normal = new THREE.Vector3(Math.cos(angle), 0, Math.sin(angle)).normalize()
      const textOffset = 0.02 // Adjust distance from surface (smaller than sprite offset maybe)
      textMesh.position.copy(position).addScaledVector(normal, textOffset)

      // Orient the text plane
      // Apply the same orientation as the decal. The PlaneGeometry is initially in the XY plane.
      // We need it to align with the tangent plane defined by the decal's orientation.
      textMesh.quaternion.setFromEuler(orientation)
      // We might need an additional rotation because PlaneGeometry default orientation
      // (face along +Z) might not align perfectly with DecalGeometry expectation after applying Euler.
      // Often requires rotating the plane itself by 90 degrees on X or Y *before* applying the main orientation.
      // Let's try rotating around X first to make the plane initially vertical.
      textMesh.rotateY(Math.PI) // Rotate the plane so it's 'upright' before applying cylinder orientation

      // An alternative if the above doesn't work well is to use lookAt:
      // textMesh.lookAt(position.clone().sub(normal)); // Look back towards the position from slightly outside

      scene.add(textMesh)
      tileMesh.userData.textMesh = textMesh // Link text mesh to tile mesh
    }
  }
}

// --- Interaction ---
function setupInteraction() {
  renderer.domElement.addEventListener('pointerdown', onPointerClick, false)
}

function onPointerClick(event) {
  // Calculate mouse position in normalized device coordinates (-1 to +1)
  const rect = renderer.domElement.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  // Update the picking ray with the camera and mouse position
  raycaster.setFromCamera(mouse, camera)

  // Calculate objects intersecting the picking ray
  // IMPORTANT: Raycast against the `tileMeshes` array
  const intersects = raycaster.intersectObjects(tileMeshes, false) // false = don't check descendants

  if (intersects.length > 0) {
    const clickedObject = intersects[0].object

    // Check if it's a tile we added (using userData flag)
    if (clickedObject.userData.isTile) {
      // --- Highlighting Logic ---
      // Reset previous selection
      if (selectedTileMesh && selectedTileMesh !== clickedObject) {
        selectedTileMesh.material = baseTileMaterial // Revert material
      }

      // If clicking the same tile again, deselect it
      if (selectedTileMesh === clickedObject) {
        clickedObject.material = baseTileMaterial
        selectedTileMesh = null
        selectedTileId.value = null
      }
      else {
        // Highlight the new selection
        clickedObject.material = highlightTileMaterial
        selectedTileMesh = clickedObject
        selectedTileId.value = clickedObject.userData.id
        console.log(`Clicked Tile ID: ${selectedTileId.value}`)
      }
    }
  }
  else {
    // Clicked outside any tile - deselect
    if (selectedTileMesh) {
      selectedTileMesh.material = baseTileMaterial
      selectedTileMesh = null
      selectedTileId.value = null
    }
  }
}

// --- Animation Loop ---
function animate() {
  animationFrameId = requestAnimationFrame(animate)

  controls.update() // Only required if controls.enableDamping = true

  renderer.render(scene, camera)
}

// --- Resize Handling ---
function onWindowResize() {
  if (container.value && renderer && camera) {
    const width = container.value.clientWidth
    const height = container.value.clientHeight

    camera.aspect = width / height
    camera.updateProjectionMatrix()

    renderer.setSize(width, height)
  }
}
</script>

<template>
  <div class="visualization-container">
    <div ref="container" class="renderer-canvas" />
    <div v-if="selectedTileId" class="selected-info">
      Selected Tile: {{ selectedTileId }}
    </div>
  </div>
</template>

<style scoped>
.visualization-container {
  width: 100%;
  height: 100vh; /* Adjust height as needed */
  position: relative;
  background-color: #111; /* Fallback background */
}

.renderer-canvas {
  width: 100%;
  height: 100%;
  display: block; /* Remove extra space below canvas */
}

.selected-info {
  position: absolute;
  bottom: 10px;
  left: 10px;
  background-color: rgba(0, 0, 0, 0.6);
  color: white;
  padding: 5px 10px;
  border-radius: 4px;
  font-family: sans-serif;
  font-size: 14px;
}
</style>
