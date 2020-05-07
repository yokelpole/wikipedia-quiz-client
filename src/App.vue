<template>
  <div id="app">
    <Quiz />
  </div>
</template>

<script>
import Quiz from "./components/Quiz.vue";

export default {
  name: "App",
  components: {
    Quiz
  },
};
</script>

<style>
html {
  background: black;
  max-width: 768px;
  text-align: center;
  margin-left: auto;
  margin-right: auto;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #c1d1e2;
  margin-top: 60px;
}
</style>
