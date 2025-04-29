<script setup>
import { OrbitControls } from '@tresjs/cientos'
import { TresCanvas } from '@tresjs/core' // Import useRenderLoop if needed for animation later
import { MathUtils } from 'three' // Import MathUtils for degree to radian conversion if needed
import { ref } from 'vue'

// 圆柱体尺寸参数
const radius = 4.0 // Base radius
const height = 3
const rows = 6
const cols = 40
const tileHeight = height / rows
const tileWidthFactor = 0.99 // Gap factor horizontally
const tileHeightFactor = 0.99 // Gap factor vertically
const tileAngle = (2 * Math.PI) / cols // Angle per column
const tileArcWidth = tileAngle * radius // Approximate width along the arc
const tileThickness = 0.05 // Thickness for visual separation

// 存储所有瓦片信息
const tiles = ref([])

// 摄像机初始位置优化
const cameraPosition = ref([0, height * 0.5, 15]) // Adjusted Z for better initial view, Y centered
const cameraLookAt = ref([0, 0, 0]) // Look at the center

// OrbitControls auto-rotate
const showAxes = ref(false) // Control axes helper visibility

// 初始化瓦片数据
function initTiles() {
  tiles.value = [] // Clear previous tiles if re-initializing
  const statuses = ['normal', 'warning', 'error']
  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      const id = `${String.fromCharCode(65 + row)}${col + 1}`
      // Calculate center angle for the tile
      const angle = (col + 0.5) * tileAngle // Center the tile within its angular segment

      // Calculate position (place the center of the *flat* tile slightly outside the base radius)
      const tileRadius = radius + tileThickness / 2
      const x = tileRadius * Math.cos(angle)
      const z = tileRadius * Math.sin(angle)
      // Calculate Y position (center of the tile vertically)
      const y = (row + 0.5) * tileHeight - height / 2

      const position = [x, y, z]

      // Calculate rotation:
      // Rotate around Y axis to face outwards correctly.
      // The plane's default normal is +Z. We want it to point along the radius vector (x, 0, z).
      // The angle needed for this rotation around Y is the calculated `angle`.
      // We often need to add PI/2 (90 degrees) because the plane's *width* should align with the circumference.
      // Let's try rotating so the normal points outwards: that requires rotating by 'angle' around Y.
      // However, the default PlaneGeometry lies on XY, normal along Z.
      // To make it stand upright and face outwards, we rotate:
      // 1. Around Y by `angle` to orient it azimuthally.
      // 2. Often needs an additional rotation if the default orientation isn't what we expect.
      // Let's calculate the lookAt direction (towards the center) and use that. Or simpler:
      // Rotate around Y by `-angle` (or `angle`, depending on coordinate system/plane setup)
      const rotation = [0, -angle + Math.PI / 2, 0] // Rotate around Y axis. Negative often works with THREE convention.

      // Assign random status
      const randomStatus = statuses[Math.floor(Math.random() * statuses.length)]

      tiles.value.push({
        id,
        row,
        col,
        position,
        rotation,
        status: randomStatus, // Assign random status
        data: { info: `Some data for ${id}` }, // Add example data
      })
    }
  }
}

// 当前选中的瓦片
const selectedTile = ref(null)

// 点击瓦片处理
function handleTileClick(event) {
  // event.object gives the THREE.Mesh that was clicked
  // We need to find the corresponding tile data based on the mesh
  const clickedMesh = event.object
  // A simple way is to find the tile whose position matches closely,
  // or more robustly, assign the tile id to the mesh's userData.
  // Let's find by position for now (less robust if positions overlap exactly)
  const clickedPosition = clickedMesh.position.toArray()
  const foundTile = tiles.value.find(tile =>
    tile.position[0] === clickedPosition[0]
    && tile.position[1] === clickedPosition[1]
    && tile.position[2] === clickedPosition[2],
  )

  if (foundTile) {
    selectedTile.value = foundTile
    console.log(`Selected tile ${foundTile.id}`, foundTile)
  }
  else {
    console.log('Clicked on background or failed to find tile match.')
    selectedTile.value = null // Deselect if clicking elsewhere or match fails
  }
}

// Assign userData to meshes for easier identification (alternative to position matching)
// We can do this within the v-for loop using a little trick or a custom directive later if needed.
// For now, the position matching above will work.

initTiles()

// Optional: Add rotation animation
// const { onLoop } = useRenderLoop()
// onLoop(({ delta }) => {
//   // Rotate the group or individual tiles if needed
// })
</script>

<template>
  <TresCanvas window-size clear-color="#1A237E" shadows>
    <TresPerspectiveCamera
      :position="cameraPosition"
      :look-at="cameraLookAt"
      :fov="35"
    />
    <OrbitControls
      :enable-rotate="true"
      :enable-pan="false"
      :enable-zoom="true"
      :min-distance="radius * 1.5"
      :max-distance="radius * 5"
      :min-polar-angle="MathUtils.degToRad(70)"
      :max-polar-angle="MathUtils.degToRad(83)"
      :auto-rotate="false"
      :target="cameraLookAt"
    />

    <!-- Main Cylinder Body (Slightly smaller radius than tiles) -->
    <TresMesh :position="[0, 0, 0]" :receive-shadow="true">
      <TresCylinderGeometry :args="[radius, radius, height, 64, 1, false]" />
      <TresMeshStandardMaterial
        color="#455A64"
        :roughness="0.6"
        :metalness="0.4"
      />
    </TresMesh>

    <!-- Inner Cylinder Wall (Optional, for thickness illusion) -->
    <TresMesh :position="[0, 0, 0]">
      <TresCylinderGeometry :args="[radius * 0.98, radius * 0.98, height * 0.99, 64, 1, true]" />
      <TresMeshStandardMaterial
        color="#37474F"
        :roughness="0.7"
        :metalness="0.3"
        side="BackSide"
      />
    </TresMesh>

    <!-- Tiles Group -->
    <TresGroup>
      <TresMesh
        v-for="tile in tiles"
        :key="tile.id"
        :position="tile.position"
        :rotation="tile.rotation"
        :cast-shadow="true"
        :receive-shadow="true"
        @click="handleTileClick"
      >
        <TresPlaneGeometry
          :args="[tileArcWidth * tileWidthFactor, tileHeight * tileHeightFactor]"
        />
        <TresMeshStandardMaterial
          :color="tile.status === 'normal' ? '#81C784' // Lighter Green
            : tile.status === 'warning' ? '#FFB74D' // Lighter Orange
              : '#E57373'"
          :roughness="0.8"
          :metalness="0.1"
        />
      </TresMesh>
    </TresGroup>

    <!-- Selected Tile Highlight -->
    <TresMesh
      v-if="selectedTile"
      :position="selectedTile.position"
      :rotation="selectedTile.rotation"
    >
      <TresPlaneGeometry
        :args="[tileArcWidth * tileWidthFactor * 1.05, tileHeight * tileHeightFactor * 1.05]"
      />
      <TresMeshStandardMaterial
        color="#64B5F6"
        :metalness="0.1"
        :roughness="0.5"
        :transparent="true"
        :opacity="0.8"
        :depth-test="false"
      />
    </TresMesh>

    <!-- Lighting -->
    <TresAmbientLight :intensity="0.6" />
    <TresDirectionalLight
      :position="[5, 10, 7]"
      :intensity="1.0"
      cast-shadow
      :shadow-camera-top="height * 1.5"
      :shadow-camera-bottom="-height * 1.5"
      :shadow-camera-left="-radius * 1.5"
      :shadow-camera-right="radius * 1.5"
    />
    <TresDirectionalLight
      :position="[-5, -5, -5]"
      :intensity="0.3"
    />

    <!-- Optional Floor -->
    <!--
    <TresMesh :rotation-x="-Math.PI / 2" :position-y="-height / 2 - 0.1" receive-shadow>
      <TresPlaneGeometry :args="[radius * 4, radius * 4]" />
      <TresMeshStandardMaterial color="#263238" :roughness="0.9" />
    </TresMesh>
    -->

    <!-- Axes Helper -->
    <TresAxesHelper v-if="showAxes" :args="[radius * 1.5]" />
  </TresCanvas>
</template>

<style>
body {
  margin: 0;
  overflow: hidden;
  background-color: #1a237e; /* Match canvas background */
}
</style>
