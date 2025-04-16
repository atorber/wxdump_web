<script setup lang="ts">
import http from "@/utils/axios.js";
import ProgressBar from "@/components/utils/ProgressBar.vue";
import {defineEmits, onMounted, ref, watch} from "vue";
import {ElTable, ElTableColumn, ElMessage, ElMessageBox} from "element-plus";
import type {Action} from 'element-plus'
import router from "@/router";

interface wxinfo {
  pid: string;
  version: string;
  account: string;
  mobile: string;
  nickname: string;
  mail: string;
  wxid: string;
  wx_dir: string;
  key: string;
}

interface LocalWxid {
  wxid: string;
}

const percentage = ref(0);
const startORstop = ref(-1);  // 用于进度条的开始和停止 0表示0% 1表示100%

const init_type = ref("");

const is_init = ref(false);
const wxinfoData = ref<wxinfo[]>([]);

const oneWx = ref("");
const decryping = ref(false);
const isErrorShow = ref(false);
const isUseKey = ref("false");

const merge_path = ref("");
const wx_path = ref("");
const key = ref("");
const my_wxid = ref("");

const local_wxids = ref<LocalWxid[]>([]);

const db_init = (init: boolean) => {
  if (init) {
    localStorage.setItem('isDbInit', "t");
    router.push('/');
    ElMessage({
      type: 'success',
      message: '初始化成功！',
    })
  }
}

const showError = (error: unknown) => {
  const errorMessage = error instanceof Error ? error.message : String(error);
  ElMessageBox.alert(errorMessage, '错误', {
    confirmButtonText: '确认',
    callback: (action: Action) => {
      ElMessage({
        type: 'error',
        message: `操作: ${action}`,
      })
    },
  });
}

// ** 是否使用key的初始化** START
const init_key = async () => {
  if (decryping.value) {
    console.log("正在解密中，请稍后再试！")
    return;
  }
  decryping.value = true;
  try {
    decryping.value = true;
    startORstop.value = 0; // 进度条开始
    let reqdata = {
      "wx_path": wx_path.value,
      "key": key.value,
      "my_wxid": my_wxid.value
    }
    const body_data = await http.post('/api/ls/init_key', reqdata);
    is_init.value = body_data.is_init;
    if (body_data.is_init) {
      percentage.value = 100; // 进度条 100%
    }
    decryping.value = false;
    db_init(body_data.is_init);
  } catch (error) {
    percentage.value = 0; // 进度条 0%
    isErrorShow.value = true;
    decryping.value = false;
    showError(error);
    return [];
  }
  decryping.value = false;
}

const init_nokey = async () => {
  try {
    let reqdata = {
      "wx_path": wx_path.value,
      "merge_path": merge_path.value,
      "my_wxid": my_wxid.value
    }
    const body_data = await http.post('/api/ls/init_nokey', reqdata);
    is_init.value = body_data.is_init;
    if (body_data.is_init) {
      percentage.value = 100; // 进度条 100%
    }
    decryping.value = false;
    db_init(body_data.is_init);
  } catch (error) {
    percentage.value = 0; // 进度条 0%
    isErrorShow.value = true;
    decryping.value = false;
    showError(error);
    return [];
  }
  decryping.value = false;
}
// ** 是否使用key的初始化** END

// ** 使用上次数据部分** START
const selectLastWx = async (row: wxinfo) => {
  my_wxid.value = row.wxid;
}

const get_init_last_local_wxid = async () => {
  try {
    const body_data = await http.post('/api/ls/init_last_local_wxid');
    local_wxids.value = body_data.local_wxids.map((item: string) => {
      return {wxid: item}
    });
    if (local_wxids.value.length === 1) {
      my_wxid.value = local_wxids.value[0].wxid;
      await init_last();
    }
  } catch (error) {
    showError(error);
    return [];
  }
}

const init_last = async () => {
  try {
    let reqdata = {
      "wx_path": wx_path.value,
      "merge_path": merge_path.value,
      "my_wxid": my_wxid.value
    }
    const body_data = await http.post('/api/ls/init_last', reqdata);
    is_init.value = body_data.is_init;
    if (body_data.is_init) {
      percentage.value = 100; // 进度条 100%
      decryping.value = false;
      db_init(body_data.is_init);
    } else {
      isErrorShow.value = true;
      decryping.value = false;
      ElMessageBox.alert("未发现上次的设置数据！", '错误', {
        confirmButtonText: '确认',
        callback: (action: Action) => {
          init_type.value = "";// 刷新
        },
      })
    }
    decryping.value = false;
  } catch (error) {
    isErrorShow.value = true;
    decryping.value = false;
    showError(error);
    return [];
  }
  decryping.value = false;
}

// ** 使用上次数据部分** END

// **自动解密微信部分** START 查看有多少个微信正在登录 ， 并调用init_key解密初始化
const get_wxinfo = async () => {
  try {
    wxinfoData.value = await http.post('/api/ls/wxinfo');
    if (wxinfoData.value.length === 1) {
      selectWx(wxinfoData.value[0]);
      oneWx.value = " (检测到只有一个微信，将在5秒后自动选择) ";
      setTimeout(okWx, 5000);
    }
  } catch (error) {
    showError(error);
    return [];
  }
}

const selectWx = async (row: wxinfo) => {
  merge_path.value = "";
  wx_path.value = row.wx_dir;
  key.value = row.key;
  my_wxid.value = row.wxid;
}

const okWx = () => {
  if (wx_path.value === '' && key.value === '' && my_wxid.value === '') {
    console.log("请填写完整信息! ")
    return;
  }
  if (decryping.value) {
    console.log("正在解密...，请稍后再试！")
    return;
  }
  init_key();
}

// **自动解密微信部分**  END 查看有多少个微信正在登录 ， 并调用init_key解密初始化

// 监测isAutoShow是否为aoto，如果是则执行get_wxinfo
watch(init_type, (val) => {
  if (val === 'auto') {
    get_wxinfo();
  } else if (val === 'custom') {
    // init();
  } else if (val === 'last') {
    get_init_last_local_wxid();
  }
})

</script>

<template>
  <div class="db-init-container">
    <!-- 初始选择界面 -->
    <div v-if="init_type === ''" class="init-options">
      <el-card class="option-card" @click="init_type = 'last'">
        <div class="option-content">
          <el-radio v-model="init_type" label="last" class="option-radio" />
          <div class="option-title">使用历史数据</div>
          <div class="option-desc">使用上次的配置信息进行初始化</div>
        </div>
      </el-card>

      <el-card class="option-card" @click="init_type = 'auto'">
        <div class="option-content">
          <el-radio v-model="init_type" label="auto" class="option-radio" />
          <div class="option-title">自动解密已登录微信</div>
          <div class="option-desc">自动检测并解密当前登录的微信</div>
        </div>
      </el-card>

      <el-card class="option-card" @click="init_type = 'custom'">
        <div class="option-content">
          <el-radio v-model="init_type" label="custom" class="option-radio" />
          <div class="option-title">自定义文件位置</div>
          <div class="option-desc">手动指定数据库和密钥位置</div>
        </div>
      </el-card>
    </div>

    <!-- 上次数据 -->
    <el-card v-else-if="init_type==='last'" class="main-card">
      <template #header>
        <div class="card-header">
          <span class="title">选择要查看的微信</span>
        </div>
      </template>

      <el-table 
        :data="local_wxids" 
        @current-change="selectLastWx" 
        highlight-current-row 
        class="wx-table"
      >
        <el-table-column prop="wxid" label="微信原始ID" min-width="200" />
      </el-table>

      <div class="action-area">
        <el-button 
          type="primary" 
          @click="init_last"
          :loading="decryping"
          class="submit-btn"
        >
          确定
        </el-button>
      </div>
    </el-card>

    <!-- 自动解密和显示 -->
    <el-card v-else-if="init_type==='auto'" class="main-card">
      <template #header>
        <div class="card-header">
          <span class="title">选择要查看的微信</span>
          <span class="subtitle">(会清空work下对应wxid数据)</span>
        </div>
      </template>

      <ProgressBar v-if="decryping" :startORstop="startORstop" class="progress-bar" />

      <template v-else>
        <el-table 
          :data="wxinfoData" 
          @current-change="selectWx" 
          highlight-current-row 
          class="wx-table"
        >
          <el-table-column prop="pid" label="进程ID" min-width="80" />
          <el-table-column prop="version" label="微信版本" min-width="100" />
          <el-table-column prop="account" label="账号" min-width="120" />
          <el-table-column prop="nickname" label="昵称" min-width="120" />
          <el-table-column prop="wxid" label="微信原始ID" min-width="200" />
        </el-table>

        <div class="action-area">
          <el-button 
            type="primary" 
            @click="okWx"
            :loading="decryping"
            class="submit-btn"
          >
            确定{{ oneWx }}
          </el-button>
        </div>
      </template>
    </el-card>

    <!-- 自定义参数 -->
    <el-card v-else-if="init_type==='custom'" class="main-card">
      <template #header>
        <div class="card-header">
          <span class="title">自定义文件位置</span>
        </div>
      </template>

      <ProgressBar v-if="decryping" :startORstop="startORstop" class="progress-bar" />

      <template v-else>
        <el-radio-group v-model="isUseKey" class="radio-group">
          <el-radio label="true">使用 KEY</el-radio>
          <el-radio label="false">不使用 KEY</el-radio>
        </el-radio-group>

        <div class="description">
          <div v-if="isUseKey=='false'" class="desc-item">
            <h4>说明：</h4>
            <p>1、表示数据库已解密并合并</p>
            <p>2、合并后的数据库需要包含(MediaMSG,MSG,MicroMsg,OpenIMMsg)这些数据库合并的内容</p>
          </div>
          <div v-if="isUseKey=='true'" class="desc-item">
            <h4>说明：</h4>
            <p>1、自动根据key解密微信文件夹下的数据库</p>
            <p>2、必须保证key正确，否则解密失败</p>
          </div>
        </div>

        <el-divider />

        <el-form label-position="top" class="custom-form">
          <el-form-item v-if="isUseKey=='true'" label="密钥key">
            <el-input 
              v-model="key" 
              placeholder="密钥key (64位)"
              clearable
            />
          </el-form-item>

          <el-form-item v-if="isUseKey=='false'" label="merge_all.db 文件路径">
            <el-input 
              v-model="merge_path" 
              placeholder="(MediaMSG.db,MSG.db,MicroMsg.db,OpenIMMsg.db)合并后的数据库"
              clearable
            />
          </el-form-item>

          <el-form-item label="微信文件夹路径">
            <el-input 
              v-model="wx_path" 
              placeholder="C:\***\WeChat Files\wxid_*******"
              clearable
            />
          </el-form-item>

          <el-form-item label="微信原始id">
            <el-input 
              v-model="my_wxid" 
              placeholder="wxid_*******"
              clearable
            />
          </el-form-item>

          <el-form-item>
            <el-button 
              type="primary" 
              @click="isUseKey=='true' ? init_key() : init_nokey()"
              :loading="decryping"
              class="submit-btn"
            >
              确定
            </el-button>
          </el-form-item>
        </el-form>
      </template>
    </el-card>
  </div>
</template>

<style scoped>
.db-init-container {
  min-height: 100vh;
  background-color: #f5f7fa;
  padding: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.init-options {
  display: flex;
  gap: 20px;
  justify-content: center;
  flex-wrap: wrap;
}

.option-card {
  width: 300px;
  cursor: pointer;
  transition: all 0.3s;
}

.option-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.option-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  text-align: center;
}

.option-radio {
  margin-bottom: 10px;
}

.option-title {
  font-size: 18px;
  font-weight: bold;
  margin-bottom: 10px;
  color: #303133;
}

.option-desc {
  font-size: 14px;
  color: #606266;
}

.main-card {
  width: 90%;
  max-width: 1000px;
  border-radius: 8px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.title {
  font-size: 20px;
  font-weight: bold;
  color: #303133;
}

.subtitle {
  font-size: 14px;
  color: #909399;
  margin-left: 10px;
}

.wx-table {
  margin: 20px 0;
}

.action-area {
  margin-top: 20px;
  display: flex;
  justify-content: center;
}

.submit-btn {
  width: 100%;
}

.radio-group {
  margin-bottom: 20px;
}

.description {
  margin: 20px 0;
  padding: 15px;
  background-color: #f5f7fa;
  border-radius: 4px;
}

.desc-item h4 {
  margin: 0 0 10px 0;
  color: #303133;
}

.desc-item p {
  margin: 5px 0;
  color: #606266;
}

.custom-form {
  margin-top: 20px;
}

.progress-bar {
  margin: 20px 0;
}

:deep(.el-input__wrapper) {
  box-shadow: 0 0 0 1px #dcdfe6;
}

:deep(.el-input__wrapper:hover) {
  box-shadow: 0 0 0 1px #c0c4cc;
}

:deep(.el-input__wrapper.is-focus) {
  box-shadow: 0 0 0 1px #409EFF;
}
</style>