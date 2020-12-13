<template>
  <div id="app">
    <div>
      <input type="file" @change="handleFileChange">
      <el-button @click="handleUpload">上传</el-button>
    </div>
    <div>
      <div>计算文件 hash</div>
      <el-progress :percentage="hashPercentage"></el-progress>
      <div>总进度</div>
      <el-progress :percentage="fakeUploadPercentage"></el-progress>
    </div>
    <el-table :data="data">
      <el-table-column prop="hash" label="切片hash" align="center"></el-table-column>
      <el-table-column label="大小(KB)" align="center" width="120">
        <template v-slot="{ row }">
          {{ row.size | transformByte }}
        </template>
      </el-table-column>
      <el-table-column label="进度" align="center">
        <template v-slot="{ row }">
          <el-progress
            :percentage="row.percentage"
            color="#909399"
          ></el-progress>
        </template>
      </el-table-column>
    </el-table>
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
      data: [],
      fakeUploadPercentage: 0,
      hashPercentage: 0
    }
  },
  filters: {
    transformByte: function(value) {
      return Number((value / 1024).toFixed(0));
    }
  },
  watch: {
    uploadPercentage(now) {
      if(now > this.fakeUploadPercentage) {
        this.fakeUploadPercentage = now;
      }
    }
  },
  computed: {
    uploadPercentage() {
      if(!this.container.file || !this.data.length) return 0;
      const loaded = this.data
        .map(item => item.size * item.percentage)
        .reduce((acc, cur) => acc + cur);
      return parseInt((loaded / this.container.file.size).toFixed(2));
    }
  },
  methods: {
    request({url, method="post", data, headers={},onProgress = e => e, requestList}) {
      return new Promise(resolve => {
        const xhr = new XMLHttpRequest();
        xhr.upload.onprogress = onProgress;
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
        .map(({chunk, hash, index}) => {
          const formData = new FormData();
          formData.append('chunk', chunk);
          formData.append('hash', hash);
          formData.append('filename', this.container.file.name);
          return { formData, index };
        })
        .map(async ({formData, index}) => 
          this.request({
            url: 'http://localhost:3000',
            data: formData,
            onProgress: this.createProgressHandler(this.data[index])
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
    calculateHash(fileChunkList) {
      return new Promise(resolve => {
        this.container.worker = new Worker('/hash.js');
        this.container.worker.postMessage({fileChunkList});
        this.container.worker.onmessage = e => {
          const {percentage, hash} = e.data;
          this.hashPercentage = percentage;
          if(hash) {
            resolve(hash);
          }
        }
      })
    },
    async handleUpload() {
      if(!this.container.file) return;
      const fileChunkList = this.createFileChunk(this.container.file);    // [{file: 切片}]
      this.container.hash = await this.calculateHash(fileChunkList);
      this.data = fileChunkList.map(({file}, index) => ({
        fileHash: this.container.hash,
        size: file.size,
        chunk: file,
        index,
        hash: this.container.file.name + '-' + index,
        percentage: 0
      }));
      await this.uploadChunks();
    },
    createProgressHandler(item) {
      return e => {
        item.percentage = parseInt(String((e.loaded / e.total) * 100));
      }
    }
  },
}
</script>

<style>
</style>
