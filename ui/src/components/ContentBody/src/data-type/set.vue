<template>
  <div class="set">
    <div class="key-data-title">
      <span><span class="highlight">{{ k }}</span> 查询结果</span>
      <div style="display: flex">
        <el-input v-model="pattern" placeholder="请输入查询表达式" style="margin-right: 8px;width: 140px"/>
        <el-button @click="searchFn" type="primary">查询</el-button>
        <el-button @click="clearFn" type="primary" :disabled="!pattern">清除</el-button>
        <el-button @click="newFn" type="primary">新增</el-button>
      </div>
    </div>
    <div class="key-data-scroller" style="position: relative">
      <div style="position: absolute;left: 0;top: 0;right: 0;bottom: 70px">
        <el-table :data="tableData" height="100%" style="width: 100%">
          <el-table-column label="#" width="80px">
            <template #default="scope">
              <span>{{ scope.$index }}</span>
            </template>
          </el-table-column>
          <el-table-column :show-overflow-tooltip="true" prop="member" label="成员"></el-table-column>
          <el-table-column width="110px" label="操作">
            <template #default="{row}">
              <el-button type="text" @click="deleteFn(row.member)">删除</el-button>
              <el-divider direction="vertical"></el-divider>
              <el-button type="text" @click="lookFn(row.member)">查看</el-button>
            </template>
          </el-table-column>
        </el-table>
      </div>
      <div style="position: absolute;left: 8px;bottom: 15px">
        <span style="padding-right: 8px;color: gray">每页条数</span>
        <el-select v-model="pager.count" style="width: 80px;margin-right: 8px">
          <el-option label="20" :value="20"></el-option>
          <el-option label="50" :value="50"></el-option>
          <el-option label="100" :value="100"></el-option>
          <el-option label="200" :value="200"></el-option>
          <el-option label="500" :value="500"></el-option>
        </el-select>
        <el-button type="primary" size="mini" :loading="dataLoading" :disabled="isEnd" @click="scanFn">{{
            isEnd ? '已到最后一页' : '下一页'
          }}
        </el-button>
        <el-button type="primary" size="mini" :loading="dataLoading" v-if="pager.cursor !== 0 || isEnd" @click="resetFn">重置游标
        </el-button>
      </div>
    </div>
    <el-dialog :close-on-click-modal="false" title="新增" v-model="addVisible" width="380px">
      <el-form label-width="80px" :model="addFormData" ref="addForm">
        <el-form-item prop="member" label="成员" :rules="{required: true,message:'成员不能为空',trigger:'blur'}">
          <el-input v-model="addFormData.member" placeholder="请输入成员"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="newOkFn" :loading="addLoading">确定</el-button>
          <el-button type="danger" :loading="addLoading" @click="addVisible = false">取消</el-button>
        </el-form-item>
      </el-form>
    </el-dialog>
  </div>
</template>

<script>
import {request} from ':/tools/invoke';
import {WebviewWindow} from '@tauri-apps/api/window';

export default {
  name: 'set',
  props: {
    k: {
      type: String,
      required: true,
      cursor: 0
    }
  },
  data() {
    return {
      dataLoading: false,
      tableData: [],
      isEnd: false,
      pattern: '',
      currentCursor: 0,
      pager: {
        cursor: 0,
        count: 20
      },
      addFormData: {},
      addVisible: false,
      addLoading: false
    }
  },
  created() {
    this.fetchSetData()
  },
  methods: {
    refresh() {
      this.isEnd = false
      this.pager = {
        cursor: 0,
        count: 20
      }
      this.fetchSetData()
    },
    fetchSetData() {
      this.dataLoading = true
      request('scan_set_data', {
        key: this.k,
        pattern: this.pattern || '*',
        ...this.pager
      })
          .then(res => {
            this.tableData = res.data.records

            if (res.data.cursor === 0) {
              this.isEnd = true
              return
            }
            this.pager.cursor = res.data.cursor
          })
          .finally(() => this.dataLoading = false)
    },
    scanFn() {
      this.currentCursor = this.pager.cursor
      this.fetchSetData()
    },
    resetFn() {
      this.pager.cursor = 0
      this.isEnd = false
      this.fetchSetData()
    },
    searchFn() {
      this.pager.cursor = 0
      this.isEnd = false
      this.fetchSetData()
    },
    clearFn() {
      this.pattern = ''
      this.pager.cursor = 0
      this.isEnd = false
      this.fetchSetData()
    },
    deleteFn(member) {
      this.$confirm('确定要删除吗？')
          .then(() => {
            return request('del_set_member', {member, key: this.k})
          })
          .then(res => {
            this.pager.cursor = this.currentCursor
            this.alert.success(res.msg)
            this.fetchSetData()
          })
    },
    newFn() {
      this.addFormData = {}
      this.addVisible = true
    },
    newOkFn() {
      this.$refs.addForm.validate(valid => {
        if (!valid) return
        this.addFormData.key = this.k
        this.addLoading = true
        request('add_new_member_to_exists_set', this.addFormData)
            .then(res => {
              this.pager.cursor = this.currentCursor
              this.alert.success(res.msg)
              this.fetchSetData()
              this.addVisible = false
            })
            .finally(() => this.addLoading = false)
      })
    },
    lookFn(row) {
    let w = new WebviewWindow(Math.random().toString(36).slice(2), {
      url: '/#/look-more',
      title: '数据详情',
      alwaysOnTop: true
    });
    w.listen('data', () => {
      w.emit('data', row)
    })
  }
  }
}
</script>

<style scoped lang="scss">
@include common-title;
</style>
