<template>
  <div class="com-container" @dblclick="revertMap">
    <div class="com-chart" ref="map_ref"></div>
  </div>
</template>

<script>
import { mapState } from 'vuex'
import axios from 'axios'
import { getProvinceMapInfo } from '@/utils/map_utils'
export default {
  data () {
    return {
      chartInstance: null,
      allData: null,
      mapData: {} // 所获取的省份的地图矢量数据
    }
  },
  created () {
    // 在组件创建完成之后，进行回调函数的注册
    this.$socket.registerCallBack('mapData', this.getData)
  },
  mounted () {
    this.initChart()
    // this.getData()
    // 发送数据给服务器，告诉服务器，我现在需要数据
    this.$socket.send({
      action: 'getData',
      socketType: 'mapData',
      chartName: 'map',
      value: ''
    })
    window.addEventListener('resize', this.screenAdapter)
    this.screenAdapter()
  },
  destroyed () {
    window.removeEventListener('resize', this.screenAdapter)
    // 在组件销毁的时候，进行回调函数的取消
    this.$socket.unRegisterCallBack('mapData')
  },
  methods: {
    async initChart () {
      this.chartInstance = this.$echarts.init(this.$refs.map_ref, this.theme)
      // 获取中国地图的矢量数据
      // http://localhost:8080/static/map/china.json
      // 由于我们现在获取的地图矢量数据并不是位于KOA2的后台，所以咱们不能使用this.$http
      const ret = await axios.get(`${window.location.origin}/static/map/china.json`)
      console.log(ret)
      this.$echarts.registerMap('china', ret.data)// registerMap是注册地图的矢量数据
      const initOption = {
        title: {
          text: '▎商家分布',
          left: 20,
          top: 20
        },
        geo: {
          type: 'map',
          map: 'china',
          top: '5%',
          bottom: '5%',
          itemStyle: {
            areaColor: '#2E72BF', // 地图省份背景颜色
            borderColor: '#333' // 省份间隔之间线条颜色
          }
        },
        legend: {
          left: '5%',
          bottom: '5%',
          orient: 'vertical'
        }
      }
      this.chartInstance.setOption(initOption)
      this.chartInstance.on('click', async arg => {
        const provinceInfo = getProvinceMapInfo(arg.name)
        // 需要获取这个省份的地图矢量数据
        // 判断当前所点击的这个省份的地图矢量数据在mapData中是否存在,如果存在不会再次发起接口请求
        if (!this.mapData[provinceInfo.key]) {
          const provinceRes = await axios.get(`${window.location.origin}/${provinceInfo.path}`)
          this.mapData[provinceInfo.key] = provinceRes.data
          this.$echarts.registerMap(provinceInfo.key, provinceRes.data)
        }
        console.log(this.mapData)
        console.log(provinceInfo)
        const changeOption = {
          geo: {
            map: provinceInfo.key
          }
        }
        this.chartInstance.setOption(changeOption)
        // await axios.get(`${window.location.origin}/${provinceInfo.path}`).then(res => {
        //   console.log(res.data)
        // })
      })
    },
    // ret 就是服务端发送给客户端的图表的数据
    getData (ret) {
      // const { data: ret } = await this.$http.get('map')
      this.allData = ret
      // console.log(ret)
      // 获取服务器的数据，对this.allData进行赋值之后，调用updateChart方法更新图表
      this.updateChart()
    },
    updateChart () {
      // 处理图表需要的数据
      const seriesArr = this.allData.map(item => {
        // retrun的这个对象就代表的是一个类别下的所有散点数据
        // 如果想在地图中显示散点的数据，所以我们需要给散点的图表增加一个配置，coordinateSystem:'geo'
        return {
          type: 'effectScatter',
          rippleEffect: {
            scale: 5, // 涟漪效果数值
            brushType: 'stroke' // 空心的涟漪
          },
          name: item.name,
          data: item.children,
          coordinateSystem: 'geo'// 地图中显示散点的数据必须加这个
        }
      })
      // 图例的数据
      const legendArr = this.allData.map(item => {
        return item.name
      })
      const dataOption = {
        legend: {
          data: legendArr
        },
        series: seriesArr
      }
      this.chartInstance.setOption(dataOption)
    },
    screenAdapter () {
      const titleFontSize = this.$refs.map_ref.offsetWidth / 100 * 3.6
      const adapterOption = {
        title: {
          textStyle: {
            fontSize: titleFontSize
          }
        },
        legend: {
          itemWidth: titleFontSize / 2,
          itemHeight: titleFontSize / 2,
          itemGap: titleFontSize / 2,
          textStyle: {
            fontSize: titleFontSize / 2
          }
        }
      }
      this.chartInstance.setOption(adapterOption)
      this.chartInstance.resize()
    },
    // 回到中国地图
    revertMap () {
      const revertOption = {
        geo: {
          map: 'china'
        }
      }
      this.chartInstance.setOption(revertOption)
    }
  },
  computed: {
    ...mapState(['theme'])
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

<style>

</style>
