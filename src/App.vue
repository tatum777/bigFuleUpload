<template>
  <div id="app">
    <input type="file" @change="handleFileChange">
    <el-button @click="handleUpload">上传</el-button>
  </div>
</template>

<script>
const SIZE = 10 * 1024 * 1024;

export default {
  name: 'App',
  data() {
    return {
      container: {
        file: null
      },
      data: []
    }
  },
  methods: {
    request({url, method="post", data, headers={}, requestList}) {
      return new Promise(resolve => {
        const xhr = new XMLHttpRequest();
        xhr.open(method, url);
        Object.keys(headers).forEach(key =>
          xhr.setRequestHeader(key, headers[key])
        );
        xhr.send(data);
        xhr.onload = e => {
          resolve({
            data: e.target.response
          })
        }
      })
    },
    handleFileChange(e) {
      const [file] = e.target.files;
      if(!file) return;
      Object.assign(this.$data, this.$options.data());
      this.container.file = file;
    },
    createFileChunk(file, size = SIZE) {
      const fileChunkList = [];
      let cur = 0;
      while(cur < file.size) {
        fileChunkList.push({file: file.slice(cur, cur + size)});
        cur += size;
      }
      return fileChunkList;
    },
    async uploadChunks() {
      const requestList = this.data
        .map(({chunk, hash}) => {
          const formData = new FormData();
          formData.append('chunk', chunk);
          formData.append('hash', hash);
          formData.append('filename', this.container.file.name);
          return { formData };
        })
        .map(async ({formData}) => 
          this.request({
            url: 'http://localhost:3000',
            data: formData
          })
        )
      await Promise.all(requestList);
      await this.mergeRequest();
    },
    async mergeRequest() {
      await this.request({
        url: 'http://localhost:3000/merge',
        headers: {
          'content-type': 'application/json'
        },
        data: JSON.stringify({
          filename: this.container.file.name,
          size: SIZE
        })
      })
    },
    async handleUpload() {
      if(!this.container.file) return;
      const fileChunkList = this.createFileChunk(this.container.file);    // [{file: 切片}]
      this.data = fileChunkList.map(({file}, index) => ({
        chunk: file,
        hash: this.container.file.name + '-' + index
      }));
      await this.uploadChunks();
    }
  },
}
</script>

<style>
</style>
