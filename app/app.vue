<template>
  <div class="app-bg">
    <NuxtPage />
  </div>
</template>

<style>
html, body, #__nuxt {
  height: 100%;
  margin: 0;   
  padding: 0;
}

.app-bg {
  min-height: 100svh;
  background: url('/resources/gradient-bg.jpg') center / cover no-repeat;
  display: flow-root;     
}

</style>
