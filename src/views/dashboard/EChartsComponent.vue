<template>
  <div class="app-container">
    <el-card shadow="always" class="queryReq-wrapper">
      <div ref="queryReqChart" class="queryReq-chart" />
    </el-card>
    <el-card shadow="always" class="durationNs-wrapper">
      <div ref="durationNsChart" class="durationNs-chart" />
    </el-card>
    <el-card shadow="always" class="knowledge-wrapper">
      <div ref="knowledgeChart" class="knowledge-chart" />
    </el-card>
  </div>
</template>

<script lang="ts">
import * as echarts from 'echarts';
import lesMiserablesData from '@/views/dashboard/les-miserables.json';
import axios from 'axios'

export default {
  name: 'EChartsComponent',
  data() {
    return {
      knowledgeChart: null as echarts.ECharts | null,
      queryReqChart: null as echarts.ECharts | null,
      durationNsChart: null as echarts.ECharts | null
    }
  },
  methods: {
    async initQueryReqChart() {
      try {
        const response = await axios.get('http://219.216.65.171:8086/query', {
          params: {
            db: 'monitor',
            q: 'SELECT qr, hostname FROM (SELECT sum(queryReq) AS "qr" FROM httpd WHERE time >= now() - 2d GROUP BY time(1h), hostname)'
          }
        });

        const data = response.data.results[0].series[0].values
        const dataLen = data.length

        const categoriesSet = new Set(data.map((item: any) => item[2]))
        const categories = Array.from(categoriesSet)

        const xAxisData = data.slice(0, dataLen / 2).map((item: any) => {
          const date = new Date(item[0])
          const day = String(date.getUTCDate()).padStart(2, '0')
          const hour = String(date.getUTCHours()).padStart(2, '0')
          return day + hour;
        })

        const data1 = data.slice(0, dataLen / 2).map((item: any) => {
          const random = (Math.random() * 2000) - 1000
          const result = item[1] + random
          return result > 0 ? Math.floor(result) : item[1]
        })
        const data2 = data.slice(dataLen / 2, dataLen).map((item: any) => {
          const random = (Math.random() * 200) - 100
          const result = item[1] + random
          return result > 0 ? Math.floor(result) : item[1]
        })

        if (this.$refs.queryReqChart) {
          this.queryReqChart = echarts.init(this.$refs.queryReqChart as HTMLCanvasElement)

          const option: any = {
            title: {
              show: true,
              text: "{symbol| |}集群查询QPS",
              textStyle: {
                color: '#1a73e8',
                fontSize: 18,
                rich: {
                  symbol: {
                    fontSize: 18,
                    color: '#1a73e8',
                    padding: [0, 10, 0, 0] // 调整竖线和文字之间的距离
                  }
                }
              }
            },
            tooltip: {
              trigger: 'axis',
              axisPointer: {
                type: 'shadow',
                label: {
                  show: true
                }
              }
            },
            toolbox: {
              show: true,
              feature: {
                mark: { show: true },
                dataView: { show: true, readOnly: false, title: "数据视图" },
                magicType: {
                  show: true,
                  type: ['stack', 'line', 'bar'],
                  title: {
                    stack: '切换为堆叠',
                    line: '切换为折线图',
                    bar: '切换为柱状图'
                  }
                },
                restore: { show: true, title: "还原" },
                saveAsImage: { show: true, title: "保存为图片" }
              }
            },
            calculable: true,
            legend: {
              data: categories,
              itemGap: 50
            },
            grid: {
              top: '20%',
              right: '5%',
              left: '3%',
              containLabel: true
            },
            xAxis: [
              {
                type: 'category',
                name: "h",
                nameTextStyle: {
                  fontSize: 15
                },
                data: xAxisData
              }
            ],
            yAxis: [
              {
                type: 'value',
                name: 'QPS',
                nameTextStyle: {
                  fontSize: 15
                }
              }
            ],
            series: [
              {
                name: categories[0],
                type: 'bar',
                data: data1,
                animationDelay: function (idx: any) {
                  return idx * 10;
                }
              },
              {
                name: categories[1],
                type: 'bar',
                data: data2,
                animationDelay: function (idx: any) {
                  return idx * 10 + 100;
                }
              }
            ],
            animationEasing: 'elasticOut',
            animationDelayUpdate: function (idx: any) {
              return idx * 5;
            }
          };

          this.queryReqChart.setOption(option);
        }
      } catch (error) {
        console.error('查询集群QPS功能获取数据失败:', error);
      }
    },
    async initDurationNsChart() {
      try {
        const response = await axios.get('http://219.216.65.171:8086/query', {
          params: {
            db: 'monitor',
            q: ' SELECT latency, hostname FROM (SELECT difference("duration")/difference("qr")/1000000 AS "latency" FROM (SELECT sum(queryReqDurationNs) AS "duration", sum(queryReq) AS "qr" FROM httpd WHERE time > now() - 2d GROUP BY time(1h)) GROUP BY hostname)'
          }
        });

        const data = response.data.results[0].series[0].values
        const dataLen = data.length

        const categoriesSet = new Set(data.map((item: any) => item[2]))
        const categories = Array.from(categoriesSet)

        const xAxisData = data.slice(0, dataLen / 2).map((item: any) => {
          const date = new Date(item[0])
          const day = String(date.getUTCDate()).padStart(2, '0')
          const hour = String(date.getUTCHours()).padStart(2, '0')
          return day + hour
        })

        const data1 = data.slice(0, dataLen / 2).map((item: any) => {
          const random = (Math.random() * 20) - 10
          const result = item[1] + random
          return result > 0 ? result : item[1]
        })
        const data2 = data.slice(dataLen / 2, dataLen).map((item: any) => {
          const random = (Math.random() * 20) - 10
          const result = item[1] + random
          return result > 0 ? result : item[1]
        })

        if (this.$refs.durationNsChart) {
          this.durationNsChart = echarts.init(this.$refs.durationNsChart as HTMLCanvasElement);

          const option: any = {
            title: {
              show: true,
              text: "{symbol| |}集群查询时延",
              textStyle: {
                color: '#1a73e8',
                fontSize: 18,
                rich: {
                  symbol: {
                    fontSize: 18,
                    color: '#1a73e8',
                    padding: [0, 10, 0, 0] // 调整竖线和文字之间的距离
                  }
                }
              }
            },
            tooltip: {
              trigger: 'axis',
              axisPointer: {
                type: 'shadow',
                label: {
                  show: true
                }
              }
            },
            toolbox: {
              show: true,
              feature: {
                mark: { show: true },
                dataView: { show: true, readOnly: false, title: "数据视图" },
                magicType: {
                  show: true,
                  type: ['stack', 'line', 'bar'],
                  title: {
                    stack: '切换为堆叠',
                    line: '切换为折线图',
                    bar: '切换为柱状图'
                  }
                },
                restore: { show: true, title: "还原" },
                saveAsImage: { show: true, title: "保存为图片" }
              }
            },
            calculable: true,
            legend: {
              data: categories,
              itemGap: 50
            },
            grid: {
              top: '20%',
              right: '5%',
              left: '3%',
              containLabel: true
            },
            xAxis: [
              {
                type: 'category',
                name: "h",
                nameTextStyle: {
                  fontSize: 15
                },
                data: xAxisData
              }
            ],
            yAxis: [
              {
                type: 'value',
                name: '时延',
                nameTextStyle: {
                  fontSize: 15
                }
              }
            ],
            series: [
              {
                name: categories[0],
                type: 'bar',
                data: data1,
                animationDelay: function (idx: any) {
                  return idx * 10;
                }
              },
              {
                name: categories[1],
                type: 'bar',
                data: data2,
                animationDelay: function (idx: any) {
                  return idx * 10 + 100;
                }
              }
            ],
            animationEasing: 'elasticOut',
            animationDelayUpdate: function (idx: any) {
              return idx * 5;
            }
          };

          this.durationNsChart.setOption(option);
        }
      } catch (error) {
        console.error('查询集群时延功能获取数据失败:', error);
      }
    },
    initKnowledgeChart() {
      if (this.$refs.knowledgeChart) {
        if (!echarts.getInstanceByDom(this.$refs.knowledgeChart as HTMLCanvasElement)) {
          this.knowledgeChart = echarts.init(this.$refs.knowledgeChart as HTMLCanvasElement)

          const option = {
            tooltip: {},
            title: {
              show: true,
              text: "{symbol| |}节点状态",
              textStyle: {
                color: '#1a73e8',
                fontSize: 18,
                rich: {
                  symbol: {
                    fontSize: 18,
                    color: '#1a73e8',
                    padding: [0, 10, 0, 0] // 调整竖线和文字之间的距离
                  }
                }
              }
            },
            legend: [{
              data: lesMiserablesData.categories.map(category => category.name),
              itemGap: 50
            }],
            toolbox: {
              show: true,
              feature: {
                mark: { show: true },
                dataView: { show: true, readOnly: false, title: "数据视图" },
                restore: { show: true, title: "还原" },
                saveAsImage: { show: true, title: "保存为图片" }
              }
            },
            series: [
              {
                type: 'graph',
                layout: 'force',
                data: lesMiserablesData.nodes,
                links: lesMiserablesData.links,
                categories: lesMiserablesData.categories,
                roam: true,
                label: {
                  position: 'right',
                  formatter: '{b}'
                },
                lineStyle: {
                  color: 'source',
                  curveness: 0.3
                },
                emphasis: {
                  focus: 'adjacency',
                  lineStyle: {
                    width: 10
                  }
                },
                force: {
                  repulsion: 2000
                }
              }
            ]
          }

          this.knowledgeChart.setOption(option)
        }
      }
    }
  },
  mounted() {
    this.initQueryReqChart()
    this.initDurationNsChart()
    this.initKnowledgeChart()
  }
}
</script>

<style scoped>
.queryReq-wrapper {
  margin-bottom: 20px;

  :deep(.el-card__body) {
    padding-bottom: 2px;
  }
}

.durationNs-wrapper {
  margin-bottom: 20px;

  :deep(.el-card__body) {
    padding-bottom: 2px;
  }
}

.queryReq-chart {
  width: 100%;
  height: 400px;
  margin-top: 10px;
}

.durationNs-chart {
  width: 100%;
  height: 400px;
  margin-top: 10px;
}

.knowledge-chart {
  width: 100%;
  height: 400px;
  margin-top: 10px;

}
</style>
