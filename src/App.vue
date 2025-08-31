<script setup>
import { ref } from 'vue'
import FileSender from './components/FileSender.vue'
import FileReceiver from './components/FileReceiver.vue'

// 选择模式：sender 或 receiver
const mode = ref('sender')
</script>

<template>
  <v-app class="app-container">
    <!-- 模式选择顶栏 -->
    <div class="mode-selector">
      <v-container fluid>
        <v-row justify="center">
          <v-btn-group>
            <v-btn
              :color="mode === 'sender' ? 'primary' : 'default'"
              @click="mode = 'sender'"
              outlined
              class="px-6"
            >
              发送文件
            </v-btn>
            <v-btn
              :color="mode === 'receiver' ? 'primary' : 'default'"
              @click="mode = 'receiver'"
              outlined
              class="px-6"
            >
              接收文件
            </v-btn>
          </v-btn-group>
        </v-row>
      </v-container>
    </div>

    <!-- 主内容区域 -->
    <main class="app-main">
      <v-container class="mt-8 mb-8">
        <v-row justify="center">
          <v-col cols="12" md="8">
            <!-- 根据选择的模式显示对应的组件 -->
            <component :is="mode === 'sender' ? FileSender : FileReceiver" />
          </v-col>
        </v-row>
      </v-container>
    </main>

    <!-- 底部信息 -->
    <footer class="app-footer">
      <v-footer dark>
        <v-row justify="center" align="center" no-gutters class="py-2">
          <v-col class="text-center">
            <p class="text-sm">Copyright © 2025 Yuyudifiesh</p>
          </v-col>
        </v-row>
      </v-footer>
    </footer>
  </v-app>
</template>

<style scoped>
.app-container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.app-header {
  position: sticky;
  top: 0;
  z-index: 100;
}

.app-main {
  flex: 1;
}

.mode-selector {
  background-color: #f5f5f5;
  padding: 12px 0;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
</style>
