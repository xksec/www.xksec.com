<template>
  <headerVue></headerVue>
  <homeVue></homeVue>
  <aboutVue></aboutVue>
  <copyrightVue></copyrightVue>
</template>

<script>
import headerVue from "./layout/header.vue";
import homeVue from "./layout/home.vue";
import aboutVue from "./layout/about.vue";
import copyrightVue from "./layout/copyright.vue";
export default {
  name: "App",
  components: {
    headerVue,
    homeVue,
    aboutVue,
    copyrightVue,
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  scroll-behavior: smooth;
}

html,
body,
#app {
  margin: 0;
  width: 100%;
  height: 100%;
}
</style>
