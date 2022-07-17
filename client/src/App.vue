<template>
  <div id="app">
    <div>
      <input
        type="file"
        :disabled="status !== Status.wait"
        @change="handleFileChange"
      />
      <button @click="handleUpload" :disabled="uploadDisabled"
        >upload</button
      >
      <button @click="handleResume" v-if="status === Status.pause"
        >resume</button
      >
      <button
        v-else
        :disabled="status !== Status.uploading || !container.hash"
        @click="handlePause"
        >pause</button
      >
      <button @click="handleDelete">delete</button>
    </div>
    <h5>{{ message }}</h5>
    <div style = 'display: none'>
      <div>
        <div>calculate chunk hash</div>
        <progress :percentage="hashPercentage"></progress>
      </div>
      <div>
        <div>percentage</div>
        <progress :percentage="fakeUploadPercentage"></progress>
      </div>
    </div>
    <!-- <el-table :data="data">
      <el-table-column
        prop="hash"
        label="chunk hash"
        align="center"
      ></el-table-column>
      <el-table-column label="size(KB)" align="center" width="120">
        <template v-slot="{ row }" v-if = "row">
          {{ row.size | transformByte }}
        </template>
      </el-table-column>
      <el-table-column label="percentage" align="center">
        <template v-slot="{ row }">
          <progress
            :percentage="row.percentage"
            color="#909399"
          ></progress>
        </template>
      </el-table-column>
    </el-table> -->
  </div>
</template>

<script>
// 切片大小
// chunk size
const SIZE = 10 * 1024 * 1024;

const Status = {
  wait: 'wait',
  pause: 'pause',
  uploading: 'uploading',
};

export default {
  data: () => ({
    Status,
    container: {
      file: null,
      hash: '',
      worker: null,
    },
    hashPercentage: 0,
    data: [], // 切片
    requestList: [],
    status: Status.wait,
    // 當暂停时会取消 xhr 導進度條後退
    // 為了避免這種情況，需要定一一个假的進度條
    // use fake progress to avoid progress backwards when upload is paused
    fakeUploadPercentage: 0,
    message: '',
  }),
  filters: {
    transformByte(val) {
      return Number((val / 1024).toFixed(0));
    },
  },
  computed: {
    uploadDisabled() {
      return (
        !this.container.file
        || [Status.pause, Status.uploading].includes(this.status)
      );
    },
    uploadPercentage() {
      if (!this.container.file || !this.data.length) return 0;
      const loaded = this.data
        .map((item) => {
          const it = item;
          return it.size * it.percentage;
        })
        .reduce((acc, cur) => acc + cur);
      return parseInt((loaded / this.container.file.size).toFixed(2), 10);
    },
  },
  watch: {
    uploadPercentage(now) {
      if (now > this.fakeUploadPercentage) {
        this.fakeUploadPercentage = now;
      }
    },
  },
  methods: {
    async handleDelete() {
      const { data } = await this.request({
        url: 'http://localhost:3000/delete',
      });
      if (JSON.parse(data).code === 0) {
        this.$message.success('delete success');
      }
    },
    handlePause() {
      this.status = Status.pause;
      this.resetData();
    },
    resetData() {
      this.requestList.forEach((xhr) => xhr?.abort());
      this.requestList = [];
      if (this.container.worker) {
        this.container.worker.onmessage = null;
      }
    },
    async handleResume() {
      this.status = Status.uploading;
      const { uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash,
      );
      await this.uploadChunks(uploadedList);
    },
    // xhr
    request({
      url,
      method = 'post',
      data,
      headers = {},
      onProgress = (e) => e,
      requestList = [],
    }) {
      return new Promise((resolve) => {
        const xhr = new XMLHttpRequest();
        xhr.upload.onprogress = onProgress;
        xhr.open(method, url);
        Object.keys(headers).forEach((key) => xhr.setRequestHeader(key, headers[key]));
        xhr.send(data);
        xhr.onload = (e) => {
          // 將請求成功的 xhr 从列表中删除
          // remove xhr which status is success
          if (requestList) {
            const xhrIndex = requestList.findIndex((item) => item === xhr);
            requestList.splice(xhrIndex, 1);
          }
          resolve({
            data: e.target.response,
          });
        };
        // 暴露當前 xhr 给外部
        // export xhr
        requestList.push(xhr);
      });
    },
    // 生成文件切片
    // create file chunk
    createFileChunk(file, size = SIZE) {
      const fileChunkList = [];
      let cur = 0;
      while (cur < file.size) {
        fileChunkList.push({ file: file.slice(cur, cur + size) });
        cur += size;
      }
      return fileChunkList;
    },
    // 生成文件 hash（web-worker）
    // use web-worker to calculate hash
    calculateHash(fileChunkList) {
      return new Promise((resolve) => {
        this.container.worker = new Worker('/hash.js');
        // 傳送{fileChunkList}給/hash.js
        this.container.worker.postMessage({ fileChunkList });
        // 接收/hash.js傳送的物件
        // 計算出一個文件的hash值
        this.container.worker.onmessage = (e) => {
          const { percentage, hash } = e.data;
          this.hashPercentage = percentage;
          if (hash) {
            // 回傳切片計算出的hash值
            resolve(hash);
          }
        };
      });
    },
    handleFileChange(e) {
      const [file] = e.target.files;
      if (!file) return;
      this.resetData();
      Object.assign(this.$data, this.$options.data());
      this.container.file = file;
    },
    async handleUpload() {
      if (!this.container.file) return;
      this.status = Status.uploading;
      // arr = [切片1, 切片2....]
      const fileChunkList = this.createFileChunk(this.container.file);
      // 計算出一個文件的hash值
      this.container.hash = await this.calculateHash(fileChunkList);

      const { shouldUpload, uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash,
      );
      if (!shouldUpload) {
        // this.$message.success('skip upload：file upload success');
        this.message = '秒傳: 上傳成功';
        this.status = Status.wait;
        return;
      }

      this.data = fileChunkList.map(({ file }, index) => ({
        fileHash: this.container.hash,
        index,
        hash: `${this.container.hash}-${index}`,
        chunk: file,
        size: file.size,
        percentage: uploadedList.includes(index) ? 100 : 0,
      }));

      await this.uploadChunks(uploadedList);
    },
    // 上傳切片，同時過濾已上傳的切片
    // upload chunks and filter uploaded chunks
    async uploadChunks(uploadedList = []) {
      const requestList = this.data
        .filter(({ hash }) => !uploadedList.includes(hash))
        .map(({ chunk, hash, index }) => {
          const formData = new FormData();
          formData.append('chunk', chunk);
          formData.append('hash', hash);
          formData.append('filename', this.container.file.name);
          formData.append('fileHash', this.container.hash);
          return { formData, index };
        })
        .map(({ formData, index }) => this.request({
          url: 'http://localhost:3000',
          data: formData,
          onProgress: this.createProgressHandler(this.data[index]),
          requestList: this.requestList,
        }));
      await Promise.all(requestList);
      // 之前上傳的切片數量 + 本次上傳的切片數量 = 所有切片數量時合併切片
      // merge chunks when the number of chunks uploaded before and
      // the number of chunks uploaded this time
      // are equal to the number of all chunks
      if (uploadedList.length + requestList.length === this.data.length) {
        await this.mergeRequest();
      }
    },
    // 通知服務端合併切片
    // notify server to merge chunks
    async mergeRequest() {
      await this.request({
        url: 'http://localhost:3000/merge',
        headers: {
          'content-type': 'application/json',
        },
        data: JSON.stringify({
          size: SIZE,
          fileHash: this.container.hash,
          filename: this.container.file.name,
        }),
      });
      this.message = '上傳成功';
      this.status = Status.wait;
    },
    // 根據 hash 驗證文件是否曾经已经被上傳過
    // 没有才進行上傳
    // verify that the file has been uploaded based on the hash
    // skip if uploaded
    async verifyUpload(filename, fileHash) {
      const { data } = await this.request({
        url: 'http://localhost:3000/verify',
        headers: {
          'content-type': 'application/json',
        },
        data: JSON.stringify({
          filename,
          fileHash,
        }),
      });
      return JSON.parse(data);
    },
    // 用閉包保存每個 chunk 的進度數據
    // use closures to save progress data for each chunk
    createProgressHandler(item) {
      const it = item;
      return (e) => {
        it.percentage = parseInt(String((e.loaded / e.total) * 100), 10);
      };
    },
  },
};
</script>
