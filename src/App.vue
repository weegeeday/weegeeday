<script setup>
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <div class="app-shell">
    <nav class="app-nav">
      <RouterLink class="nav-button" to="/">Home</RouterLink>
      <RouterLink class="nav-button" to="/about">About</RouterLink>
    </nav>

    <main class="app-view">
      <RouterView />
    </main>
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  width: 100%;
  position: relative;
  padding: clamp(4rem, 8vw, 6rem) clamp(1rem, 4vw, 3rem) clamp(1.5rem, 4vw, 3rem);
}

.app-nav {
  position: absolute;
  top: clamp(0.75rem, 2vw, 1.5rem);
  left: clamp(0.75rem, 2vw, 1.5rem);
  display: flex;
  gap: 0.5rem;
  z-index: 10;
}

.nav-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.35rem 0.75rem;
  border-radius: 999px;
  border: 1px solid transparent;
  background-color: color-mix(in srgb, var(--color-background) 86%, transparent);
  color: var(--color-text);
  font-size: 0.9rem;
  line-height: 1.2;
}

.nav-button.router-link-exact-active {
  border-color: var(--color-border-hover);
  background-color: color-mix(in srgb, var(--color-background-soft) 92%, transparent);
}

.nav-button:hover {
  background-color: color-mix(in srgb, var(--color-background-soft) 88%, transparent);
}

.app-view {
  min-height: calc(100vh - clamp(5.5rem, 10vw, 8rem));
  width: 100%;
}
</style>
