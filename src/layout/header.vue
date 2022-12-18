<template>
  <div class="header">
    <!-- <img src="../assets/logo.png" /> -->
    <div class="logo">
      <img class="logo-image" src="../assets/logo-white.png" />
      <img class="logo-name" src="../assets/logo-name.png" />
    </div>

    <ul>
      <li><a href="#home">Home</a></li>
      <li><a href="#about">About</a></li>
    </ul>
  </div>
</template>

<script></script>

<style scoped>
.header {
  width: 100%;
  position: fixed;
  display: flex;
  justify-content: space-between;
  max-height: 78px;
  height: 78px;
  padding: 0 20px 0 20px;
  z-index: 1000;
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

li a {
  text-decoration: none;
  color: #fff;
  font-size: 20px;
  font-weight: 600;
}
</style>
