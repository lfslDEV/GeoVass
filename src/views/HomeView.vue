<template>
    <div class="home">
      <Map />
    </div>
  </template>
  
  <script>
  import Map from '../components/Map.vue';
  
  export default {
    name: 'HomeView',
    components: {
      Map,
    }
  };
  </script>