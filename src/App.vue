<script setup>
const projects = [
  {
    name: '学科竞赛管理系统',
    description: '高校竞赛全流程管理：赛事目录、报名配置、学生报名、审核、获奖填报、赛事总结和数据统计。',
    href: '/compemanage/#/home',
  },
]
</script>

<template>
  <main class="page">
    <section class="profile">
      <h1>叶蔚轩的项目展示</h1>
      <p class="intro">这里集中放置可以在线演示的个人项目，目前包含学科竞赛管理系统。</p>
    </section>

    <section class="projects" aria-label="项目列表">
      <article v-for="project in projects" :key="project.name" class="project">
        <div>
          <h2>{{ project.name }}</h2>
          <p>{{ project.description }}</p>
        </div>
        <a :href="project.href">进入演示</a>
      </article>
    </section>
  </main>
</template>
