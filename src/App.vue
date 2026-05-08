<script lang="ts">
import TextEditor from './components/TextEditor.vue'

export default {
  name: 'App',
  components: {
    TextEditor
  },
  methods: {
    handleEditorChange(changeData: any) {
      console.log('Editor change:', changeData)
    }
  }
}
</script>

<template>
  <TextEditor @editor-change="handleEditorChange" />
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

#app {
  width: 100%;
  height: 100vh;
  overflow: hidden;
}
</style>
