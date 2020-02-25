<template>
  <main>
    <h1>{{ message }}</h1>
    <p>
      <nuxt-link to="/">
        重复来自客户端的API请求
      </nuxt-link>
    </p>
  </main>
</template>

<script>
export default {
  async asyncData ({ $api }) {
    const { data } = await $api.hello.msg()
    return data
  }
}
</script>
