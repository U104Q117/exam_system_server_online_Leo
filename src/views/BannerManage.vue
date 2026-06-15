<template>
  <div class="banner-manage-container">
    <div class="action-bar">
      <el-button type="primary" @click="showAddDialog" icon="Plus">添加輪播圖</el-button>
      <el-button @click="getBannerList" icon="Refresh">刷新列表</el-button>
    </div>

    <div class="banner-list">
      <el-table :data="bannerList" v-loading="loading" stripe>
        <el-table-column prop="id" label="ID" width="80" />
        <el-table-column label="預覽圖" width="120">
          <template #default="scope">
            <el-image :src="scope.row.imageUrl" :preview-src-list="[scope.row.imageUrl]" fit="cover" style="width: 80px; height: 45px; border-radius: 4px;" :preview-teleported="true" />
          </template>
        </el-table-column>
        <el-table-column prop="title" label="標題" min-width="150" />
        <el-table-column prop="description" label="描述" min-width="200" show-overflow-tooltip />
        <el-table-column prop="linkUrl" label="跳轉鏈接" min-width="150" show-overflow-tooltip />
        <el-table-column prop="sortOrder" label="排序" width="80" />
        <el-table-column label="狀態" width="100">
          <template #default="scope">
            <el-tag :type="scope.row.isActive ? 'success' : 'danger'">{{ scope.row.isActive ? '启用' : '禁用' }}</el-tag>
          </template>
        </el-table-column>
        <el-table-column prop="createTime" label="創建時間" width="180" />
        <el-table-column label="操作" width="200" fixed="right">
          <template #default="scope">
            <el-button size="small" @click="editBanner(scope.row)" icon="Edit">編輯</el-button>
            <el-button size="small" :type="scope.row.isActive ? 'warning' : 'success'" @click="toggleBannerStatus(scope.row)" icon="Switch">{{ scope.row.isActive ? '禁用' : '启用' }}</el-button>
            <el-button size="small" type="danger" @click="deleteBanner(scope.row)" icon="Delete">刪除</el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>

    <el-dialog v-model="dialogVisible" :title="dialogTitle" width="600px" :close-on-click-modal="false">
      <el-form :model="bannerForm" :rules="bannerRules" ref="bannerFormRef" label-width="100px">
        <el-form-item label="標題" prop="title">
          <el-input v-model="bannerForm.title" placeholder="請輸入輪播圖標題" />
        </el-form-item>
        <el-form-item label="描述" prop="description">
          <el-input v-model="bannerForm.description" type="textarea" :rows="3" placeholder="請輸入輪播圖描述" />
        </el-form-item>
        <el-form-item label="圖片" prop="imageUrl">
          <el-radio-group v-model="imageInputType" @change="handleImageInputTypeChange" style="margin-bottom: 10px;">
            <el-radio label="upload">上傳圖片到伺服器</el-radio>
            <el-radio label="url">輸入外部圖片URL</el-radio>
          </el-radio-group>
          <div v-if="imageInputType === 'upload'">
            <el-upload ref="uploadRef" :action="uploadAction" :headers="uploadHeaders" :show-file-list="false" :on-success="handleUploadSuccess" :on-error="handleUploadError" :before-upload="beforeUpload" accept="image/*" :loading="uploadLoading">
              <el-button type="primary" :loading="uploadLoading"><el-icon><Upload /></el-icon>{{ uploadLoading ? '上传中...' : '选择图片文件' }}</el-button>
            </el-upload>
            <div class="upload-tips"><p>支援 JPG PNG GIF 格式，檔案大小不超過 5MB</p></div>
          </div>
          <div v-if="imageInputType === 'url'">
            <el-input v-model="bannerForm.imageUrl" placeholder="请输入图片URL地址（支持外部网站图片）" style="margin-bottom: 10px;" />
            <div class="url-tips"><p>可以輸入其他網站的圖片位址，例如：https://example.com/image.jpg</p></div>
          </div>
          <div v-if="bannerForm.imageUrl" class="preview-image">
            <el-image :src="bannerForm.imageUrl" fit="cover" style="width: 200px; height: 112px; border-radius: 4px; margin-top: 10px;" :preview-src-list="[bannerForm.imageUrl]" :preview-teleported="true" />
            <div style="margin-top: 5px; font-size: 12px; color: #999;">點擊圖片可預覽</div>
          </div>
        </el-form-item>
        <el-form-item label="跳轉鏈接">
          <el-input v-model="bannerForm.linkUrl" placeholder="請輸入跳轉鏈接（可選）" />
          <div class="link-tips">
            <p><strong>支援的鏈接格式：</strong></p>
            <p>• 外部網站：https://www.baidu.com</p>
            <p>• 内部頁面：/practice 或 /exam/list</p>
            <p>• 留空則點擊無效跳轉效果</p>
          </div>
        </el-form-item>
        <el-form-item label="排序">
          <el-input-number v-model="bannerForm.sortOrder" :min="0" :max="999" placeholder="數字越小排序越靠前" />
        </el-form-item>
        <el-form-item label="狀態">
          <el-switch v-model="bannerForm.isActive" active-text="啟用" inactive-text="禁用" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" @click="submitBanner" :loading="submitLoading">確認</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Upload, Plus, Refresh, Edit, Switch, Delete } from '@element-plus/icons-vue'
import request from '../utils/request'

const loading = ref(false)
const bannerList = ref([])
const dialogVisible = ref(false)
const dialogTitle = ref('')
const submitLoading = ref(false)
const bannerFormRef = ref()
const imageInputType = ref('upload')
const uploadRef = ref()
const uploadLoading = ref(false)

const uploadAction = ref('http://localhost:8080/api/banners/upload-image')
const uploadHeaders = ref({})

const bannerForm = reactive({ id: null, title: '', description: '', imageUrl: '', linkUrl: '', sortOrder: 0, isActive: true })

const bannerRules = {
  title: [{ required: true, message: '請輸入輪播圖標題', trigger: 'blur' }],
  imageUrl: [{ required: true, message: '請選擇圖片文件或輸入圖片URL', trigger: 'blur' }]
}

const getBannerList = async () => {
  loading.value = true
  try {
    const res = await request.get('/api/banners/list')
    bannerList.value = res.data || []
  } catch (error) {
    ElMessage.error('獲取輪播圖列表失敗')
  } finally {
    loading.value = false
  }
}

const showAddDialog = () => { resetForm(); dialogTitle.value = '添加輪播圖'; dialogVisible.value = true; }
const editBanner = (banner) => { Object.assign(bannerForm, banner); dialogTitle.value = '編輯輪播圖'; dialogVisible.value = true; }

const resetForm = () => {
  Object.assign(bannerForm, { id: null, title: '', description: '', imageUrl: '', linkUrl: '', sortOrder: 0, isActive: true })
  imageInputType.value = 'upload'
  if (bannerFormRef.value) bannerFormRef.value.resetFields()
}

const submitBanner = async () => {
  if (!bannerFormRef.value) return
  try {
    await bannerFormRef.value.validate()
    submitLoading.value = true
    const isEdit = bannerForm.id !== null
    const url = isEdit ? '/api/banners/update' : '/api/banners/add'
    const method = isEdit ? 'put' : 'post'
    await request[method](url, bannerForm)
    ElMessage.success(isEdit ? '輪播圖更新成功' : '輪播圖添加成功')
    dialogVisible.value = false
    await getBannerList()
  } catch (error) {
    ElMessage.error('操作失敗，請重試')
  } finally {
    submitLoading.value = false
  }
}

const toggleBannerStatus = async (banner) => {
  try {
    const newStatus = !banner.isActive
    await request.put(`/api/banners/toggle/${banner.id}?isActive=${newStatus}`)
    banner.isActive = newStatus
    ElMessage.success(`輪播圖已${newStatus ? '啟用' : '禁用'}`)
  } catch (error) {
    ElMessage.error('操作失敗')
  }
}

const deleteBanner = async (banner) => {
  try {
    await ElMessageBox.confirm(`確認要刪除輪播圖"${banner.title}"嗎？`, '確認刪除', { confirmButtonText: '確認', cancelButtonText: '取消', type: 'warning' })
    await request.delete(`/api/banners/delete/${banner.id}`)
    ElMessage.success('刪除成功')
    await getBannerList()
  } catch (error) {
    if (error !== 'cancel') ElMessage.error('删除失败')
  }
}

const handleImageInputTypeChange = () => { bannerForm.imageUrl = '' }

const beforeUpload = (file) => {
  const isImage = file.type.startsWith('image/')
  if (!isImage) {
    ElMessage.error('只能上传图片文件！')
    return false
  }
  const isLt5M = file.size / 1024 / 1024 < 5
  if (!isLt5M) {
    ElMessage.error('圖片大小不能超過5MB！')
    return false
  }
  uploadLoading.value = true
  return true
}

const handleUploadSuccess = (response, file) => {
  uploadLoading.value = false
  if (response.code === 200) {
    bannerForm.imageUrl = response.data
    ElMessage.success('图片上传成功！')
  } else {
    ElMessage.error(response.message || '圖片上傳失敗！')
  }
}

const handleUploadError = (error, file) => {
  uploadLoading.value = false
  ElMessage.error('圖片上傳失敗，請重試！')
}

onMounted(() => getBannerList())

</script>

<style scoped>
.banner-manage-container { padding: 20px; }
.action-bar { margin-bottom: 20px; }
.banner-list { background: white; border-radius: 8px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
.preview-image { margin-top: 10px; text-align: center; padding: 10px; background: #fafafa; border-radius: 8px; border: 1px dashed #d9d9d9; }
.upload-tips, .url-tips { margin-top: 8px; font-size: 12px; color: #999; line-height: 1.4; }
.link-tips { margin-top: 8px; font-size: 12px; color: #666; line-height: 1.5; background: #f0f9ff; border: 1px solid #bfdbfe; border-radius: 6px; padding: 12px; }
.link-tips p { margin: 0 0 4px 0; }
.link-tips p:last-child { margin-bottom: 0; }
.link-tips strong { color: #1d4ed8; }
</style>