<template>
  <div class="app-container">
    <!-- 第一个卡片 - 查询条件 -->
    <el-card shadow="always" class="search-wrapper">
      <div class="top-container">
        <el-form-item>
          <el-select
            class="drop-down-for-timeRange"
            v-model="formData.timeRange"
            @change="handleTimeRangeChange"
            clearable
            placeholder="Time range"
          >
            <el-option label="Recent 30 min" value="Recent 30 min" />
            <el-option label="Recent 1 hour" value="Recent 1 hour" />
            <el-option label="Recent 1 day" value="Recent 1 day" />
            <el-option label=" Recent 1 week" value="Recent 1 week" />
            <!-- Add more options as needed -->
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-select
            class="drop-down-for-rowLimit"
            v-model="formData.searchLevel"
            clearable
            placeholder="请选择日志类型"
          >
            <el-option v-for="item in options" :key="item.value" :label="item.label" :value="item.value" />
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-input
            class="input-for-filterKeyword"
            v-model="formData.searchStr"
            placeholder="请输入日志关键字"
            clearable
          />
        </el-form-item>
        <el-form-item>
          <el-button class="button-for-query" type="primary" round @click="fetchData">搜索</el-button>
        </el-form-item>
      </div>
    </el-card>

    <el-card shadow="always">
      <div class="table-wrapper">
        <el-table
          :data="displayedData"
          :header-cell-style="{ background: '#FFFFFF', color: '#606266' }"
          @sort-change="fetchData"
          border
          style="width: 100%"
        >
          <el-table-column
            :prop="item"
            :label="item"
            v-for="item in tableHeader"
            :sortable="'custom'"
            :key="item"
            show-overflow-tooltip
          />
        </el-table>
      </div>
      <!-- 分页 -->
      <div class="pager-wrapper">
        <el-pagination
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
          :current-page="currentPage"
          :page-sizes="[10, 20, 50]"
          :page-size="pageSize"
          :total="tableData.length"
          layout="sizes, prev, pager, next, jumper"
          style="float: right"
        />
      </div>
    </el-card>
  </div>
</template>

<script lang="ts">
import axios from "axios"

export default {
  name: "LogTable",
  data() {
    return {
      tableHeader: [],
      tableData: [],
      displayedData: [],
      currentPage: 1,
      pageSize: 10,
      formData: {
        searchStartTime: "",
        searchEndTime: "",
        searchHostname: "",
        searchLevel: "",
        searchMsg: "",
        searchStr: "",
        timeRange: "Recent 1 day"
      },
      options: [
        {
          value: "info",
          label: "info"
        },
        {
          value: "debug",
          label: "debug"
        },
        {
          value: "warn",
          label: "warn"
        },
        {
          value: "critical",
          label: "critical"
        },
        {
          value: "error",
          label: "error"
        }
      ]
    }
  },
  methods: {
    fetchData(column: any) {
      this.tableData = []
      //this.tableHeader = []
      this.displayedData = []
      this.currentPage = 1
      //this.pageSize = 8 // 排序的时候每页大小又回到8，bug
      const queryHost = "219.216.65.171"
      const queryPort = "8086"
      const apiUrl = "http://" + queryHost + ":" + queryPort + "/query"
      const queryDB = "monitor"
      let querySql = "select * from err_log where 1 = 1"

      /* 附加查询条件 */
      if ("" != this.formData.searchLevel && null != this.formData.searchLevel) {
        querySql += " " + " AND level = '" + this.formData.searchLevel + "'"
      }
      // 循环遍历表头
      if ("" != this.formData.searchStr && null != this.formData.searchStr) {
        const orConditions = this.tableHeader
          .filter((header) => header !== "time") // 排除 time 列
          .map((header) => `(${header} =~ /.*${this.formData.searchStr}.*/)`)
        querySql += " AND (" + orConditions.join(" OR ") + ")"
      }
      // if ("" != this.formData.searchStartTime && null != this.formData.searchStartTime) {
      //   querySql += " " + " AND time >= " + this.formData.searchStartTime
      // }
      // if ("" != this.formData.searchEndTime && null != this.formData.searchEndTime) {
      //   querySql += " " + " AND time <= " + this.formData.searchEndTime
      // }

      //附加排序
      //console.log(column);
      if (column != undefined) {
        if ("" != column.prop && null != column.prop) {
          //console.log(column.prop)
          //console.log(column.order)
          querySql += " " + " order by " + column.prop
          if (column.order == "ascending") {
            querySql += " " + " ASC "
          } else if (column.order == "descending") {
            querySql += " " + " DESC "
          }
        }
      }
      console.log(querySql)

      axios
        .get(apiUrl, {
          params: {
            db: queryDB,
            pretty: true,
            q: querySql
          }
        })
        .then((response) => {
          const results = response.data.results[0]
          if (results && results.series) {
            this.tableHeader = results.series[0].columns
            this.tableData = results.series[0].values.map((row: any[]) => {
              return results.series[0].columns.reduce((rowData: any, columnName: string, index: number) => {
                rowData[columnName] = row[index]
                return rowData
              }, {})
            })
            //console.log(this.tableData)
            this.updateDisplayedData()
          }
        })
        .catch((error) => {
          console.error("Error fetching data:", error)
        })
    },
    updateDisplayedData() {
      const startIndex = (this.currentPage - 1) * this.pageSize
      this.displayedData = this.tableData.slice(startIndex, startIndex + this.pageSize)
    },
    handleSizeChange(newSize: number) {
      this.pageSize = newSize
      this.currentPage = 1
      this.updateDisplayedData()
    },
    handleCurrentChange(newPage: number) {
      this.currentPage = newPage
      this.updateDisplayedData()
    },
    executeClear() {
      this.formData.searchHostname = ""
      this.formData.searchLevel = ""
      this.formData.searchMsg = ""
      this.formData.searchStartTime = ""
      this.formData.searchEndTime = ""
    },
    handleTimeRangeChange(value: string) {
      this.formData.timeRange = value
      this.fetchData(undefined)
    }
  },
  mounted() {
    this.fetchData(undefined)
  }
}
</script>

<style lang="scss" scoped>
.card-wrapper {
  margin-bottom: 20px;
  :deep(.el-card__body) {
    padding-bottom: 2px;
  }
}
.input-search {
  width: 450px;
}

.input-container {
  margin: 10px;
  width: 300px;
  /* 调整宽度以对齐两行内容 */
}

.bottom-container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
}
.search-wrapper {
  margin-bottom: 20px;

  :deep(.el-card__body) {
    padding-bottom: 2px;
  }
}

.table-wrapper {
  margin-bottom: 20px;
}

.pager-wrapper {
  display: flex;
  justify-content: flex-end;
}

.top-container {
  display: flex;
}

.drop-down-for-timeRange {
  margin-left: 4px;
  width: 170px;
  border-radius: 5px;
  background-color: rgba(255, 255, 255, 1);
  color: rgba(79, 79, 79, 1);
  font-size: 20px;
  text-align: center;
  font-family: Microsoft Yahei;
  border: 1px solid rgba(154, 154, 154, 1);
}

.drop-down-for-databases {
  left: 25px;
  width: 189px;
  border-radius: 5px;
  background-color: rgba(255, 255, 255, 1);
  color: rgba(79, 79, 79, 1);
  font-size: 20px;
  text-align: left;
  font-family: Microsoft Yahei;
  border: 1px solid rgba(187, 187, 187, 1);
}

.drop-down-for-rowLimit {
  left: 50px;
  width: 150px;
  border-radius: 5px;
  background-color: rgba(255, 255, 255, 1);
  color: rgba(79, 79, 79, 1);
  font-size: 20px;
  text-align: left;
  font-family: Microsoft Yahei;
  border: 1px solid rgba(187, 187, 187, 1);
}

.input-for-filterKeyword {
  left: 75px;
  width: 339px;
  border-radius: 5px;
  color: rgba(136, 136, 136, 1);
  font-size: 14px;
  text-align: left;
  font-family: Roboto;
  border: 1px solid rgba(187, 187, 187, 1);
}

.button-for-query {
  margin-left: 100px;
  width: 70px;
  border-radius: 8px;
  background-color: #1a73e8;
  color: rgba(255, 255, 255, 1);
  font-size: 14px;
  text-align: center;
  font-family: Microsoft Yahei;
}

.button-for-reset {
  margin-left: 15px;
  width: 70px;
  border-radius: 8px;
  background-color: rgba(255, 255, 255, 1);
  color: rgba(79, 79, 79, 1);
  font-size: 14px;
  text-align: center;
  font-family: Microsoft Yahei;
  border: 1px solid rgba(154, 154, 154, 1);
  transition: box-shadow 0.35s;
  /* 添加过渡效果，使得阴影效果更加平滑 */
}

.button-for-reset:active {
  box-shadow: 2px 2px 8px rgba(0, 0, 0, 0.2);
  /* 点击时添加更大的阴影效果 */
}
</style>
