<template>
  <div class="company-detail">
    <!-- 面包屑导航 -->
    <div class="breadcrumb-nav">
      <el-breadcrumb separator="/">
        <el-breadcrumb-item @click="$router.push('/interview-questions')">
          <span class="breadcrumb-link">企業題庫</span>
        </el-breadcrumb-item>
        <el-breadcrumb-item>{{ company.name }}</el-breadcrumb-item>
      </el-breadcrumb>
    </div>

    <!-- 公司信息头部 -->
    <div class="company-header">
      <div class="company-logo">
        <img :src="company.logo || '/default-company-logo.png'" :alt="company.name">
      </div>
      <div class="company-info">
        <h1 class="company-title">
          {{ company.name }} 面試真題
          <span class="plus-badge" v-if="company.isPremium">Plus</span>
        </h1>
        <p class="company-desc">{{ company.description }}</p>
        <div class="company-stats">
          <div class="stat-item">
            <span class="stat-number">{{ company.totalQuestions }}</span>
            <span class="stat-label">道題目</span>
          </div>
          <div class="stat-item">
            <span class="stat-number">{{ company.questionSets.length }}</span>
            <span class="stat-label">套題集</span>
          </div>
          <div class="stat-item">
            <span class="stat-number">{{ getAvgDifficulty() }}</span>
            <span class="stat-label">平均難度</span>
          </div>
        </div>
      </div>
      <div class="company-actions">
        <el-button type="primary" size="large" @click="startCompanyMockInterview">
          開始模拟面試
        </el-button>
        <el-button size="large" @click="showBatchUploadDialog">
          添加題集
        </el-button>
      </div>
    </div>

    <!-- 筛选区域 -->
    <div class="filter-section">
      <div class="filter-left">
        <el-select v-model="filters.direction" placeholder="技術方向" clearable style="width: 150px" @change="handleFilterChange">
          <el-option label="全部" value=""></el-option>
          <el-option label="Java" value="java"></el-option>
          <el-option label="前端" value="frontend"></el-option>
          <el-option label="大數據" value="bigdata"></el-option>
          <el-option label="算法" value="algorithm"></el-option>
          <el-option label="運维" value="devops"></el-option>
          <el-option label="測試" value="testing"></el-option>
        </el-select>
        
        <el-select v-model="filters.difficulty" placeholder="難度" clearable style="width: 120px" @change="handleFilterChange">
          <el-option label="全部" value=""></el-option>
          <el-option label="簡單" value="easy"></el-option>
          <el-option label="中等" value="medium"></el-option>
          <el-option label="困難" value="hard"></el-option>
        </el-select>
        
        <el-select v-model="filters.year" placeholder="年份" clearable style="width: 120px" @change="handleFilterChange">
          <el-option label="全部" value=""></el-option>
          <el-option 
            v-for="year in yearOptions" 
            :key="year" 
            :label="year" 
            :value="year">
          </el-option>
        </el-select>
      </div>
      
      <div class="filter-right">
        <el-input v-model="filters.keyword" placeholder="搜索題集或題目" clearable style="width: 250px" @input="handleFilterChange">
          <template #prefix>
            <el-icon><Search /></el-icon>
          </template>
        </el-input>
      </div>
    </div>

    <!-- 题集列表 -->
    <div class="question-sets" v-loading="loading">
      <div 
        v-for="questionSet in filteredQuestionSets" 
        :key="questionSet.id"
        class="question-set-card"
      >
        <!-- 题集头部 -->
        <div class="set-header">
          <div class="set-title-section">
            <h3 class="set-title">{{ questionSet.name }}</h3>
            <div class="set-meta">
              <el-tag :type="getDifficultyType(questionSet.difficulty)" size="small">
                {{ getDifficultyLabel(questionSet.difficulty) }}
              </el-tag>
              <el-tag type="info" size="small">{{ getTechLabel(questionSet.direction) }}</el-tag>
              <span class="set-year">{{ questionSet.year }}年</span>
              <span class="question-count">{{ questionSet.questionCount }}道題</span>
              <span class="view-count">{{ questionSet.viewCount || 0 }}瀏覽</span>
            </div>
          </div>
          <div class="set-actions">
            <el-button 
              size="small" 
              @click="toggleQuestions(questionSet)"
              :icon="questionSet.expanded ? 'ArrowUp' : 'ArrowDown'"
            >
              {{ questionSet.expanded ? '收起' : '展开' }}
            </el-button>
            <el-button size="small" @click="favoriteSet(questionSet)" :type="questionSet.isFavorite ? 'primary' : 'default'">
              <el-icon><Star /></el-icon>
              {{ questionSet.favoriteCount || 0 }}
            </el-button>
            <el-button type="primary" size="small" @click="startSetPractice(questionSet)">
              开始练习
            </el-button>
          </div>
        </div>

        <!-- 题目列表（可折叠） -->
        <div v-if="questionSet.expanded" class="questions-list">
          <div v-if="questionSet.questions.length === 0" class="empty-questions">
            <el-empty description="暂无題目數據" :image-size="80" />
          </div>
          <div v-else>
            <div 
              v-for="(question, index) in questionSet.questions" 
              :key="question.id"
              class="question-item"
            >
              <div class="question-number">{{ index + 1 }}.</div>
              <div class="question-content">
                <p class="question-text">{{ question.content }}</p>
                <div class="question-tags">
                  <el-tag size="small" type="success">{{ question.type }}</el-tag>
                  <el-tag size="small" type="warning" v-if="question.isHot">🔥 熱門</el-tag>
                  <el-tag size="small" v-if="question.difficulty">{{ getDifficultyLabel(question.difficulty) }}</el-tag>
                </div>
              </div>
              <div class="question-actions">
                <el-button size="small" @click="viewQuestionDetail(question)">
                  查看詳情
                </el-button>
                <el-button size="small" type="primary" @click="practiceQuestion(question)">
                  開始練習
                </el-button>
              </div>
            </div>
          </div>
        </div>
      </div>
      
      <!-- 空状态 -->
      <div v-if="filteredQuestionSets.length === 0" class="empty-state">
        <el-empty description="暂无符合条件的題集" :image-size="120">
          <el-button type="primary" @click="showBatchUploadDialog">添加題集</el-button>
        </el-empty>
      </div>
    </div>

    <!-- 批量录入题集对话框 -->
    <el-dialog v-model="batchUploadDialogVisible" title="添加題集" width="800px" :close-on-click-modal="false">
      <el-form :model="batchForm" :rules="batchRules" ref="batchFormRef" label-width="120px">
        <!-- 基本信息 -->
        <div class="form-section">
          <h4>基本信息</h4>
          <el-row :gutter="20">
            <el-col :span="12">
              <el-form-item label="题集名称" prop="setName">
                <el-input v-model="batchForm.setName" placeholder="如：Java高级工程师面试题（2024春招）"></el-input>
              </el-form-item>
            </el-col>
            <el-col :span="12">
              <el-form-item label="技术方向" prop="direction">
                <el-select v-model="batchForm.direction" placeholder="選擇技術方向" style="width: 100%">
                  <el-option label="Java" value="java"></el-option>
                  <el-option label="前端" value="frontend"></el-option>
                  <el-option label="大數據" value="bigdata"></el-option>
                  <el-option label="算法" value="algorithm"></el-option>
                  <el-option label="運维" value="devops"></el-option>
                  <el-option label="測試" value="testing"></el-option>
                </el-select>
              </el-form-item>
            </el-col>
          </el-row>
          
          <el-row :gutter="20">
            <el-col :span="12">
              <el-form-item label="難度級別" prop="difficulty">
                <el-select v-model="batchForm.difficulty" placeholder="選擇難度" style="width: 100%">
                  <el-option label="簡單" value="easy"></el-option>
                  <el-option label="中等" value="medium"></el-option>
                  <el-option label="困難" value="hard"></el-option>
                </el-select>
              </el-form-item>
            </el-col>
            <el-col :span="12">
              <el-form-item label="面試年份" prop="year">
                <el-select v-model="batchForm.year" placeholder="選擇年份" style="width: 100%">
                  <el-option 
                    v-for="year in yearOptions" 
                    :key="year" 
                    :label="year" 
                    :value="year">
                  </el-option>
                </el-select>
              </el-form-item>
            </el-col>
          </el-row>
        </div>

        <!-- 题目列表 -->
        <div class="form-section">
          <div class="section-header">
            <h4>題目列表</h4>
            <div class="section-actions">
              <el-button size="small" @click="addQuestion">+ 添加題目</el-button>
              <el-button size="small" @click="showBatchInputDialog">批量輸入</el-button>
            </div>
          </div>
          
          <div class="questions-list-form">
            <div 
              v-for="(question, index) in batchForm.questions" 
              :key="index"
              class="question-item-form"
            >
              <div class="question-number">{{ index + 1 }}.</div>
              <div class="question-input">
                <el-input
                  v-model="question.content"
                  type="textarea"
                  :rows="2"
                  :placeholder="`请输入第${index + 1}题内容`"
                  maxlength="500"
                  show-word-limit
                ></el-input>
              </div>
              <div class="question-actions">
                <el-button size="small" type="danger" @click="removeQuestion(index)" :disabled="batchForm.questions.length <= 1">
                  删除
                </el-button>
              </div>
            </div>
          </div>
        </div>
      </el-form>
      
      <template #footer>
        <el-button @click="batchUploadDialogVisible = false">取消</el-button>
        <el-button @click="previewQuestionSet">预览效果</el-button>
        <el-button type="primary" @click="handleBatchSubmit" :loading="batchUploading">保存題集</el-button>
      </template>
    </el-dialog>

    <!-- 批量输入对话框 -->
    <el-dialog v-model="batchInputDialogVisible" title="批量输入题目" width="600px">
      <div class="batch-input-tip">
        <p>請每行輸入一道題目，系統會自動自動分割：</p>
      </div>
      <el-input
        v-model="batchInputText"
        type="textarea"
        :rows="10"
        placeholder="例如：
1. 解释Spring IOC容器的工作原理？
2. 如何實現分布式鎖？
3. MySQL索引優化有哪些策略？"
      ></el-input>
      <template #footer>
        <el-button @click="batchInputDialogVisible = false">取消</el-button>
        <el-button type="primary" @click="handleBatchInput">確定導入</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script>
import { ref, reactive, onMounted, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Search, Star, ArrowUp, ArrowDown } from '@element-plus/icons-vue'

export default {
  name: 'CompanyDetail',
  components: {
    Search,
    Star,
    ArrowUp,
    ArrowDown
  },
  setup() {
    const route = useRoute()
    const router = useRouter()
    
    // 响应式数据
    const loading = ref(false)
    const company = ref({})
    
    // 动态生成年份选项
    const currentYear = new Date().getFullYear()
    const yearOptions = Array.from({ length: 6 }, (_, i) => String(currentYear - i))
    
    // 筛选条件
    const filters = reactive({
      direction: '',
      difficulty: '',
      year: '',
      keyword: ''
    })
    
    // 批量录入相关
    const batchUploadDialogVisible = ref(false)
    const batchUploading = ref(false)
    const batchFormRef = ref()
    const batchForm = reactive({
      setName: '',
      direction: '',
      difficulty: '',
      year: '',
      questions: [
        { content: '' }
      ]
    })
    
    const batchRules = {
      setName: [{ required: true, message: '请输入题集名称', trigger: 'blur' }],
      direction: [{ required: true, message: '请选择技术方向', trigger: 'change' }],
      difficulty: [{ required: true, message: '请选择难度级别', trigger: 'change' }],
      year: [{ required: true, message: '请选择面试年份', trigger: 'change' }]
    }
    
    // 批量输入相关
    const batchInputDialogVisible = ref(false)
    const batchInputText = ref('')
    
    // 计算过滤后的题集列表
    const filteredQuestionSets = computed(() => {
      if (!company.value.questionSets) return []
      
      return company.value.questionSets.filter(set => {
        // 技术方向筛选
        if (filters.direction && set.direction !== filters.direction) {
          return false
        }
        
        // 难度筛选
        if (filters.difficulty && set.difficulty !== filters.difficulty) {
          return false
        }
        
        // 年份筛选
        if (filters.year && set.year !== parseInt(filters.year)) {
          return false
        }
        
        // 关键词筛选
        if (filters.keyword) {
          const keyword = filters.keyword.toLowerCase()
          const matchTitle = set.name.toLowerCase().includes(keyword)
          const matchQuestions = set.questions.some(q => 
            q.content.toLowerCase().includes(keyword)
          )
          if (!matchTitle && !matchQuestions) {
            return false
          }
        }
        
        return true
      })
    })
    
    // 工具函数
    const getTechLabel = (tech) => {
      const labels = {
        java: 'Java',
        frontend: '前端',
        bigdata: '大数据',
        algorithm: '算法',
        devops: '运维',
        testing: '测试'
      }
      return labels[tech] || tech
    }
    
    const getDifficultyLabel = (difficulty) => {
      const labels = {
        easy: '简单',
        medium: '中等',
        hard: '困难'
      }
      return labels[difficulty] || difficulty
    }
    
    const getDifficultyType = (difficulty) => {
      const types = {
        easy: 'success',
        medium: 'warning',
        hard: 'danger'
      }
      return types[difficulty] || ''
    }
    
    const getAvgDifficulty = () => {
      if (!company.value.questionSets || company.value.questionSets.length === 0) {
        return '暂无'
      }
      
      const difficulties = company.value.questionSets.map(set => set.difficulty)
      const counts = { easy: 0, medium: 0, hard: 0 }
      difficulties.forEach(d => counts[d]++)
      
      if (counts.hard > counts.medium && counts.hard > counts.easy) return '困难'
      if (counts.medium > counts.easy) return '中等'
      return '简单'
    }
    
    // 初始化企业数据
    const initCompanyData = () => {
      // 模拟根据路由参数获取企业数据
      const companyId = parseInt(route.params.id)
      
      // 模拟企业数据
      const mockCompanies = {
        1: {
          id: 1,
          name: '阿里巴巴',
          logo: '/logos/alibaba.png',
          description: '中国领先的电商和云计算公司，致力于让天下没有难做的生意',
          isPremium: true,
          totalQuestions: 156,
          questionSets: [
            {
              id: 1,
              name: 'Java高级开发工程师面试题（2024春招）',
              direction: 'java',
              difficulty: 'hard',
              year: 2024,
              questionCount: 15,
              viewCount: 1250,
              favoriteCount: 89,
              expanded: false,
              questions: [
                { 
                  id: 1, 
                  content: '请解释Spring IOC容器的工作原理，以及如何实现依赖注入？', 
                  type: '简答题',
                  isHot: true,
                  difficulty: 'medium'
                },
                { 
                  id: 2, 
                  content: '如何实现分布式锁？请比较Redis和Zookeeper两种方案的优缺点。', 
                  type: '简答题',
                  difficulty: 'hard'
                },
                { 
                  id: 3, 
                  content: 'MySQL索引优化有哪些策略？请结合实际案例说明。', 
                  type: '简答题',
                  difficulty: 'medium'
                }
              ]
            },
            {
              id: 2,
              name: '前端开发面试题（2024校招）',
              direction: 'frontend',
              difficulty: 'medium',
              year: 2024,
              questionCount: 12,
              viewCount: 890,
              favoriteCount: 45,
              expanded: false,
              questions: [
                { 
                  id: 4, 
                  content: 'Vue3 Composition API相比Options API有哪些优势？', 
                  type: '简答题',
                  difficulty: 'medium'
                },
                { 
                  id: 5, 
                  content: '如何实现前端性能优化？请从多个维度分析。', 
                  type: '简答题',
                  isHot: true,
                  difficulty: 'medium'
                }
              ]
            }
          ]
        }
      }
      
      company.value = mockCompanies[companyId] || {}
    }
    
    // 事件处理函数
    const handleFilterChange = () => {
      // 筛选逻辑已通过计算属性实现
    }
    
    const toggleQuestions = (questionSet) => {
      questionSet.expanded = !questionSet.expanded
    }
    
    const favoriteSet = (questionSet) => {
      questionSet.isFavorite = !questionSet.isFavorite
      if (questionSet.isFavorite) {
        questionSet.favoriteCount = (questionSet.favoriteCount || 0) + 1
        ElMessage.success('收藏成功')
      } else {
        questionSet.favoriteCount = Math.max(0, (questionSet.favoriteCount || 0) - 1)
        ElMessage.success('取消收藏')
      }
    }
    
    const startSetPractice = (questionSet) => {
      router.push(`/practice/set/${questionSet.id}`)
    }
    
    const startCompanyMockInterview = () => {
      router.push(`/mock-interview?company=${company.value.id}`)
    }
    
    const viewQuestionDetail = (question) => {
      router.push(`/question/${question.id}`)
    }
    
    const practiceQuestion = (question) => {
      router.push(`/practice/question/${question.id}`)
    }
    
    // 批量录入相关方法
    const showBatchUploadDialog = () => {
      batchUploadDialogVisible.value = true
    }
    
    const addQuestion = () => {
      batchForm.questions.push({ content: '' })
    }
    
    const removeQuestion = (index) => {
      if (batchForm.questions.length > 1) {
        batchForm.questions.splice(index, 1)
      }
    }
    
    const showBatchInputDialog = () => {
      batchInputDialogVisible.value = true
    }
    
    const handleBatchInput = () => {
      if (!batchInputText.value.trim()) {
        ElMessage.warning('请输入题目内容')
        return
      }
      
      // 解析批量输入的文本
      const lines = batchInputText.value.split('\n').filter(line => line.trim())
      const questions = lines.map(line => {
        // 移除序号（如 "1. "）
        const content = line.replace(/^\d+\.\s*/, '').trim()
        return { content }
      }).filter(q => q.content)
      
      if (questions.length === 0) {
        ElMessage.warning('未解析到有效题目')
        return
      }
      
      batchForm.questions = questions
      batchInputDialogVisible.value = false
      batchInputText.value = ''
      
      ElMessage.success(`成功导入 ${questions.length} 道题目`)
    }
    
    const previewQuestionSet = () => {
      ElMessage.info('预览功能开发中...')
    }
    
    const handleBatchSubmit = async () => {
      try {
        await batchFormRef.value.validate()
        
        // 验证题目内容
        const validQuestions = batchForm.questions.filter(q => q.content.trim())
        if (validQuestions.length === 0) {
          ElMessage.warning('请至少添加一道题目')
          return
        }
        
        batchUploading.value = true
        
        // 模拟API调用
        const questionSet = {
          id: Date.now(),
          name: batchForm.setName,
          direction: batchForm.direction,
          difficulty: batchForm.difficulty,
          year: parseInt(batchForm.year),
          questionCount: validQuestions.length,
          viewCount: 0,
          favoriteCount: 0,
          expanded: false,
          questions: validQuestions.map((q, index) => ({
            id: Date.now() + index,
            content: q.content,
            type: '简答题',
            difficulty: batchForm.difficulty
          }))
        }
        
        // 更新企业数据
        company.value.questionSets.push(questionSet)
        company.value.totalQuestions += validQuestions.length
        
        ElMessage.success('题集添加成功')
        batchUploadDialogVisible.value = false
        
        // 重置表单
        Object.keys(batchForm).forEach(key => {
          if (key === 'questions') {
            batchForm[key] = [{ content: '' }]
          } else {
            batchForm[key] = ''
          }
        })
        
      } catch (error) {
        ElMessage.error('添加失败：' + error.message)
      } finally {
        batchUploading.value = false
      }
    }
    
    // 生命周期
    onMounted(() => {
      initCompanyData()
    })
    
    return {
      loading,
      company,
      yearOptions,
      filters,
      filteredQuestionSets,
      batchUploadDialogVisible,
      batchUploading,
      batchFormRef,
      batchForm,
      batchRules,
      batchInputDialogVisible,
      batchInputText,
      handleFilterChange,
      toggleQuestions,
      favoriteSet,
      startSetPractice,
      startCompanyMockInterview,
      viewQuestionDetail,
      practiceQuestion,
      showBatchUploadDialog,
      addQuestion,
      removeQuestion,
      showBatchInputDialog,
      handleBatchInput,
      previewQuestionSet,
      handleBatchSubmit,
      getTechLabel,
      getDifficultyLabel,
      getDifficultyType,
      getAvgDifficulty
    }
  }
}
</script>

<style scoped>
.company-detail {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  background: #f8f9fa;
  min-height: 100vh;
}

/* 面包屑导航 */
.breadcrumb-nav {
  margin-bottom: 24px;
}

.breadcrumb-link {
  color: #1890ff;
  cursor: pointer;
}

.breadcrumb-link:hover {
  text-decoration: underline;
}

/* 公司信息头部 */
.company-header {
  display: flex;
  align-items: flex-start;
  gap: 24px;
  margin-bottom: 32px;
  padding: 32px;
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.08);
}

.company-logo {
  width: 80px;
  height: 80px;
  border-radius: 16px;
  overflow: hidden;
  flex-shrink: 0;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.company-logo img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.company-info {
  flex: 1;
  min-width: 0;
}

.company-title {
  margin: 0 0 12px 0;
  font-size: 28px;
  font-weight: 700;
  color: #262626;
  display: flex;
  align-items: center;
  gap: 12px;
}

.plus-badge {
  background: #ff9800;
  color: #fff;
  font-size: 14px;
  font-weight: 500;
  padding: 4px 12px;
  border-radius: 8px;
  line-height: 1;
}

.company-desc {
  margin: 0 0 20px 0;
  font-size: 16px;
  color: #666;
  line-height: 1.6;
}

.company-stats {
  display: flex;
  gap: 32px;
}

.stat-item {
  text-align: center;
}

.stat-number {
  display: block;
  font-size: 24px;
  font-weight: 700;
  color: #1890ff;
  margin-bottom: 4px;
}

.stat-label {
  font-size: 14px;
  color: #666;
}

.company-actions {
  display: flex;
  flex-direction: column;
  gap: 12px;
  flex-shrink: 0;
}

/* 筛选区域 */
.filter-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
  margin-bottom: 24px;
  padding: 20px;
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}

.filter-left {
  display: flex;
  gap: 16px;
}

.filter-right {
  flex-shrink: 0;
}

/* 题集列表 */
.question-sets {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.question-set-card {
  background: #fff;
  border-radius: 16px;
  overflow: hidden;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
  transition: all 0.3s ease;
}

.question-set-card:hover {
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.12);
}

.set-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 24px;
  border-bottom: 1px solid #f0f0f0;
}

.set-title-section {
  flex: 1;
  min-width: 0;
}

.set-title {
  margin: 0 0 12px 0;
  font-size: 20px;
  font-weight: 600;
  color: #262626;
  line-height: 1.4;
}

.set-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  align-items: center;
}

.set-year, .question-count, .view-count {
  font-size: 14px;
  color: #666;
}

.set-actions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}

/* 题目列表 */
.questions-list {
  padding: 24px;
  background: #fafafa;
}

.empty-questions {
  text-align: center;
  padding: 40px;
}

.question-item {
  display: flex;
  gap: 16px;
  padding: 16px;
  margin-bottom: 16px;
  background: #fff;
  border-radius: 12px;
  border: 1px solid #e8e8e8;
  transition: all 0.3s ease;
}

.question-item:hover {
  border-color: #1890ff;
  box-shadow: 0 2px 8px rgba(24, 144, 255, 0.1);
}

.question-item:last-child {
  margin-bottom: 0;
}

.question-number {
  font-size: 16px;
  font-weight: 600;
  color: #1890ff;
  flex-shrink: 0;
  width: 32px;
  text-align: center;
}

.question-content {
  flex: 1;
  min-width: 0;
}

.question-text {
  margin: 0 0 12px 0;
  font-size: 16px;
  color: #262626;
  line-height: 1.6;
}

.question-tags {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.question-actions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
  align-items: flex-start;
}

/* 空状态 */
.empty-state {
  text-align: center;
  padding: 60px;
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}

/* 表单样式 */
.form-section {
  margin-bottom: 24px;
}

.form-section h4 {
  margin: 0 0 16px 0;
  font-size: 16px;
  font-weight: 600;
  color: #262626;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.section-actions {
  display: flex;
  gap: 8px;
}

.questions-list-form {
  max-height: 400px;
  overflow-y: auto;
  border: 1px solid #e8e8e8;
  border-radius: 8px;
  padding: 16px;
  background: #fafafa;
}

.question-item-form {
  display: flex;
  gap: 12px;
  margin-bottom: 16px;
  align-items: flex-start;
}

.question-item-form:last-child {
  margin-bottom: 0;
}

.question-number {
  font-size: 14px;
  font-weight: 500;
  color: #666;
  margin-top: 8px;
  flex-shrink: 0;
  width: 24px;
}

.question-input {
  flex: 1;
}

.question-actions {
  flex-shrink: 0;
  margin-top: 4px;
}

/* 批量输入提示 */
.batch-input-tip {
  margin-bottom: 12px;
  padding: 12px;
  background: #f6ffed;
  border: 1px solid #b7eb8f;
  border-radius: 8px;
}

.batch-input-tip p {
  margin: 0;
  font-size: 14px;
  color: #52c41a;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .company-detail {
    padding: 16px;
  }
  
  .company-header {
    flex-direction: column;
    text-align: center;
  }
  
  .company-stats {
    justify-content: center;
  }
  
  .filter-section {
    flex-direction: column;
    align-items: stretch;
  }
  
  .filter-left {
    flex-direction: column;
    gap: 12px;
  }
  
  .set-header {
    flex-direction: column;
    gap: 16px;
  }
  
  .set-actions {
    width: 100%;
    justify-content: center;
  }
  
  .question-item {
    flex-direction: column;
    gap: 12px;
  }
  
  .question-actions {
    justify-content: center;
  }
}

/* 滚动条样式 */
.questions-list-form::-webkit-scrollbar {
  width: 6px;
}

.questions-list-form::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

.questions-list-form::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

.questions-list-form::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}
</style> 