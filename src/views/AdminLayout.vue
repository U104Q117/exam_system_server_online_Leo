<template>
  <div class="app-container">
    <!-- 左侧菜单 -->
    <div class="sidebar">
      <h3 class="sidebar-title">管理菜單</h3>
      <el-menu
        :default-active="activeMenu"
        class="sidebar-menu"
        @select="handleMenuSelect"
        router
      >
        <!-- 试题管理分组 -->
        <el-sub-menu index="question">
          <template #title>試題管理</template>
          <el-menu-item index="/admin/question-manage">
            <el-icon><Document /></el-icon>
            <span>題目管理</span>
          </el-menu-item>
          <el-menu-item index="/admin/category-manage">
            <el-icon><Folder /></el-icon>
            <span>類別管理</span>
          </el-menu-item>
        </el-sub-menu>
        <!-- 考试管理分组 -->
        <el-sub-menu index="exam">
          <template #title>考試管理</template>
          <el-menu-item index="/admin/paper-manage">
            <el-icon><Files /></el-icon>
            <span>試卷管理</span>
          </el-menu-item>
          <el-menu-item index="/admin/score-manage">
            <el-icon><DataAnalysis /></el-icon>
            <span>成績管理</span>
          </el-menu-item>
        </el-sub-menu>
        <!-- 系统管理分组 -->
        <el-sub-menu index="system">
          <template #title>系統管理</template>
          <el-menu-item index="/admin/banner-manage">
            <el-icon><Picture /></el-icon>
            <span>輪播圖管理</span>
          </el-menu-item>
          <el-menu-item index="/admin/notice-manage">
            <el-icon><Bell /></el-icon>
            <span>公告管理</span>
          </el-menu-item>
        </el-sub-menu>
        <!-- 视频管理分组 -->
        <el-sub-menu index="video">
          <template #title>視頻管理</template>
          <el-menu-item index="/admin/video-manage">
            <el-icon><VideoPlay /></el-icon>
            <span>視頻管理</span>
          </el-menu-item>
          <el-menu-item index="/admin/video-category-manage">
            <el-icon><Collection /></el-icon>
            <span>視頻分類</span>
          </el-menu-item>
        </el-sub-menu>
      </el-menu>
    </div>

    <!-- 右侧内容区 -->
    <div class="main-content">
      <router-view />
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { 
  Document, Folder, Files, DataAnalysis, 
  Picture, Bell, VideoPlay, Collection 
} from '@element-plus/icons-vue'

const route = useRoute()
const router = useRouter()

const activeMenu = computed(() => {
  return route.path
})

const handleMenuSelect = (index) => {
  router.push(index)
}
</script>

<style scoped>
.app-container {
  display: flex;
  height: 100vh;
  background-color: #f5f5f5;
}

.sidebar {
  width: 250px;
  background: white;
  box-shadow: 2px 0 8px rgba(0,0,0,0.1);
}

.sidebar-title {
  padding: 20px;
  margin: 0;
  border-bottom: 1px solid #eee;
  color: #333;
}

.sidebar-menu {
  border: none;
}

.main-content {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
}
</style>
