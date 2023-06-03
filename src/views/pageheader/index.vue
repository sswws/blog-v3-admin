<script setup lang="ts">
import { ref, onMounted, reactive } from "vue";
import { Plus, Edit, Delete } from "@element-plus/icons-vue";
import Upload from "@/components/Upload/upload.vue";
import {
  getPageHeaderList,
  addOrUpdatePageHeader,
  deletePageHeader
} from "@/api/pageheader";
import { imgUpload } from "@/api/site";
import { message } from "@/utils/message";
import { cloneDeep } from "@pureadmin/utils";
import { ElLoading } from "element-plus";
const coverV = (rule, value, cb) => {
  if (!pageHeaderForm.coverList.length) {
    return new Error("请上传背景");
  }
  cb();
};

defineOptions({
  name: "pageHeader"
});
const pageHeaderList = ref([]);

// 背景新增 编辑有关
const dialogVisible = ref(false);
const pageHeaderFormRef = ref();
const pageHeaderForm = reactive({
  id: "",
  route_name: "",
  bg_url: "",
  coverList: []
});
const primaryPageHeaderForm = reactive({ ...pageHeaderForm });
const pageHeaderFormRules = reactive({
  route_name: {
    required: true,
    message: "请输入路由名称",
    trigger: ["blur"]
  },
  pageHeader_cover: {
    required: true,
    validator: coverV,
    trigger: ["change"]
  }
});

const closeDialog = () => {
  pageHeaderFormRef.value.resetFields();
  Object.assign(pageHeaderForm, primaryPageHeaderForm);
  dialogVisible.value = false;
};

// 提交表单
const submitForm = async formEl => {
  if (!formEl) return;
  await formEl.validate(async valid => {
    if (valid) {
      // 先进行图片上传
      if (!pageHeaderForm.coverList[0].id) {
        const upLoadLoading = ElLoading.service({
          fullscreen: true,
          text: "图片上传中"
        });
        const imgRes = await imgUpload(pageHeaderForm.coverList[0]);
        if (imgRes.code == 0) {
          const { url } = imgRes.result;
          pageHeaderForm.bg_url = url;
        }
        upLoadLoading.close();
      } else {
        pageHeaderForm.bg_url = pageHeaderForm.coverList[0].url;
      }

      const res = await addOrUpdatePageHeader(pageHeaderForm);
      if (res.code == 0) {
        await pageGetPageHeader();
        message(`${pageHeaderForm.id ? "修改" : "新增"}成功`, {
          type: "success"
        });
        formEl.resetFields();
        Object.assign(pageHeaderForm, primaryPageHeaderForm);
        dialogVisible.value = false;
      }
    }
  });
};

const operate = (type, val?) => {
  switch (type) {
    case "edit":
      Object.assign(pageHeaderForm, cloneDeep(val));
      dialogVisible.value = true;
      break;
    case "delete":
      deletePageHeaderById(val.id, val.bg_url);
      break;
    case "add":
      dialogVisible.value = true;
      break;
    default:
      return;
  }
};

const deletePageHeaderById = async (id, url) => {
  const res = await deletePageHeader({ id, url });
  if (res.code == 0) {
    pageGetPageHeader();
    message("删除成功", { type: "success" });
  }
};

const pageGetPageHeader = async () => {
  const res = await getPageHeaderList();
  if (res.code == 0) {
    pageHeaderList.value = res.result.map(l => {
      return {
        id: l.id,
        route_name: l.route_name,
        bg_url: l.bg_url,
        coverList: [
          {
            id: 1,
            url: l.bg_url
          }
        ]
      };
    });
  }
};

onMounted(() => {
  pageGetPageHeader();
});
</script>

<template>
  <el-card>
    <template #header>
      <div class="flex_r_b">
        背景管理
        <el-button
          type="primary"
          :icon="Plus"
          size="small"
          @click="operate('add')"
          >新增</el-button
        >
      </div>
      <div class="tip">路由名称就是当前页面打印的route.name</div>
    </template>
    <el-row class="page-header">
      <el-col
        style="padding: 10px"
        :xs="24"
        :sm="6"
        v-for="item in pageHeaderList"
        :key="item.id"
      >
        <div class="page-header-card">
          <div class="flex_r_b pad5">
            <div class="page-header-title">
              <span class="mr-[5px]">路由名称: </span>
              <span :title="item.route_name">{{ item.route_name }}</span>
            </div>
            <div class="operate flex_r_b">
              <el-icon color="#66b1ff" size="18" @click="operate('edit', item)">
                <Edit />
              </el-icon>
              <el-popconfirm
                :title="`删除背景-${item.route_name}?`"
                icon-color="#66b1ff"
                @confirm="operate('delete', item)"
              >
                <template #reference>
                  <el-icon color="#f56c6c" size="16" style="margin-left: 5px">
                    <Delete />
                  </el-icon>
                </template>
              </el-popconfirm>
            </div>
          </div>
          <el-image
            class="page-header-image animate__animated animate__fadeIn"
            :src="item.bg_url"
            fit="cover"
            loading="lazy"
            preview-teleported
            :preview-src-list="item.coverList.map(v => v.url)"
          />
        </div>
      </el-col>
    </el-row>
    <el-dialog
      :title="pageHeaderForm.id ? '编辑背景' : '新增背景'"
      v-model="dialogVisible"
      :width="400"
      :before-close="closeDialog"
    >
      <el-form
        ref="pageHeaderFormRef"
        :inline="true"
        :model="pageHeaderForm"
        :rules="pageHeaderFormRules"
        class="bg-bg_color w-[99/100]"
        label-width="100"
      >
        <el-form-item class="form-item80" label="路由名称" prop="route_name">
          <el-input
            v-model="pageHeaderForm.route_name"
            placeholder="请输入路由名称"
            clearable
            maxlength="25"
          />
        </el-form-item>
        <el-form-item class="cover" label="背景图" prop="pageHeader_cover">
          <Upload
            v-model:fileList="pageHeaderForm.coverList"
            :width="200"
            :height="200"
            :limit="1"
          />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button size="small" @click="closeDialog()">取消</el-button>
        <el-button
          size="small"
          type="danger"
          plain
          @click="submitForm(pageHeaderFormRef)"
        >
          保存
        </el-button>
      </template>
    </el-dialog>
  </el-card>
</template>

<style lang="scss" scoped>
.page-header {
  height: calc(100vh - 220px);
  overflow: auto;

  &-card {
    position: relative;
    box-shadow: 0px 0px 12px rgba(0, 0, 0, 0.12);
  }
  &-title {
    font-weight: 600;
    width: 70%;
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    transition: all 0.5s;
    color: #606266;
    vertical-align: baseline;

    &:hover {
      cursor: pointer;
      color: #66b1ff;
    }
  }
  .route {
    font-weight: 600;
  }

  &-image {
    vertical-align: bottom;
    width: 100%;
    position: relative;
    transition: all 0.5s;

    &:hover {
      cursor: pointer;
    }
  }
}

.tip {
  font-size: 0.7rem;
  color: #f56c6c;
}

.page-header::-webkit-scrollbar {
  display: none;
}

:deep(.el-card__body) {
  padding: 0 !important;
}

.pagination {
  margin: 0 0 10px 10px;
}

.flex_r_b {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.operate {
  &:hover {
    cursor: pointer;
  }
}

.pad5 {
  padding: 5px;
}

.form-item {
  &50 {
    width: 50%;
    font-weight: bold;
  }

  &80 {
    width: 80%;
    font-weight: bold;
  }
}

.cover {
  font-weight: bold;

  :deep(.el-form-item__content) {
    width: 200px;
    height: 200px;
  }

  :deep(.el-upload-list__item) {
    width: 200px !important;
    height: 200px !important;
    margin: 0 !important;
    border: none !important;
    border-radius: 0;
  }

  :deep(.el-upload-list--picture-card) {
    width: 200px !important;
    height: 200px !important;
    margin: 0 !important;
    border: none !important;
    border-radius: 0;
  }

  :deep(.el-upload--picture-card) {
    width: 200px !important;
    height: 200px !important;
    border-radius: 0;
  }
}

@media screen and (max-width: 798px) {
  .page-header-image {
    height: 240px;
  }
}

@media screen and (min-width: 799px) {
  .page-header-image {
    height: 200px;
  }
}
</style>
