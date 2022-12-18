<template>
  <section id="about">

    <div class="content">
      <h2>关于我</h2>
      <div class="label">
        <span class="name">君自强</span>
        <span class="sublabel">专注于安全领域的全栈工程师</span>
      </div>
      <div class="desc">
        这是一段自我介绍,这是一段自我介绍,这是一段自我介绍, <br>
        这是一段自我介绍,这是一段自我介绍,这是一段自我介绍, <br>
        这是一段自我介绍,这是一段自我介绍,这是一段自我介绍, <br>
        这是一段自我介绍,这是一段自我介绍,这是一段自我介绍, <br>
        这是一段自我介绍,这是一段自我介绍,这是一段自我介绍, <br>
        这是一段自我介绍,这是一段自我介绍,这是一段自我介绍, <br>
        这是一段自我介绍,这是一段自我介绍,这是一段自我介绍, <br>
        这是一段自我介绍,这是一段自我介绍,这是一段自我介绍, <br>
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

#about{
  display: flex;
  justify-content: center;
  align-items: center;
}

#about .mypic,
#about .content{
  width: 500px;
  margin: 40px;
}

.mypic img {
  border-radius: 20px;
  width: 500px;
  height: 600px;
}

.content h2{
  font-size: 3em;
  font-weight: 700;
}

.label .name{
  font-size: 1.5em;
  font-weight: 500;
  margin-right: 30px;
}

.label .sublabel{
  font-size: 1.2em;
  font-weight: 400;
}

.content .desc{
  font-size: 1em;
  margin-top: 40px;
}
</style>
