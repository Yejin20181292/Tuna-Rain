<template>
  <div class="fullscreen-bg">
    <div class="conveyor-container">
      <img src="/conveyor.png" alt="Conveyor Belt" class="conveyor-belt" />
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
  pointer-events: none; /* Let clicks pass through if needed */
}

.conveyor-belt {
  width: 100%;
  height: 100%;
  object-fit: fill; /* Stretch horizontally to cover the bottom screen */
  filter: drop-shadow(0 -8px 12px rgba(0, 0, 0, 0.4)); /* Adds a shadow to give a 3D depth effect against the background */
}
</style>
