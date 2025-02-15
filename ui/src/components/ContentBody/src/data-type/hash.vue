<template>
  <div class="hash">
    <div class="key-data-title">
      <span><span class="highlight">{{ k }}</span> {{ lang.title }}</span>
      <div style="display: flex">
        <el-input v-model="pattern" :placeholder="lang.queryPlaceHolder" style="margin-right: 8px;width: 140px"/>
        <el-button @click="searchFn" type="primary">{{ lang.q }}</el-button>
        <el-button @click="clearFn" type="primary" :disabled="!pattern">{{ lang.r }}</el-button>
        <el-button @click="newFn" type="primary">{{ lang.a }}</el-button>
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
          <el-table-column :show-overflow-tooltip="true" prop="key" :label="lang.tableHeader[0]"></el-table-column>
          <el-table-column :show-overflow-tooltip="true" prop="value" :label="lang.tableHeader[1]"></el-table-column>
          <el-table-column width="180px" :label="lang.tableHeader[2]">
            <template #default="{row}">
              <el-button type="text" @click="modifyFn(row)">{{ lang.tableAction[0] }}</el-button>
              <el-divider direction="vertical"></el-divider>
              <el-button type="text" @click="deleteFn(row.key)">{{ lang.tableAction[1] }}</el-button>
              <el-divider direction="vertical"></el-divider>
              <el-button type="text" @click="lookFn(row.value)">{{ lang.tableAction[2] }}</el-button>
            </template>
          </el-table-column>
        </el-table>
      </div>
      <div style="position: absolute;left: 8px;bottom: 15px">
        <span style="padding-right: 8px;color: gray">{{ lang.pager.size }}</span>
        <el-select v-model="pager.count" style="width: 80px;margin-right: 8px">
          <el-option label="20" :value="20"></el-option>
          <el-option label="50" :value="50"></el-option>
          <el-option label="100" :value="100"></el-option>
          <el-option label="200" :value="200"></el-option>
          <el-option label="500" :value="500"></el-option>
          <el-option label="1000" :value="1000"></el-option>
        </el-select>
        <el-button type="primary" size="mini" :loading="dataLoading" :disabled="isEnd" @click="scanFn">{{
            isEnd ? lang.pager.end : lang.pager.next
          }}
        </el-button>
        <el-button type="primary" size="mini" :loading="dataLoading" v-if="pager.cursor !== 0 || isEnd"
                   @click="resetFn">{{ lang.pager.reset }}
        </el-button>
      </div>
    </div>

    <el-dialog :close-on-click-modal="false" :title="lang.dialog.modifyTitle" v-model="modifyVisible" width="380px">
      <el-form label-width="80px" :model="modifyFormData" ref="modifyForm">
        <el-form-item :label="lang.dialog.form.field" prop="field"
                      :rules="{required:true,message:lang.dialog.ruleMsg.field,trigger:'blur'}">
          <el-input readonly="readonly" v-model="modifyFormData.field"/>
        </el-form-item>
        <el-form-item prop="value" :label="lang.dialog.form.value"
                      :rules="{required: true,message:lang.dialog.ruleMsg.value,trigger:'blur'}">
          <el-input v-model="modifyFormData.value" :placeholder="lang.dialog.form.field"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="modifyOkFn" :loading="modifyLoading">{{ lang.dialog.button[0] }}</el-button>
          <el-button type="danger" :loading="modifyLoading" @click="modifyVisible = false">{{ lang.dialog.button[1] }}
          </el-button>
        </el-form-item>
      </el-form>
    </el-dialog>

    <el-dialog :close-on-click-modal="false" :title="lang.dialog.addTitle" v-model="addVisible" width="380px">
      <el-form label-width="80px" :model="addFormData" ref="addForm">
        <el-form-item :label="lang.dialog.form.field" prop="field"
                      :rules="{required:true,message:lang.dialog.ruleMsg.field,trigger:'blur'}">
          <el-input :placeholder="lang.dialog.form.field" v-model="addFormData.field"/>
        </el-form-item>
        <el-form-item prop="value" :label="lang.dialog.form.value"
                      :rules="{required: true,message:lang.dialog.ruleMsg.value,trigger:'blur'}">
          <el-input v-model="addFormData.value" :placeholder="lang.dialog.form.value"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="newOkFn" :loading="addLoading">{{ lang.dialog.button[0] }}</el-button>
          <el-button type="danger" :loading="addLoading" @click="addVisible = false">{{ lang.dialog.button[1] }}
          </el-button>
        </el-form-item>
      </el-form>
    </el-dialog>
  </div>
</template>

<script>
import {request} from ':/tools/invoke';
import {WebviewWindow} from '@tauri-apps/api/window';
import {mapGetters} from "vuex";

export default {
  name: 'hash',
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
      modifyFormData: {},
      modifyVisible: false,
      modifyLoading: false,
      addFormData: {},
      addVisible: false,
      addLoading: false
    }
  },
  computed: {
    ...mapGetters(['i18n']),
    lang() {
      return this.i18n.mainPage.content.dataPage.hash
    }
  },
  created() {
    this.fetchHashData()
  },
  methods: {
    refresh() {
      this.isEnd = false
      this.pager = {
        cursor: 0,
        count: 20
      }
      this.fetchHashData()
    },
    fetchHashData() {
      this.dataLoading = true
      request('scan_hash_data', {
        key: this.k,
        pattern: this.pattern || '*',
        ...this.pager
      })
          .then(res => {
            this.tableData = res.data.records.map(kv => ({
              key: String.fromCharCode(...kv.key),
              value: String.fromCharCode(...kv.value),
            }))

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
      this.fetchHashData()
    },
    resetFn() {
      this.pager.cursor = 0
      this.isEnd = false
      this.fetchHashData()
    },
    searchFn() {
      this.pager.cursor = 0
      this.isEnd = false
      this.fetchHashData()
    },
    clearFn() {
      this.pattern = ''
      this.pager.cursor = 0
      this.isEnd = false
      this.fetchHashData()
    },
    deleteFn(field) {

      this.$confirm('确定要删除吗？')
          .then(() => {
            return request('del_hash_field', {field, key: this.k})
          })
          .then(res => {
            this.pager.cursor = this.currentCursor
            this.alert.success(res.msg)
            this.fetchHashData()
          })
    },
    modifyFn(row) {
      this.modifyVisible = true
      this.modifyFormData = {field: row.key}
    },
    modifyOkFn() {
      this.$refs.modifyForm.validate(valid => {
        if (!valid) return
        this.modifyFormData.key = this.k
        this.modifyLoading = true
        request('modify_hash_value_by_field', this.modifyFormData)
            .then(res => {
              this.pager.cursor = this.currentCursor
              this.alert.success(res.msg)
              this.fetchHashData()
              this.modifyVisible = false
            })
            .finally(() => this.modifyLoading = false)
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
        request('add_new_field_to_exists_hash', this.addFormData)
            .then(res => {
              this.pager.cursor = this.currentCursor
              this.alert.success(res.msg)
              this.fetchHashData()
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
