<!--
 * @Author: 羊驼
 * @Date: 2023-10-27 16:15:23
 * @LastEditors: 羊驼
 * @LastEditTime: 2023-10-31 14:45:43
 * @Description: file content
-->
<template>
  <div id="app">
    <el-upload
      class="upload-demo"
      action="https://jsonplaceholder.typicode.com/posts/"
      :auto-upload="false"
      :limit="1"
      :on-change="initModel"
      accept=".fbx"
      :show-file-list="false"
    >
      <el-button
        size="small"
        type="primary"
      >点击上传模型文件</el-button>
    </el-upload>
    <fbx-viewer
      :type="type"
      :file="file.raw"
      ref="model"
      v-if="file"
    />
  </div>
</template>

<script>
import FbxViewer from "./components/FbxViewer.vue";
export default {
  components: { FbxViewer },
  name: "App",
  data() {
    return {
      type: "file",
      file: null,
    };
  },
  methods: {
    initModel(file) {
      if (file != this.file) {
        this.file = file;
        this.$nextTick(() => {
          this.$refs.model.init();
        });
      }
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}
</style>
