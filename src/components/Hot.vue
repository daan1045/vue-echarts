<template>
  <div class="com-container">
    <div class="com-chart" ref="hot_ref"></div>
    <span
      class="icon-arrow icon-left iconfont icon-arrow-down"
      @click="toLeft"
      :style="comStyle"
    ></span>
    <span
      class="icon-arrow icon-right iconfont icon-arrow-up"
      @click="toRight"
      :style="comStyle"
    ></span>
    <span class="my-title" :style="comStyle">{{ myTitle }}</span>
  </div>
</template>

<script>
import { mapState } from 'vuex'
import { getThemeValue } from '@/utils/theme_utils'
export default {
  data () {
    return {
      chartInstance: null,
      allData: null,
      currentIndex: 0, // 当前所展示出的一级分类数据
      titleFontSize: 0
    }
  },
  created () {
    // 在组件创建完成之后，进行回调函数的注册
    this.$socket.registerCallBack('hotData', this.getData)
  },
  mounted () {
    this.initChart()
    // this.getData()
    // 发送数据给服务器，告诉服务器，我现在需要数据
    this.$socket.send({
      action: 'getData',
      socketType: 'hotData',
      chartName: 'hot',
      value: ''
    })
    window.addEventListener('resize', this.screenAdapter)
    this.screenAdapter()
  },
  destroyed () {
    window.removeEventListener('resize', this.screenAdapter)
    // 在组件销毁的时候，进行回调函数的取消
    this.$socket.unRegisterCallBack('hotData')
  },
  computed: {
    myTitle () {
      if (!this.allData) {
        return ''
      } else {
        return this.allData[this.currentIndex].name
      }
    },
    comStyle () {
      return {
        fontSize: this.titleFontSize + 'px',
        color: getThemeValue(this.theme).titleColor
      }
    },
    ...mapState(['theme'])
  },
  methods: {
    initChart () {
      this.chartInstance = this.$echarts.init(this.$refs.hot_ref, this.theme)
      const initOption = {
        title: {
          text: '▎热销商品销售金额占比统计',
          left: 20,
          top: 20
        },
        legend: {
          top: '15%',
          icon: 'circle'
        },
        tooltip: {
          show: true,
          formatter: (arg) => {
            const childrenArr = arg.data.children
            let total = null
            childrenArr.forEach((item) => {
              total += item.value
            })
            let retStr = ''
            childrenArr.forEach((item) => {
              retStr += `
              ${item.name}: ${parseInt((item.value / total) * 100) + '%'}<br/>`
            })
            return retStr
          }
        },
        series: [
          {
            type: 'pie',
            label: {
              show: false
            },
            emphasis: {
              label: {
                show: true
              },
              labelLine: {
                show: false
              }
            }
          }
        ]
      }
      this.chartInstance.setOption(initOption)
    },
    // ret 就是服务端发送给客户端的图表的数据
    getData (ret) {
      // const { data: ret } = await this.$http.get('hotproduct')
      // console.log(ret)
      this.allData = ret
      this.updateChart()
    },
    updateChart () {
      // 处理数据
      const seriesArr = this.allData[this.currentIndex].children.map((item) => {
        return {
          name: item.name,
          value: item.value,
          children: item.children
        }
      })
      const legendArr = this.allData[this.currentIndex].children.map((item) => {
        return item.name
      })
      const updateChart = {
        legend: {
          data: legendArr
        },
        series: [
          {
            data: seriesArr
          }
        ]
      }
      this.chartInstance.setOption(updateChart)
    },
    screenAdapter () {
      this.titleFontSize = (this.$refs.hot_ref.offsetWidth / 100) * 3.6
      const adapterOption = {
        title: {
          textStyle: {
            fontSize: this.titleFontSize
          }
        },
        legend: {
          itemWidth: this.titleFontSize,
          itemHeight: this.titleFontSize,
          itemGap: this.titleFontSize / 2,
          textStyle: {
            fontSize: this.titleFontSize / 2
          }
        },
        series: [
          {
            radius: this.titleFontSize * 4.5,
            center: ['50%', '60%'] // 数组的第一项是横坐标，第二项是纵坐标
          }
        ]
      }
      this.chartInstance.setOption(adapterOption)
      this.chartInstance.resize()
    },
    toLeft () {
      this.currentIndex--
      if (this.currentIndex < 0) {
        this.currentIndex = this.allData.length - 1
      }
      this.updateChart()
    },
    toRight () {
      this.currentIndex++
      if (this.currentIndex > this.allData.length - 1) {
        this.currentIndex = 0
      }
      this.updateChart()
    }
  },
  watch: {
    theme () {
      this.chartInstance.dispose() // 销毁当前的图表
      this.initChart() // 重新以最新的主题名称初始化图表对象
      this.screenAdapter() // 完成屏幕的适配
      this.updateChart() // 更新图表的展示
    }
  }
}
</script>

<style lang="less" scoped>
.com-container {
  .icon-arrow {
    position: absolute;
    top: 50%;
    color: #fff;
    cursor: pointer;
  }
  .icon-left {
    left: 5%;
  }
  .icon-right {
    right: 5%;
  }
  .my-title {
    position: absolute;
    right: 3%;
    bottom: 3%;
    color: #fff;
  }
}
</style>
