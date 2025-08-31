<template>
  <div class="receiver-container">
    <v-card elevation="4" class="p-6">
      <v-card-title class="text-h5 mb-4">文件接收端</v-card-title>

      <!-- 连接状态显示 -->
      <v-alert
        :type="connectionStatus === 'connected' ? 'success' : 'info'"
        dense
        class="mb-4"
      >
        {{ connectionStatus === 'connected' ? '已连接，等待接收文件' : '等待连接...' }}
      </v-alert>

      <!-- 扫描二维码区域 -->
      <div class="scan-container text-center mb-4">
        <v-btn @click="startScan" :disabled="isScanning" color="primary" class="mb-2">
          {{ isScanning ? '扫描中...' : '扫描二维码' }}
        </v-btn>
        <p class="text-sm text-muted">或手动输入Offer</p>
      </div>

      <!-- 手动输入offer区域 -->
      <v-textarea
        v-model="offerSdp"
        label="请输入发送端的Offer"
        outlined
        rows="4"
        class="mb-4"
      ></v-textarea>

      <!-- Answer显示区域 -->
      <div v-if="answerSdp" class="answer-container mb-4">
        <v-textarea
          v-model="answerSdp"
          label="Answer（请复制到发送端）"
          outlined
          rows="4"
          readonly
          class="mb-2"
        ></v-textarea>
      </div>

      <!-- 接收文件进度 -->
      <div v-if="isReceiving" class="mb-4">
        <p class="mb-2">{{ currentFileName }} ({{ formatFileSize(receivedBytes) }} / {{ formatFileSize(totalFileSize) }})</p>
        <v-progress-linear
          v-model="receiveProgress"
          color="primary"
          height="8"
          rounded
          class="mb-2"
        ></v-progress-linear>
        <p class="text-sm text-center">{{ Math.round(receiveProgress) }}% 已接收</p>
      </div>

      <!-- 已接收文件列表 -->
      <div v-if="receivedFiles.length > 0" class="received-files-container mb-4">
        <h4 class="text-subtitle-2 mb-2">已接收文件</h4>
        <v-list dense>
          <v-list-item v-for="(file, index) in receivedFiles" :key="index">
            <v-list-item-content>
              <v-list-item-title>{{ file.name }}</v-list-item-title>
              <v-list-item-subtitle>{{ formatFileSize(file.size) }} · {{ new Date(file.receivedAt).toLocaleString() }}</v-list-item-subtitle>
            </v-list-item-content>
            <v-list-item-actions>
              <v-btn @click="downloadFile(file)" color="primary" size="small" icon>
                <v-icon>mdi-download</v-icon>
              </v-btn>
            </v-list-item-actions>
          </v-list-item>
        </v-list>
      </div>

      <!-- 按钮组 - 居中显示 -->
      <div class="action-buttons text-center">
        <v-btn @click="startScan" :disabled="isScanning" color="primary" class="mb-2 mx-2">
          {{ isScanning ? '扫描中...' : '扫描二维码' }}
        </v-btn>
        
        <v-btn @click="processOffer" :disabled="!offerSdp || connectionStatus === 'connected'" color="secondary" class="mb-2 mx-2">
          处理Offer
        </v-btn>
        
        <v-btn v-if="answerSdp" @click="copyAnswer" color="info" class="mb-2 mx-2">
          复制Answer
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

// WebRTC相关状态
const peerConnection = ref(null)
const dataChannel = ref(null)
const connectionStatus = ref('disconnected') // disconnected, connecting, connected

// SDP相关状态
const offerSdp = ref('')
const answerSdp = ref('')

// 扫描相关状态
const isScanning = ref(false)
const scanner = ref(null)
const videoElement = ref(null)

// 文件接收相关状态
const isReceiving = ref(false)
const receiveProgress = ref(0)
const currentFileName = ref('')
const receivedBytes = ref(0)
const totalFileSize = ref(0)
const receivedFileData = ref([])
const receivedFiles = ref([])
const fileMetadata = ref(null)

// 初始化WebRTC连接
const initPeerConnection = () => {
  // 创建PeerConnection实例
  peerConnection.value = new RTCPeerConnection({
    iceServers: [
      { urls: 'stun:stun.l.google.com:19302' },
      { urls: 'stun:stun1.l.google.com:19302' }
    ]
  })

  // 监听ICE候选事件
  peerConnection.value.onicecandidate = (event) => {
    if (event.candidate) {
      // ICE候选收集完成后，可以更新Answer
      if (peerConnection.value && peerConnection.value.localDescription) {
        answerSdp.value = JSON.stringify(peerConnection.value.localDescription)
      }
    }
  }

  // 监听连接状态变化
  peerConnection.value.onconnectionstatechange = () => {
    if (peerConnection.value.connectionState === 'connected') {
      connectionStatus.value = 'connected'
    }
  }

  // 监听DataChannel创建事件
  peerConnection.value.ondatachannel = (event) => {
    dataChannel.value = event.channel
    setupDataChannelListeners()
  }
}

// 设置DataChannel监听器
const setupDataChannelListeners = () => {
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

  // 接收数据
  dataChannel.value.onmessage = (event) => {
    handleIncomingData(event.data)
  }
}

// 处理接收到的数据
const handleIncomingData = (data) => {
  // 如果是字符串，可能是文件元数据
  if (typeof data === 'string') {
    try {
      // 尝试解析为JSON
      const metadata = JSON.parse(data)
      // 如果包含文件信息字段，则认为是文件元数据
      if (metadata.name && metadata.size) {
        fileMetadata.value = metadata
        currentFileName.value = metadata.name
        totalFileSize.value = metadata.size
        receivedBytes.value = 0
        receivedFileData.value = []
        isReceiving.value = true
        receiveProgress.value = 0
        console.log('接收到文件元数据:', metadata)
        return
      }
    } catch (e) {
      console.log('接收到文本数据:', data)
    }
  } 
  // 如果是二进制数据，认为是文件内容
  else if (data instanceof ArrayBuffer) {
    if (fileMetadata.value) {
      // 存储接收到的文件块
      receivedFileData.value.push(new Uint8Array(data))
      receivedBytes.value += data.byteLength
      
      // 更新进度
      receiveProgress.value = (receivedBytes.value / totalFileSize.value) * 100
      
      // 检查是否接收完成
      if (receivedBytes.value >= totalFileSize.value) {
        finalizeFileReception()
      }
    }
  }
}

// 完成文件接收
const finalizeFileReception = () => {
  console.log('文件接收完成')
  
  // 合并所有接收到的文件块
  const totalLength = receivedFileData.value.reduce((acc, chunk) => acc + chunk.length, 0)
  const mergedArray = new Uint8Array(totalLength)
  
  let offset = 0
  for (const chunk of receivedFileData.value) {
    mergedArray.set(chunk, offset)
    offset += chunk.length
  }
  
  // 创建Blob对象
  const blob = new Blob([mergedArray], { type: fileMetadata.value.type })
  
  // 添加到已接收文件列表
  receivedFiles.value.push({
    name: fileMetadata.value.name,
    size: fileMetadata.value.size,
    type: fileMetadata.value.type,
    blob: blob,
    receivedAt: new Date()
  })
  
  // 重置接收状态
  isReceiving.value = false
  receiveProgress.value = 100
  fileMetadata.value = null
  receivedFileData.value = []
}

// 处理Offer
const processOffer = async () => {
  try {
    if (!offerSdp.value) {
      console.error('Offer为空')
      return
    }

    // 解析Offer SDP
    const offer = JSON.parse(offerSdp.value)
    // 设置远程描述
    await peerConnection.value.setRemoteDescription(offer)
    // 创建Answer
    const answer = await peerConnection.value.createAnswer()
    // 设置本地描述
    await peerConnection.value.setLocalDescription(answer)
    // 保存Answer
    answerSdp.value = JSON.stringify(answer)
    
    console.log('Answer已生成:', answerSdp.value)
    connectionStatus.value = 'connecting'
  } catch (error) {
    console.error('处理Offer失败:', error)
  }
}

// 开始扫描二维码
const startScan = async () => {
  try {
    isScanning.value = true
    
    // 请求摄像头权限
    const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } })
    
    // 创建视频元素用于扫描
    const video = document.createElement('video')
    video.srcObject = stream
    video.autoplay = true
    video.style.position = 'fixed'
    video.style.top = '50%'
    video.style.left = '50%'
    video.style.transform = 'translate(-50%, -50%)'
    video.style.zIndex = '1000'
    video.style.width = '90%'
    video.style.maxWidth = '400px'
    
    // 创建扫描界面
    const scanContainer = document.createElement('div')
    scanContainer.style.position = 'fixed'
    scanContainer.style.top = '0'
    scanContainer.style.left = '0'
    scanContainer.style.width = '100%'
    scanContainer.style.height = '100%'
    scanContainer.style.backgroundColor = 'rgba(0, 0, 0, 0.8)'
    scanContainer.style.zIndex = '999'
    scanContainer.style.display = 'flex'
    scanContainer.style.flexDirection = 'column'
    scanContainer.style.alignItems = 'center'
    scanContainer.style.justifyContent = 'center'
    
    // 创建关闭按钮
    const closeButton = document.createElement('button')
    closeButton.innerText = '关闭'
    closeButton.style.position = 'absolute'
    closeButton.style.top = '20px'
    closeButton.style.right = '20px'
    closeButton.style.padding = '8px 16px'
    closeButton.style.backgroundColor = 'red'
    closeButton.style.color = 'white'
    closeButton.style.border = 'none'
    closeButton.style.borderRadius = '4px'
    closeButton.style.cursor = 'pointer'
    closeButton.onclick = () => {
      stopScan(stream, video, scanContainer)
    }
    
    // 创建扫描提示
    const scanHint = document.createElement('p')
    scanHint.innerText = '请将二维码对准扫描框'
    scanHint.style.color = 'white'
    scanHint.style.marginTop = '20px'
    
    // 添加元素到页面
    scanContainer.appendChild(closeButton)
    scanContainer.appendChild(video)
    scanContainer.appendChild(scanHint)
    document.body.appendChild(scanContainer)
    
    // 等待视频加载完成
    video.onloadedmetadata = () => {
      // 创建canvas用于处理视频帧
      const canvas = document.createElement('canvas')
      const ctx = canvas.getContext('2d')
      
      // 设置canvas尺寸与视频相同
      canvas.width = video.videoWidth
      canvas.height = video.videoHeight
      
      // 扫描函数
      const scan = () => {
        if (!isScanning.value) return
        
        try {
          // 绘制当前视频帧到canvas
          ctx.drawImage(video, 0, 0, canvas.width, canvas.height)
          
          // 获取图像数据
          const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
          
          // 这里应该使用QR码扫描库来解析图像，但为了简化，我们假设已经扫描到了数据
          // 实际项目中应该使用jsQR或其他QR码扫描库
          // 这里我们模拟扫描完成
          // stopScan(stream, video, scanContainer)
        } catch (error) {
          console.error('扫描出错:', error)
        }
        
        // 继续扫描下一帧
        if (isScanning.value) {
          requestAnimationFrame(scan)
        }
      }
      
      // 开始扫描
      scan()
    }
    
  } catch (error) {
    console.error('扫描二维码失败:', error)
    isScanning.value = false
    alert('无法访问摄像头，请确保已授予权限')
  }
}

// 停止扫描
const stopScan = (stream, video, scanContainer) => {
  // 停止视频流
  if (stream) {
    stream.getTracks().forEach(track => track.stop())
  }
  
  // 移除DOM元素
  if (video && video.parentNode) {
    video.parentNode.removeChild(video)
  }
  if (scanContainer && scanContainer.parentNode) {
    scanContainer.parentNode.removeChild(scanContainer)
  }
  
  // 重置状态
  isScanning.value = false
}

// 复制Answer到剪贴板
const copyAnswer = () => {
  if (answerSdp.value) {
    navigator.clipboard.writeText(answerSdp.value)
      .then(() => {
        alert('Answer已复制到剪贴板')
      })
      .catch(err => {
        console.error('复制失败:', err)
      })
  }
}

// 下载文件
const downloadFile = (file) => {
  const url = URL.createObjectURL(file.blob)
  const a = document.createElement('a')
  a.href = url
  a.download = file.name
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

// 格式化文件大小
const formatFileSize = (bytes) => {
  if (bytes < 1024) return bytes + ' B'
  else if (bytes < 1048576) return (bytes / 1024).toFixed(2) + ' KB'
  else return (bytes / 1048576).toFixed(2) + ' MB'
}

// 重置连接
const resetConnection = () => {
  if (peerConnection.value) {
    peerConnection.value.close()
  }
  
  // 重置所有状态
  initPeerConnection()
  connectionStatus.value = 'disconnected'
  offerSdp.value = ''
  answerSdp.value = ''
  isReceiving.value = false
  receiveProgress.value = 0
  currentFileName.value = ''
  receivedBytes.value = 0
  totalFileSize.value = 0
  fileMetadata.value = null
  receivedFileData.value = []
}

// 组件挂载时初始化
onMounted(() => {
  initPeerConnection()
})

// 组件卸载时清理资源
onUnmounted(() => {
  if (peerConnection.value) {
    peerConnection.value.close()
  }
  if (isScanning.value) {
    // 如果正在扫描，停止扫描
    isScanning.value = false
  }
})
</script>

<style scoped>
.receiver-container {
  max-width: 500px;
  margin: 0 auto;
}

.scan-container {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.answer-container {
  position: relative;
}

.received-files-container {
  max-height: 300px;
  overflow-y: auto;
}

/* 按钮组样式 */
.action-buttons {
  margin-top: 16px;
}

.action-buttons .v-btn {
  margin-bottom: 8px;
}
</style>