<script setup>
const projects = [
  {
    name: '学科竞赛管理系统',
    description: '高校竞赛全流程管理：赛事目录、报名配置、学生报名、审核、获奖填报、赛事总结和数据统计。',
    href: '/compemanage/#/home',
  },
]

const contacts = [
  {
    label: 'GitHub',
    value: 'github.com/b1each232425',
    href: 'https://github.com/b1each232425',
    external: true,
  },
  {
    label: '邮箱',
    value: 'T051521@outlook.com',
    href: 'mailto:T051521@outlook.com',
  },
]
</script>

<template>
  <main class="page">
    <section class="profile">
      <h1>叶蔚轩的项目展示</h1>
      <p class="intro">这里整理一些可以在线访问的项目演示，方便快速查看功能、代码和部署效果。</p>
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

    <section class="contacts" aria-label="联系方式">
      <h2>联系方式</h2>
      <div class="contact-list">
        <a
          v-for="contact in contacts"
          :key="contact.label"
          :href="contact.href"
          :target="contact.external ? '_blank' : undefined"
          :rel="contact.external ? 'noreferrer' : undefined"
        >
          <span>{{ contact.label }}</span>
          <strong>{{ contact.value }}</strong>
        </a>
      </div>
    </section>
  </main>
</template>
