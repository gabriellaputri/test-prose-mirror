<template>
  <div>
    <div @click="init">
      <Editor />
    </div>

    <div id="y-version"></div>
  </div>
</template>

<script>
import Editor from "./components/Editor";

export default {
  name: "App",
  data() {
    return {
      content: "hahah",
      isEditor: false,
    };
  },
  components: {
    Editor,
  },
  methods: {
    init(e) {
      console.log(e.target, e.currentTarget);
    },
    destroy() {},
  },
};
</script>
