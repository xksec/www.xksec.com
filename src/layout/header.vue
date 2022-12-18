<template>
  <div class="header" :class="{sticky:windowTop>0 }">
    <!-- <img src="../assets/logo.png" /> -->
    <div class="logo">
        <!-- <img class="logo-image"  src="../assets/logo-white.png" /> -->
        <img class="logo-image" :src="windowTop > 0 ? require('@/assets/logo-black.png') : require('@/assets/logo-white.png')" >
        <!-- <img class="logo-image" :src="windowTop>0? require('../assets/logo-black.png')" /> -->
      <img class="logo-name" :src="windowTop > 0 ? require('@/assets/logo-name-black.png') : require('@/assets/logo-name.png')" />
    </div>

    <ul>
      <li><a href="#home">Home</a></li>
      <li><a href="#about">About</a></li>
    </ul>
  </div>
</template>

<script>
export default {
  data() {
    return { windowTop: 0 };
  },
  mounted() {
    window.addEventListener("scroll", this.onScroll, true);
  },
  beforeUnmount() {
    window.removeEventListener("scroll", this.onScroll, true);
  },
  methods: {
    onScroll() {
      this.windowTop = window.top.scrollY ;
    },
  },
};
</script>

<style scoped>
.header {
  width: 100%;
  position: fixed;
  display: flex;
  justify-content: space-between;
  max-height: 78px;
  height: 78px;
  padding: 0 100px 0 100px;
  z-index: 1000;
  transition: all 0.5s ease ;
}

.header.sticky{
    height: 60px;
    max-height: 60px;
    background: #fff;
}

.logo {
  display: flex;
  width: 200px;
  justify-content: space-between;
  align-items: center;
}
.logo-image {
  width: 48px;
  height: 60px;
  padding: 0 0 10px;
}
.logo-name {
  width: 140px;
  height: 40px;
  padding: 10px 0 0;
}

.header ul {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
}

.header ul li {
  list-style: none;
  display: inline-block;
  margin: 0 20px;
}

.header li a {
  text-decoration: none;
  color: #fff;
  font-size: 20px;
  font-weight: 600;
}

.header.sticky li a{
    color: #111;
}

</style>
