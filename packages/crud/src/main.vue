<template>
  <div class="crud-container pull-auto">
    <el-table :data="tableData" @row-click="rowClick" @row-dblclick="rowDblclick" :height="tableOption.height" ref="table" :style="{width:setPx(tableOption.width,'99.5%')}" style="margin:0 auto;box-sizing:border-box;" :border="tableOption.border" v-loading="tableLoading" @selection-change="selectionChange">
      <!-- 选择框 -->
      <template v-if="tableOption.selection">
        <el-table-column type="selection" width="55">
        </el-table-column>
      </template>
      <!-- 序号 -->
      <template v-if="tableOption.index">
        <el-table-column type="index" width="50">
        </el-table-column>
      </template>
      <!-- 循环列 -->
      <template v-for="(column,index) in tableOption.column">
        <el-table-column :width="column.width" :label="column.label" :fixed="column.fixed" :sortable="column.sortable" v-if="!column.hide">
          <template slot-scope="scope">
            <slot :row="scope.row" :dic="setDic(column.dicData,DIC[column.dicData])" :name="column.prop" v-if="column.solt"></slot>
            <span v-else-if="!column.overHidden" v-html="detail(scope.row,column)"></span>
            <el-popover v-else trigger="hover" placement="top">
              <p>{{column.label}}: {{ scope.row[column.prop]}}</p>
              <div slot="reference" class="name-wrapper">
                <span v-html="detail(scope.row,column)" class="crud--overflow"></span>
              </div>
            </el-popover>
          </template>
        </el-table-column>
      </template>
      <el-table-column label="操作" :width="tableOption.menuWidth?tableOption.menuWidth:240" v-if="tableOption.menu==undefined?true:tableOption.menu">
        <template slot-scope="scope">
          <template v-if="tableOption.menu!=undefined?tableOption.menu:true">
            <el-button type="primary" icon="el-icon-edit" size="small" @click="rowEdit(scope.row,scope.$index)" v-if="tableOption.editBtn==undefined?true:tableOption.editBtn">编 辑</el-button>
            <el-button type="danger" icon="el-icon-delete" size="small" @click="rowDel(scope.row,scope.$index)" v-if="tableOption.delBtn==undefined?true:tableOption.delBtn">删 除</el-button>
          </template>
          <slot :row="scope.row" name="menu" :index="scope.$index"></slot>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination v-if="tableOption.page!=undefined?tableOption.page:true" class="crud-pagination pull-right" :current-page.sync="page.currentPage" :background="page.background?page.background:true" :page-size="page.pageSize" @current-change="currentChange" layout="total, sizes, prev, pager, next, jumper" :total="page.total"></el-pagination>
    <el-dialog :modal-append-to-body="false" :append-to-body="true" :title="boxType==0?'新增':'编辑'" :visible.sync="boxVisible" width="50%" :before-close="boxhandleClose">
      <el-form ref="tableForm" :model="tableForm" label-width="80px" :rules="tableFormRules">
        <el-row :gutter="20" :span="24">
          <template v-for="(column,index) in tableOption.column">
            <el-col :span="column.span||12" v-if="!column.visdiplay">
              <el-form-item :label="column.label" :prop="column.prop" :label-width="setPx(column.labelWidth,tableOption.labelWidth || 80)">
                <slot :value="tableForm[column.prop]" :column="column" :dic="setDic(column.dicData,DIC[column.dicData])" :name="column.prop+'Form'" v-if="column.formsolt"></slot>
                <component :is="getComponent(column.type)" v-else v-model="tableForm[column.prop]" :size="column.size" :clearable="column.clearable" :type="column.type" :minRows="column.minRows" :maxRows="column.maxRows" :placeholder="column.label" :dic="setDic(column.dicData,DIC[column.dicData])" :disabled="boxType==0?(column.addDisabled!=undefined?column.addDisabled:column.disabled):column.disabled"></component>
              </el-form-item>
            </el-col>
          </template>
        </el-row>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button type="primary" @click="rowUpdate" v-if="boxType==1">修 改</el-button>
        <el-button type="primary" @click="rowSave" v-else>新 增</el-button>
        <el-button @click="boxVisible = false">取 消</el-button>
      </span>
    </el-dialog>

  </div>
</template>
<script>
import crud from "avue/mixins/crud.js";
export default {
  name: "AvueCrud",
  mixins: [crud()],
  components: {},
  data() {
    return {
      boxVisible: false,
      boxType: 0,
      DIC: {},
      tableForm: {},
      tableFormRules: {},
      tableIndex: -1,
      tableSelect: []
    };
  },
  created() {
    //规则初始化
    this.rulesInit();
    //初始化字典
    this.dicInit();
  },
  watch: {
    tableOption: {
      handler(n, o) {
        this.rulesInit();
      },
      deep: true
    },
    tableForm: {
      handler(n, o) {
        this.formVal();
      },
      deep: true
    }
  },
  mounted() {},
  props: {
    value: {
      type: Object,
      default: () => {
        return {};
      }
    },
    beforeClose: Function,
    beforeOpen: Function,
    page: {
      type: Object,
      default() {
        return {
          total: 0, //总页数
          currentPage: 0, //当前页数
          pageSize: 10, //每页显示多少条
          background: true //背景颜色
        };
      }
    },
    tableLoading: {
      type: Boolean,
      default: false
    },
    tableData: {
      type: Array,
      required: true,
      default: () => {
        return [];
      }
    },
    tableOption: {
      type: Object,
      required: true,
      default: () => {
        return [];
      }
    }
  },
  methods: {
    rulesInit() {
      this.tableFormRules = {};
      this.tableOption.column.forEach(ele => {
        if (ele.rules) this.tableFormRules[ele.prop] = ele.rules;
      });
    },
    dicInit() {
      this.GetDic(this.tableOption.dic).then(data => {
        this.DIC = data;
      });
    },
    formVal() {
      this.$emit("input", this.tableForm);
    },
    formInit() {
      const list = this.tableOption.column;
      let from = {};
      list.forEach(ele => {
        if (
          ele.type == "checkbox" ||
          ele.type == "radio" ||
          ele.type == "cascader"
        ) {
          from[ele.prop] = [];
        } else if (ele.type == "number") {
          from[ele.prop] = 0;
        } else {
          from[ele.prop] = "";
        }
      });
      this.tableForm = Object.assign({}, from);
    },
    //页码回掉
    currentChange(val) {
      this.$emit("current-change", val);
    },

    // 选中实例
    toggleSelection(rows) {
      if (rows) {
        rows.forEach(row => {
          this.$refs.table.toggleRowSelection(row);
        });
      } else {
        this.$refs.table.clearSelection();
      }
    },
    //选择回调
    selectionChange(val) {
      this.tableSelect = val;
      this.$emit("selection-change", val);
    },
    //行双击
    rowDblclick(row, event) {
      this.$emit("row-dblclick", row, event);
    },

    //行单机
    rowClick(row, event, column) {
      this.$emit("row-click", row, event, column);
    },
    //处理数据
    detail(row, column) {
      let result = "";
      if (column.dataDetail) {
        if (column.type) {
          result = this.findByvalue(
            typeof column.dicData == "string"
              ? this.DIC[column.dicData]
              : column.dicData,
            row[column.prop]
          );
        } else {
          result = row[column.prop];
        }
        result = column.dataDetail(row);
      } else {
        if (column.type) {
          result = this.findByvalue(
            typeof column.dicData == "string"
              ? this.DIC[column.dicData]
              : column.dicData,
            row[column.prop]
          );
        } else {
          result = row[column.prop];
        }
      }
      return result;
    },
    // 新增
    rowAdd() {
      //form表单初始化
      this.formInit();
      this.boxType = 0;
      if (typeof this.beforeClose === "function") this.beforeOpen(this.show);
      else this.show();
    },
    // 编辑
    rowEdit(row, index) {
      this.tableForm = Object.assign({}, row);
      this.tableIndex = index;
      this.boxType = 1;
      if (typeof this.beforeClose === "function") this.beforeOpen(this.show);
      else this.show();
    },
    // 删除
    rowDel(row, index) {
      this.$emit("row-del", row, index);
    },
    //保存
    rowSave() {
      this.$refs["tableForm"].validate(valid => {
        if (valid) {
          this.$emit("row-save", this.tableForm, this.hide);
        }
      });
    },
    //更新
    rowUpdate() {
      this.$refs["tableForm"].validate(valid => {
        if (valid) {
          const index = this.tableIndex;
          this.$emit("row-update", this.tableForm, index, this.hide);
        }
      });
    },
    //显示表单
    show(cancel) {
      if (cancel !== true) {
        this.boxVisible = true;
      }
    },
    //隐藏表单
    hide(cancel) {
      if (cancel !== false) {
        this.boxVisible = false;
      }
    },
    //窗口关闭处理事件
    boxhandleClose() {
      //释放form表单
      this.tableForm = {};
      if (typeof this.beforeClose === "function") this.beforeClose(this.hide);
      else this.hide();
    }
  }
};
</script>

<style lang="scss" scoped>
.crud-container {
  // padding: 8px 10px;
}
.crud-pagination {
  margin-top: 15px;
  padding: 10px 20px;
}
.crud-header {
  margin-bottom: 10px;
  & > .el-button {
    padding: 12px 25px;
  }
}
.crud--overflow {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  display: block;
}
</style>