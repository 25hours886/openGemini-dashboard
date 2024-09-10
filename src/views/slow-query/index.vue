<template>
  <div class="app-container">
    <!-- 第一个卡片 - 查询条件 -->
    <el-card shadow="always" class="search-wrapper">
      <div class="top-container">
        <el-form-item>
          <el-select class="drop-down-for-timeRange" v-model="formData.timeRange" @change="handleTimeRangeChange"
            placeholder="Time range">
            <el-option label="Recent 30 min" value="Recent 30 min" />
            <el-option label="Recent 1 hour" value="Recent 1 hour" />
            <el-option label="Recent 1 day" value="Recent 1 day" />
            <el-option label=" Recent 10 week" value="Recent 1 week" />
            <!-- Add more options as needed -->
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-select class="drop-down-for-databases" v-model="formData.selectedDatabaseValue"
            @change="handleDatabaseChange" clearable placeholder="All databases">
            <el-option v-for="database in selectedDatabase" :label="database" :value="database" :key="database" />
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-select class="drop-down-for-rowLimit" v-model="formData.rowLimit" @change="handleRowLimitChange"
            placeholder="Limit rows">
            <el-option label="Limit 10" value="Limit 10" />
            <el-option label="Limit 20" value="Limit 20" />
            <el-option label="Limit 50" value="Limit 50" />
            <!-- Add more options as needed -->
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-input class="input-for-filterKeyword" v-model="formData.filterKeyword" placeholder="关键字过滤" />
        </el-form-item>
        <el-form-item>
          <el-button class="button-for-query" @click="handleSearch">
            <el-icon style="vertical-align: middle">
              <Search />
            </el-icon>
            <span style="vertical-align: middle"> 查询 </span>
          </el-button>
        </el-form-item>
        <el-form-item>
          <el-button class="button-for-reset" @click="resetSearch">
            <el-icon style="vertical-align: middle">
              <Refresh />
            </el-icon>
            <span style="vertical-align: middle"> 重置 </span>
          </el-button>
        </el-form-item>
      </div>
    </el-card>

    <!-- 第二个卡片 - 慢查询展示 -->
    <el-card shadow="always">
      <div class="table-wrapper">
        <el-table :data="tableData" border>
          <el-table-column prop="qid" label="序号" align="center" width="55" />
          <el-table-column prop="query" label="query语句" align="center" />
          <el-table-column prop="database" label="数据库名称" align="center" width="95" />
          <el-table-column prop="endTime" label="结束时间" align="center" width="105" sortable />
          <el-table-column prop="totalTime" label="执行时间" align="center" width="105" sortable />
          <el-table-column prop="host" label="hostname" align="center" width="160" />
        </el-table>
      </div>
      <div class="pager-wrapper">
        <!-- 分页 -->
        <el-pagination @size-change="handleSizeChange" @current-change="handleCurrentChange" :current-page="currentPage"
          :page-sizes="[10, 20, 50]" :page-size="pageSize" :total="queryData.length"
          layout="sizes, prev, pager, next, jumper" style="float: right" />
      </div>
    </el-card>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted } from 'vue'
import axios from "axios"
import { Search, Refresh } from "@element-plus/icons-vue"

interface LogItem {
  qid: number;
  query: string;
  database: string;
  endTime: number;
  totalTime: string;
  host: string;
}

const tableData = ref<LogItem[]>([]);
const selectedDatabase = ref<string[]>([]);
// const filterKeyword = ref<string>('');
const formData = ref({
  timeRange: 'Recent 1 day',
  selectedDatabaseValue: '',
  rowLimit: 'Limit 10',
  filterKeyword: ''
});

const queryData = ref<LogItem[]>([]);
const pageSize = ref<number>(10);
const currentPage = ref<number>(1);

onMounted(() => {
  fetchData();
});

const fetchData = () => {
  const rowCount = parseInt(formData.value.rowLimit.split(' ')[1].trim());
  axios.get<any>('http://219.216.65.171:8086/query', {
    params: {
      db: 'test4',
      q: `select * from logs limit ${rowCount}`
    }
  })
    .then(response => {
      // 处理响应数据
      const result = response.data.results[0];
      if (result && result.series && result.series[0] && result.series[0].values) {
        // 数据库连接成功
        console.log('Successfully connected to the database.');
        const data = result.series[0].values.map((row: any) => ({
          qid: row[4],
          query: row[5],
          database: row[1],
          endTime: new Date(row[0]),
          endTimeString: new Date(row[0]).toLocaleString('zh-CN', { // 格式化后的时间
            timeZone: 'Asia/Shanghai',
            year: 'numeric',
            month: '2-digit',
            day: '2-digit',
            hour: '2-digit',
            minute: '2-digit',
            second: '2-digit'
          }),
          totalTime: row[2],
          host: row[3]
        })).filter((item: LogItem) => {
          const startTime = getStartTime(formData.value.timeRange);
          return new Date(item.endTime).getTime() >= startTime;
        });

        selectedDatabase.value = Array.from(new Set(data.map((item: any) => item.database)));

        let filteredData = data;

        if (formData.value.selectedDatabaseValue !== '') {
          filteredData = data.filter((item: LogItem) => {
            return item.database === formData.value.selectedDatabaseValue;
          });
        }
        console.log("filteredData1:", filteredData);

        const keyword = formData.value.filterKeyword.toLowerCase();
        console.log("filterKeyword:", formData.value.filterKeyword.toLowerCase());
        if (keyword) {
          filteredData = filteredData.filter((item: any) => {
            return item.query.toLowerCase().includes(keyword) || item.database.toLowerCase().includes(keyword) || item.host.toLowerCase().includes(keyword);
          });
        }
        tableData.value = filteredData.map((item: any) => ({
          ...item,
          endTime: item.endTimeString // 显示时使用格式化后的时间
        }));

        queryData.value = tableData.value;
        handleSizeChange(pageSize.value);
        handleCurrentChange(currentPage.value);

      } else {
        console.error('Invalid response format:', result);
      }
    })
    .catch(error => {
      console.error('Error fetching data:', error);
    });
};

// 时间戳格式化函数
const formatTimestamp = (timestamp: number) => {
  const date = new Date(timestamp / 1000000); // 将纳秒时间戳转换为毫秒时间戳
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');
  const hours = String(date.getHours()).padStart(2, '0');
  const minutes = String(date.getMinutes()).padStart(2, '0');
  const seconds = String(date.getSeconds()).padStart(2, '0');
  return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
};

const getStartTime = (timeRange: string) => {
  const now = new Date().getTime();
  switch (timeRange) {
    case 'Recent 30 min':
      return now - 30 * 60 * 1000;
    case 'Recent 1 hour':
      return now - 60 * 60 * 1000;
    case 'Recent 1 day':
      return now - 24 * 60 * 60 * 1000;
    case 'Recent 10 week':
      return now - 10 * 7 * 24 * 60 * 60 * 1000;
    default:
      return 0; // 默认值，可根据实际情况修改
  }
};

const handleSearch = () => {
  fetchData();
};

const resetSearch = () => {
  formData.value.timeRange = 'Recent 1 day';
  formData.value.selectedDatabaseValue = '';
  formData.value.rowLimit = 'Limit 10';
  fetchData();
};

const handleTimeRangeChange = (value: string) => {
  formData.value.timeRange = value;
  formData.value.selectedDatabaseValue = '';
  selectedDatabase.value = [];
  //formData.value.rowLimit = 'Limit 10';
  fetchData();
  console.log('database:', formData.value.selectedDatabaseValue);
};

const handleDatabaseChange = (value: string) => {
  formData.value.selectedDatabaseValue = value;
  console.log('database:', formData.value.selectedDatabaseValue);
  fetchData();
};

const handleRowLimitChange = (value: string) => {
  formData.value.rowLimit = value;
  console.log('rowLimit:', formData.value.rowLimit);
  fetchData();
};

const updateDisplayedQueries = () => {
  const startIndex = (currentPage.value - 1) * pageSize.value;
  tableData.value = queryData.value.slice(startIndex, startIndex + pageSize.value);
};

const handleSizeChange = (newSize: number) => {
  pageSize.value = newSize;
  currentPage.value = 1;
  updateDisplayedQueries();
};

const handleCurrentChange = (newPage: number) => {
  currentPage.value = newPage;
  updateDisplayedQueries();
};

const handleClearForKeyword = () => {
  console.log("this is clear");
  formData.value.filterKeyword = '';
}
</script>

<style lang="scss" scoped>
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
  width: 120px;
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
  background-color: #1A73E8;
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
