<template>
  <section id="about">
    <div class="content">
      <h2>关于我</h2>
      <div class="label">
        <span class="name">君自强</span>
        <span class="sublabel">专注于安全领域的全栈工程师</span>
      </div>
      <div class="desc">
        本人-安宏奎，笔名君自强，具备20年左右的工作经验，擅长与服务端研发、安全产品研发等，先后供职于多家互联网大厂。
        亲手创造出WAF、KMS、CA体系、加密网关、堡垒机等产品。
        现正在努力学习前端相关的技术，希望能够利用自己的技术优势，发展一点点副业。
      </div>
      <div class="desc">
        本人比较喜欢的一句话：天行健，君子以自强不息；地势坤，君子以厚德载物。
      </div>
    </div>

    <div class="mypic">
      <img src="../assets/mypic.png" alt="" />
    </div>
  </section>
</template>

<style scoped>
section {
  width: 100%;
  height: 100vh;
  background: #fff;
}

#about {
  display: flex;
  justify-content: center;
  align-items: center;
}

#about .mypic,
#about .content {
  width: 500px;
  margin: 40px;
}

.mypic img {
  border-radius: 20px;
  width: 500px;
  height: 600px;
}

.content h2 {
  font-size: 3em;
  font-weight: 700;
}

.label .name {
  font-size: 1.5em;
  font-weight: 500;
  margin-right: 30px;
}

.label .sublabel {
  font-size: 1.2em;
  font-weight: 400;
  color: chocolate;
}

.content .desc {
  font-size: 1em;
  margin-top: 40px;
  text-indent: 2em;
  max-width: 400px;
}

@media (max-width: 1066px) {
  .label {
    display: flex;
    flex-direction: column;
  }
}
</style>
