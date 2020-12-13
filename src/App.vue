<template>
  <div id="app">
    <div>
      <input type="file" @change="handleFileChange" :disabled="status !== Status.wait">
      <el-button @click="handleUpload" :disabled="uploadDisabled">上传</el-button>
      <el-button v-if="status === Status.pause" @click="handleResume">恢复</el-button>
      <el-button v-else @click="handlePause" :disabled="status !== Status.uploading || !container.hash">暂停</el-button>
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

const Status = {
  wait: 'wait',
  pause: 'pause',
  uploading: 'uploading'
}

export default {
  name: 'App',
  filters: {
    transformByte: function(value) {
      return Number((value / 1024).toFixed(0));
    }
  },
  data() {
    return {
      Status,
      container: {
        file: null,
        hash: '',
        worker: null
      },
      hashPercentage: 0,
      data: [],
      requestList: [],
      status: Status.wait,
      fakeUploadPercentage: 0,
    }
  },
  computed: {
    uploadDisabled() {
      return (
        !this.container.file ||
        [Status.pause, Status.uploading].includes(this.status)
      )
    },
    uploadPercentage() {
      if(!this.container.file || !this.data.length) return 0;
      const loaded = this.data
        .map(item => item.size * item.percentage)
        .reduce((acc, cur) => acc + cur);
      return parseInt((loaded / this.container.file.size).toFixed(2));
    }
  },
  watch: {
    uploadPercentage(now) {
      if(now > this.fakeUploadPercentage) {
        this.fakeUploadPercentage = now;
      }
    }
  },
  methods: {
    handlePause() {
      this.status = Status.pause;
      this.resetData();
    },
    resetData() {
      this.requestList.forEach(xhr => xhr.abort());
      this.requestList = [];
      if(this.container.worker) {
        this.container.worker.onmessage = null;
      }
    },
    async handleResume() {
      this.status = Status.uploading;
      const { uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash
      );
      await this.uploadChunks(uploadedList);
    },
    request({url, method="post", data, headers={}, onProgress = e => e, requestList}) {
      return new Promise(resolve => {
        const xhr = new XMLHttpRequest();
        xhr.upload.onprogress = onProgress;
        xhr.open(method, url);
        Object.keys(headers).forEach(key =>
          xhr.setRequestHeader(key, headers[key])
        );
        xhr.send(data);
        xhr.onload = e => {
          if(requestList) {
            const xhrIndex = requestList.findIndex(item => item === xhr);
            requestList.splice(xhrIndex, 1);
          }
          resolve({
            data: e.target.response
          })
        }
        requestList && requestList.push(xhr);
      })
    },
    handleFileChange(e) {
      const [file] = e.target.files;
      if(!file) return;
      this.resetData();
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
    async uploadChunks(uploadedList = []) {
      const requestList = this.data
        .filter(({hash}) => !uploadedList.includes(hash))
        .map(({chunk, hash, index}) => {
          const formData = new FormData();
          formData.append('chunk', chunk);
          formData.append('hash', hash);
          formData.append('filename', this.container.file.name);
          formData.append('fileHash', this.container.hash);
          return { formData, index };
        })
        .map(async ({formData, index}) => 
          this.request({
            url: 'http://localhost:3000',
            data: formData,
            onProgress: this.createProgressHandler(this.data[index]),
            requestList: this.requestList
          })
        )
      await Promise.all(requestList);
      if(uploadedList.length + requestList.length === this.data.length) {
        await this.mergeRequest();
      }
    },
    async mergeRequest() {
      await this.request({
        url: 'http://localhost:3000/merge',
        headers: {
          'content-type': 'application/json'
        },
        data: JSON.stringify({
          size: SIZE,
          fileHash: this.container.hash,
          filename: this.container.file.name
        })
      })
      this.$message.success('上传成功');
      this.status = Status.wait;
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
      this.status = Status.uploading;
      const fileChunkList = this.createFileChunk(this.container.file);    // [{file: 切片}]
      this.container.hash = await this.calculateHash(fileChunkList);

      const { shouldUpload, uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash
      );
      if(!shouldUpload) {
        this.$message.success('上传成功');
        this.status = Status.wait;
        return;
      }
      this.data = fileChunkList.map(({file}, index) => ({
        fileHash: this.container.hash,
        size: file.size,
        chunk: file,
        index,
        hash: this.container.hash + '-' + index,
        percentage: uploadedList.includes(index) ? 100 : 0
      }));
      await this.uploadChunks(uploadedList);
    },
    async verifyUpload(filename, fileHash) {
      const { data } = await this.request({
        url: 'http://localhost:3000/verify',
        headers: {
          'content-type': 'application/json'
        },
        data: JSON.stringify({
          filename,
          fileHash
        })
      });
      return JSON.parse(data);
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
