<template>
  <section>
    <span>
      Copyright @ {{ new Date().getFullYear() }} by XKSEC.COM, All Rights
      Reserved.
    </span>
    <a href="http://beian.miit.gov.cn/" target="_blank">京ICP备XXXXX号</a>
  </section>
</template>

<style scoped>
section {
  background: #111;
  width: 100%;
  height: 30px;
  font-size: 16px;
  font-weight: 500;
  display: flex;
  justify-content: center;
}

section span,
section a {
  margin: 0 20px;
  color: #fff;
  text-decoration: none;
  padding: 4px 0;
}
</style>
