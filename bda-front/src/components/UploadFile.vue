<template>
  <el-upload
    accept="*/*"
    :before-upload="beforeUpload"
    :http-request="upload"
    :show-file-list="false"
    :disabled="disabled"
    style="display: inline-block"
    class="m-x-12"
  >
    <el-tooltip placement="bottom">
      <template #content>
        可上传本地文件
      </template>
      <el-button type="primary" style="font-size: 12px" :disabled="disabled">
        上传文件
      </el-button>
    </el-tooltip>
  </el-upload>
  <el-dialog
    v-model="dialogVisible"
    :fullscreen="true"
    :show-close="false"
    custom-class="dispute-upload-dialog"
  >
    <div class="center">
      <div class="fz-18 ellipsis">正在上传：{{ fileData.name }}</div>
      <el-progress :text-inside="true" :stroke-width="16" :percentage="percentage" />
      <el-button @click="cancel">取消上传</el-button>
    </div>
  </el-dialog>
</template>

<script setup lang="ts">
import axios from 'axios'
import { ElMessage, ElMessageBox } from 'element-plus'
import { ref } from 'vue'

const beforeUpload = (file: File) => {
  if (file.size / 1024 / 1024 / 1024 > 1.5) {
    ElMessage.error('文件大小不能超过 1.5G')
    ElMessage({
      type: 'error',
      message: '文件大小不能超过 1.5G',
      duration: 6000
    })
    return false
  }
  return true
}

const dialogVisible = ref(false)
const cancelUpload = ref(false)
let controller: AbortController | null = null
const chunkSize = 1 * 1024 * 1024 // 切片大小
const percentage = ref(0)
const fileData = ref({
  name: '',
  size: 0,
  type: '',
  suffix: '',
  md5: ''
})
const cancel = () => {
  dialogVisible.value = false
  cancelUpload.value = true
  controller?.abort()
  axios.post('cancelUpload', { folder: fileData.value.md5 })
}

let counter = 0
const getFileMd5 = () => {
  let guid = (+new Date()).toString(32)
  for (let i = 0; i < 5; i++) {
    guid += Math.floor(Math.random() * 65535).toString(32)
  }
  return 'wu_' + guid + (counter++).toString(32)
}

const updateUrl = (fileUrl: string) => {
  axios.post('saveUrl', {
    fileName: fileData.value.name,
    fileUrl
  })
}

const uploadChunkFile = async (i: number, fileObj: File) => {
  const start = i * chunkSize
  const end = Math.min(fileData.value.size, start + chunkSize)
  const chunkFile = fileObj.slice(start, end)
  const formData = new FormData()
  formData.append('encrypt', 'true')
  formData.append('fileName', fileData.value.name)
  formData.append('folder', fileData.value.md5)
  formData.append('file', chunkFile, String(i + 1))
  controller = new AbortController()
  return await axios
    .post('mergeUpload', formData, {
      onUploadProgress: (data) => {
        percentage.value = Number(
          (
            (Math.min(fileData.value.size, start + data.loaded) / fileData.value.size) *
            100
          ).toFixed(2)
        )
      },
      signal: controller.signal
    })
    .then((res) => updateUrl(res.data))
}

const batchUpload = async (fileObj: File) => {
  percentage.value = 0
  dialogVisible.value = true
  cancelUpload.value = false
  const chunkCount = Math.ceil(fileData.value.size / chunkSize) // 切片数量
  fileData.value.md5 = getFileMd5() // 文件唯一标识
  for (let i = 0; i < chunkCount; i++) {
    if (cancelUpload.value) return
    const res = await uploadChunkFile(i, fileObj)
    if (res.code !== 0) {
      dialogVisible.value = false
      ElMessageBox({ message: `${fileData.value.name}上传失败`, title: '提示' })
      return
    }
    if (i === chunkCount - 1) {
      setTimeout(() => {
        dialogVisible.value = false
        ElMessageBox({ message: `${fileData.value.name}上传成功`, title: '提示' })
        axios.post('mergeUpload', { folder: fileData.value.md5 }).then((res) => updateUrl(res.data))
      }, 500)
    }
  }
}

const disabled = ref(false)
const upload = async (file: { file: File }) => {
  const fileObj = file.file
  const nameList = fileObj.name.split('.')
  fileData.value.name = fileObj.name
  fileData.value.size = fileObj.size
  fileData.value.type = fileObj.type
  fileData.value.suffix = nameList[nameList.length - 1]
  if (chunkSize > fileData.value.size) {
    disabled.value = true
    axios
      .post('upload', fileObj)
      .then((res) => {
        ElMessageBox({ message: `${fileData.value.name}上传成功`, title: '提示' })
        updateUrl(res.data)
      })
      .catch(() => ElMessageBox({ message: `${fileData.value.name}上传失败`, title: '提示' }))
      .finally(() => (disabled.value = false))
    return
  }
  batchUpload(fileObj)
}
</script>

<style lang="scss">
.dispute-upload-dialog {
  background: none;
}
</style>

<style lang="scss" scoped>
.center {
  color: #fff;
  width: 50%;
  text-align: center;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
</style>
