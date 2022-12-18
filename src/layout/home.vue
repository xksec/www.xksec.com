<template>
  <section class="banner" id="home">
    <div class="textBox">
      <h2>大家好，我是</h2>
      <h2><span>君自强</span></h2>
      <h3>专注于安全领域的全栈工程师</h3>
      <a href="#about" class="btn">关于我</a>
    </div>
  </section>
</template>

<style scoped>
section {
  position: relative;
  width: 100%;
  height: 100vh;
  background: url("../assets/banner.jpg");
  background-size: cover;
  background-position: right;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

section .textBox h2 {
  font-size: 3em;
  font-weight: 500;
  line-height: 1.5em;
  color: #fff;
}

section .textBox {
  display: flex;
  flex-direction: column;
  align-items: center;
}

section .textBox h2 span {
  font-size: 1.5em;
  font-weight: 700;
}

section .textBox h3 {
  color: #fff;
  font-size: 1.5em;
  font-weight: 500;
}

.btn {
  position: relative;
  background: #2196f3;
  display: inline-block;
  color: #fff;
  margin-top: 20px;
  padding: 10px 30px;
  text-transform: uppercase;
  text-decoration: none;
  letter-spacing: 2px;
  font-weight: 500;
  border-radius: 5px;
}
</style>
