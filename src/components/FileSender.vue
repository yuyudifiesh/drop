<template>
  <div class="sender-container">
    <v-card elevation="4" class="p-6">
      <v-card-title class="text-h5 mb-4">文件发送端</v-card-title>
      
      <!-- 文件选择区域 - 虚线矩形样式 -->
      <div 
        class="file-drop-area"
        @click="triggerFileInput"
        @dragover.prevent="isDragging = true"
        @dragleave.prevent="isDragging = false"
        @drop.prevent="handleDrop"
      >
        <input
          ref="fileInputRef"
          type="file"
          accept="*/*"
          style="display: none"
          @change="onFileSelected"
        />
        <div class="file-drop-content">
          <v-icon size="48" class="mb-2">mdi-file-upload</v-icon>
          <p class="mb-1">点击或拖拽文件到此处</p>
          <p class="text-xs text-muted">支持所有文件类型</p>
          <p v-if="selectedFile" class="mt-2 text-sm file-name">{{ selectedFile.name }}</p>
        </div>
      </div>

      <!-- 连接状态显示 -->
      <v-alert
        :type="connectionStatus === 'connected' ? 'success' : 'info'"
        dense
        class="mt-4 mb-4"
      >
        {{ connectionStatus === 'connected' ? '已连接，可以发送文件' : '等待连接...' }}
      </v-alert>

      <!-- 二维码显示区域 -->
      <div v-if="qrCodeData" class="qr-code-container text-center mb-4">
    <div class="qr-code" v-html="qrCodeData"></div>
    <p class="text-sm text-muted">请使用接收端扫描二维码</p>
    <!-- 按钮组 -->
    <div style="display: flex; justify-content: center; gap: 8px;" class="mt-2">
      <v-btn @click="showOfferDebug = !showOfferDebug" size="small">
        {{ showOfferDebug ? '隐藏调试信息' : '显示调试信息' }}
      </v-btn>
      <v-btn @click="generateOffer" size="small" color="primary">
        生成发送端Offer
      </v-btn>
    </div>
    <v-card v-if="showOfferDebug" class="mt-2 bg-gray-50 p-2 text-xs break-all">
      <pre>{{ offerSdp }}</pre>
    </v-card>
  </div>

      <!-- 手动输入answer区域 -->
      <v-textarea
        v-model="answerSdp"
        label="如果二维码扫描失败，请输入接收端的Answer"
        outlined
        rows="4"
        class="mb-4"
      ></v-textarea>

      <!-- 传输进度 -->
      <v-progress-linear
        v-if="isSending"
        v-model="progress"
        color="primary"
        height="8"
        rounded
        class="mb-2"
      ></v-progress-linear>
      <p v-if="isSending" class="text-sm text-center">{{ Math.round(progress) }}% 已发送</p>

      <!-- 按钮组 - 居中显示 -->
      <div class="action-buttons text-center">
        <v-btn @click="acceptAnswer" :disabled="!answerSdp || connectionStatus === 'connected'" color="primary" class="mb-2 mx-2">
          接受Answer
        </v-btn>
        
        <v-btn
          @click="sendFile"
          :disabled="!selectedFile || connectionStatus !== 'connected' || isSending"
          color="success"
          class="mb-2 mx-2"
        >
          {{ isSending ? '发送中...' : '发送文件' }}
        </v-btn>
        
        <v-btn @click="resetConnection" outlined color="error" class="mb-2 mx-2">
          重置连接
        </v-btn>
      </div>
    </v-card>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import QRCode from 'qrcode'

// WebRTC相关状态
const peerConnection = ref(null)
const dataChannel = ref(null)
const connectionStatus = ref('disconnected') // disconnected, connecting, connected

// 文件相关状态
const selectedFile = ref(null)
const isSending = ref(false)
const progress = ref(0)
const isDragging = ref(false)
const fileInputRef = ref(null)

// SDP相关状态
const offerSdp = ref('')
const answerSdp = ref('')
const qrCodeData = ref('')

// 调试相关状态
const showOfferDebug = ref(false)

// 触发文件选择
const triggerFileInput = () => {
  if (fileInputRef.value) {
    fileInputRef.value.click()
  }
}

// 处理拖拽上传
const handleDrop = (event) => {
  isDragging.value = false
  const files = event.dataTransfer.files
  if (files.length > 0) {
    selectedFile.value = files[0]
  }
}

// 初始化WebRTC连接
const initPeerConnection = () => {
  // 创建PeerConnection实例
  peerConnection.value = new RTCPeerConnection({
    iceServers: [
      { urls: 'stun:stun.l.google.com:19302' },
      { urls: 'stun:stun1.l.google.com:19302' }
    ]
  })

  // 创建DataChannel
  dataChannel.value = peerConnection.value.createDataChannel('fileTransfer', {
    ordered: true
  })

  // 监听ICE候选事件
  peerConnection.value.onicecandidate = (event) => {
    if (event.candidate) {
      // ICE候选收集完成后，可以更新Offer
      if (peerConnection.value && peerConnection.value.localDescription) {
        offerSdp.value = JSON.stringify(peerConnection.value.localDescription)
        generateQRCode(offerSdp.value)
      }
    }
  }

  // 监听连接状态变化
  peerConnection.value.onconnectionstatechange = () => {
    if (peerConnection.value.connectionState === 'connected') {
      connectionStatus.value = 'connected'
    }
  }

  // DataChannel事件监听
  dataChannel.value.onopen = () => {
    console.log('DataChannel已打开')
    connectionStatus.value = 'connected'
  }

  dataChannel.value.onclose = () => {
    console.log('DataChannel已关闭')
    connectionStatus.value = 'disconnected'
  }

  dataChannel.value.onerror = (error) => {
    console.error('DataChannel错误:', error)
    connectionStatus.value = 'disconnected'
  }
}

// 生成SDP Offer - 只生成文本内容不刷新二维码
const generateOffer = async () => {
  try {
    // 创建Offer
    const offer = await peerConnection.value.createOffer()
    // 设置本地描述
    await peerConnection.value.setLocalDescription(offer)
    // 保存Offer到文本变量
    offerSdp.value = JSON.stringify(offer)
    console.log('Offer已生成:', offerSdp.value)
    
    // 提示用户可通过显示调试信息查看Offer内容
    alert('Offer已生成，请点击"显示调试信息"查看完整内容')
  } catch (error) {
    console.error('生成Offer失败:', error)
  }
}

// 生成二维码
const generateQRCode = async (data) => {
  try {
    // 为了调试，打印出用于生成二维码的数据
    console.log('生成二维码的数据:', data)
    
    // 使用toDataURL方法生成二维码图片URL，增加尺寸和更高的纠错级别以提高识别率
    const qrCodeUrl = await QRCode.toDataURL(data, {
      width: 300,
      margin: 4,  // 增加边距使二维码更容易识别
      errorCorrectionLevel: 'H',  // 最高纠错级别
      color: {
        dark: '#000000',
        light: '#ffffff'
      }
    })
    
    // 创建img标签来显示二维码，增加尺寸并添加样式以提高清晰度
    qrCodeData.value = `<img src="${qrCodeUrl}" alt="连接二维码" style="width: 300px; height: 300px; display: block; margin: 0 auto; border: 5px solid white; box-shadow: 0 0 10px rgba(0,0,0,0.1);" />`
    console.log('二维码生成成功')
  } catch (error) {
    console.error('生成二维码失败:', error)
    // 在出错时显示错误信息
    qrCodeData.value = '<p class="text-danger">生成二维码失败，请手动复制Offer</p>'
  }
}

// 接受Answer
const acceptAnswer = async () => {
  try {
    if (!answerSdp.value) {
      console.error('Answer为空')
      return
    }

    // 解析Answer SDP
    const answer = JSON.parse(answerSdp.value)
    // 设置远程描述
    await peerConnection.value.setRemoteDescription(answer)
    console.log('Answer已接受:', answer)
    connectionStatus.value = 'connecting'
  } catch (error) {
    console.error('接受Answer失败:', error)
  }
}

// 选择文件事件
const onFileSelected = (file) => {
  selectedFile.value = file
}

// 发送文件
const sendFile = () => {
  if (!selectedFile.value || !dataChannel.value || dataChannel.value.readyState !== 'open') {
    console.error('无法发送文件: 文件未选择或连接未建立')
    return
  }

  isSending.value = true
  progress.value = 0

  const reader = new FileReader()
  
  reader.onload = (event) => {
    const fileData = event.target.result
    const fileInfo = {
      name: selectedFile.value.name,
      size: selectedFile.value.size,
      type: selectedFile.value.type
    }

    // 先发送文件信息
    const metadata = JSON.stringify(fileInfo)
    dataChannel.value.send(metadata)
    
    // 分片发送文件数据
    const chunkSize = 16384 // 16KB
    let offset = 0
    
    const sendNextChunk = () => {
      if (offset >= fileData.byteLength) {
        console.log('文件发送完成')
        isSending.value = false
        progress.value = 100
        return
      }

      const chunk = fileData.slice(offset, offset + chunkSize)
      dataChannel.value.send(chunk)
      
      offset += chunk.size
      progress.value = (offset / fileData.byteLength) * 100
      
      // 使用setTimeout避免阻塞
      setTimeout(sendNextChunk, 0)
    }
    
    // 开始发送文件块
    sendNextChunk()
  }
  
  reader.onerror = () => {
    console.error('文件读取失败')
    isSending.value = false
  }
  
  // 以ArrayBuffer形式读取文件
  reader.readAsArrayBuffer(selectedFile.value)
}

// 重置连接
const resetConnection = () => {
  if (peerConnection.value) {
    peerConnection.value.close()
  }
  
  // 重置所有状态
  initPeerConnection()
  generateOffer()
  connectionStatus.value = 'connecting'
  answerSdp.value = ''
  isSending.value = false
  progress.value = 0
}

// 组件挂载时初始化
onMounted(() => {
  initPeerConnection()
  generateOffer()
})

// 组件卸载时清理资源
onUnmounted(() => {
  if (peerConnection.value) {
    peerConnection.value.close()
  }
})
</script>

<style scoped>
.sender-container {
  max-width: 500px;
  margin: 0 auto;
}

/* 文件拖拽上传区域样式 */
.file-drop-area {
  border: 2px dashed #ddd;
  border-radius: 8px;
  padding: 30px 20px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
  background-color: #fafafa;
}

.file-drop-area:hover {
  border-color: #1976d2;
  background-color: #f5f5f5;
}

.file-drop-area[dragover] {
  border-color: #1976d2;
  background-color: #e3f2fd;
}

.file-drop-content {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.file-name {
  word-break: break-all;
  max-width: 100%;
  color: #1976d2;
}

/* 二维码区域样式 */
.qr-code-container {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.qr-code {
  padding: 20px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

/* 按钮组样式 */
.action-buttons {
  margin-top: 16px;
}

.action-buttons .v-btn {
  margin-bottom: 8px;
}
</style>