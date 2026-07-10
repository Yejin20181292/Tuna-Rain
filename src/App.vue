<script setup lang="ts">
import { ref, onMounted } from 'vue';

const processedTunaSrc = ref('');

onMounted(() => {
  const img = new Image();
  img.src = '/tuna-can.png';
  img.onload = () => {
    const canvas = document.createElement('canvas');
    canvas.width = img.width;
    canvas.height = img.height;
    const ctx = canvas.getContext('2d');
    if (!ctx) return;
    
    ctx.drawImage(img, 0, 0);
    const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const data = imgData.data;
    const width = canvas.width;
    const height = canvas.height;
    
    // Helper to get pixel index
    const getIdx = (x: number, y: number) => (y * width + x) * 4;
    
    // Visited map for flood fill
    const visited = new Uint8Array(width * height);
    const queue: [number, number][] = [];
    
    // Start flood fill from the four corners of the image
    const corners = [
      [0, 0],
      [width - 1, 0],
      [0, height - 1],
      [width - 1, height - 1]
    ];
    
    for (const [cx, cy] of corners) {
      const idx = cy * width + cx;
      if (!visited[idx]) {
        visited[idx] = 1;
        queue.push([cx, cy]);
      }
    }
    
    // BFS to find all connected background (near-white) pixels
    while (queue.length > 0) {
      const [x, y] = queue.shift()!;
      const idx = getIdx(x, y);
      
      const r = data[idx];
      const g = data[idx+1];
      const b = data[idx+2];
      
      // Color threshold: if close to white, make transparent and traverse
      if (r > 240 && g > 240 && b > 240) {
        data[idx+3] = 0; // Alpha = 0
        
        const neighbors = [
          [x - 1, y],
          [x + 1, y],
          [x, y - 1],
          [x, y + 1]
        ];
        
        for (const [nx, ny] of neighbors) {
          if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
            const nidx = ny * width + nx;
            if (!visited[nidx]) {
              visited[nidx] = 1;
              queue.push([nx, ny]);
            }
          }
        }
      }
    }
    
    ctx.putImageData(imgData, 0, 0);
    processedTunaSrc.value = canvas.toDataURL('image/png');
  };
});
</script>

<template>
  <div class="fullscreen-bg">
    <div class="conveyor-container">
      <!-- Conveyor Belt Background -->
      <img src="/conveyor.png" alt="Conveyor Belt" class="conveyor-belt" />
      
      <!-- Moving Tuna Cans -->
      <div v-if="processedTunaSrc" class="cans-container">
        <div 
          v-for="i in 6" 
          :key="i" 
          class="tuna-can"
          :style="{ animationDelay: `${(i - 1) * 3}s` }"
        >
          <img :src="processedTunaSrc" alt="Tuna Can" />
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.fullscreen-bg {
  position: relative;
  width: 100vw;
  height: 100vh;
  background-image: url('/factory-bg.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  overflow: hidden;
}

.conveyor-container {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 120px; /* Good height for the conveyor belt */
  display: flex;
  align-items: flex-end;
  z-index: 10;
}

.conveyor-belt {
  width: 100%;
  height: 100%;
  object-fit: fill; /* Stretch horizontally to cover the bottom screen */
  filter: drop-shadow(0 -8px 12px rgba(0, 0, 0, 0.4)); /* Adds a shadow to give a 3D depth effect against the background */
}

.cans-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  pointer-events: none;
}

.tuna-can {
  position: absolute;
  bottom: 15px; /* Adjust this value so the can sits precisely on the guide rail track */
  height: 52px; /* Proportional size of the can compared to the conveyor belt */
  transform: translateX(-150px);
  animation: moveCan 18s linear infinite;
}

.tuna-can img {
  height: 100%;
  width: auto;
  object-fit: contain;
  filter: drop-shadow(2px 5px 4px rgba(0, 0, 0, 0.35)); /* Adds a shadow to the can itself */
}

@keyframes moveCan {
  0% {
    transform: translateX(-150px);
  }
  100% {
    transform: translateX(100vw);
  }
}
</style>
