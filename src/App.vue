<template>
  <div>
    <nav-bar></nav-bar>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.3/css/all.min.css">
  </div>
</template>

<script>
import Vue from 'vue'
import VueMaterial from 'vue-material'
import NavBar from './components/NavBar.vue';
Vue.use(VueMaterial)

export default {
  components: { NavBar },
  name: 'App'
}
</script>

<style>
</style>
