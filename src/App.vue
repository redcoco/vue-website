<template>
  <div id="app" @resize="isApp">
    <loader :isshow="getShowLoading" loaderbackground="rgba(0,0,0,0.3)"></loader>
    <div class="maincontent">
      <fixed-bg v-if="imageInfo.url && imageSetting" :imagepath="imageInfo.url" :maskcolor="getGlobalStyle.contentInfo.bgcolor" :maskopacity="getGlobalStyle.contentInfo.opacity" :masktype="getGlobalStyle.contentInfo.type" :maskglobainfo="getGlobalStyle" ></fixed-bg>
    </div>
    <v-content></v-content>
    <updatetips :defaultvalue="getVersionList"></updatetips>
    <audio :src="getCurrentMusic.url" ref="myAudio"></audio>
  </div>
</template>
<script>
// import axios from 'axios'
import store from 'store'
import fecth from 'utils/fecth.js'
import home from 'components/home.vue'
import fixedbg from 'components/common/fixedbg/fixedbg.vue'
import content from 'components/common/content/content.vue'
import pic from 'components/pic/pic.vue'
import loader from 'components/common/loader/loader.vue'
import updatetips from 'components/common/updatetips/updatetips.vue'
import DGlobal from 'common/js/global.js'

// 引入背景请求的api
import {getBingInfo, getMineBgByIndex} from 'common/api/background.js'
// import $ from 'jquery'

export default {
  data () {
    return {
      isgetimagebybing: store.getters.getShowBingImage
    }
  },
  name: 'app',
  components: {
    home,
    'v-content': content,
    'fixed-bg': fixedbg,
    pic,
    loader,
    updatetips
  },
  methods: {
    fetchData () {
      // 是否需要获取壁纸信息
      const isShowBingImage = store.getters.getGlobalInfo.showBingImage
      // api地址
      let getbingApi = isShowBingImage ? 'http://www.daiwei.org/vue/server/home.php?inAjax=1&do=getImageByBingJson' : 'http://www.daiwei.org/vue/server/home.php?inAjax=1&do=getHomeImage'
      // 判断本地是否有背景设置的数据信息
      const hasFixedImageBg = localStorage.getItem('fixedImageBg')
      // bing 的每日一图
      if (isShowBingImage) {
        getBingInfo(getbingApi, 0)
      } else {
        // 自定义图片  默认是我设置的图片
        if ((hasFixedImageBg === null || hasFixedImageBg === '' || JSON.parse(hasFixedImageBg).type === 'bing')) {
           let index = store.getters.getFixedImageInfo.index
           getMineBgByIndex(getbingApi, index)
        } else {
          // 背景图片地址设置本地存储
          const getFixedImageBg = JSON.parse(localStorage.getItem('fixedImageBg'))
          let index = store.getters.getFixedImageInfo.index
          fecth.post(getbingApi, {index: index}).then((res) => {
            if (!(getFixedImageBg === null || '') && res.data) {
              // 判断是否和本地数据一样  一样则用本地的数据 不一样则请求最新的数据
              if (getFixedImageBg.url === res.data.url) {
                store.dispatch({
                  type: 'set_FixedImageInfo',
                  data: getFixedImageBg
                })
              } else {
                // 否则获取最新的图片信息
                getBingInfo(getbingApi, 0)
              }
            }
          })
        }
      }
    },
    getPlace () {
      fecth.get('http://www.daiwei.org/vue/server/home.php?inAjax=1&do=getAdress').then((res) => {
        if (res.data) {
          store.dispatch({
            type: 'set_Place',
            data: res.data.data
          })
          this.getWeather(res.data.data.city)
        }
      }, (err) => {
        alert(err)
      })
    },
    // 获取天气信息
    getWeather (city) {
      fecth.post(`http://www.daiwei.org/vue/server/home.php?inAjax=1&do=getWeather`, {place: city}).then((res) => {
        try {
          store.dispatch({
            type: 'set_Weather',
            data: res.data.data.forecast
          })
        } catch (e) {
          store.dispatch({
            type: 'set_Weather',
            data: {}
          })
        }
      }, (err) => {
        alert(err)
      })
    },
    formatDate (data, fmt) {
      var o = {
        'M+': data.getMonth() + 1,                 // 月份
        'd+': data.getDate(),                    // 日
        'h+': data.getHours(),                   // 小时
        'm+': data.getMinutes(),                 // 分
        's+': data.getSeconds(),                 // 秒
        'q+': Math.floor((data.getMonth() + 3) / 3), // 季度
        'S': data.getMilliseconds()             // 毫秒
      }
      if (/(y+)/.test(fmt)) {
        fmt = fmt.replace(RegExp.$1, (data.getFullYear() + '').substr(4 - RegExp.$1.length))
      }
      for (var k in o) {
        if (new RegExp('(' + k + ')').test(fmt)) {
          fmt = fmt.replace(RegExp.$1, (RegExp.$1.length === 1) ? (o[k]) : (('00' + o[k]).substr(('' + o[k]).length)))
        }
      }
      return fmt
    },
    isApp () {
      let isTrue = false
      if (document.body.clientWidth < 768) {
        isTrue = false
      } else {
        isTrue = true
      }
      store.commit({
        type: 'setIsHigher768',
        data: isTrue
      })
      // console.log(isTrue)
    },
    shouldLoadingBg () {
      if (window.localStorage) {
        // 全局设置
        const getAllStorage = JSON.parse(localStorage.getItem('globalInfo'))
        if (!(getAllStorage === null || '')) {
          store.dispatch({
            type: 'set_GlobalInfo',
            data: getAllStorage
          })
        }
        // 背景图片地址设置本地存储
        const getFixedImageBg = JSON.parse(localStorage.getItem('fixedImageBg'))
        if (!(getFixedImageBg === null || '')) {
          store.dispatch({
            type: 'set_FixedImageInfo',
            data: getFixedImageBg
          })
        }
        return true
      } else {
        return false
      }
    },
    setAudioRef () {
      store.commit({
        type: 'setAudioEle',
        data: this.$refs.myAudio
      })
    }
  },
  watch: {
    getIsBingBg (nowval, oldval) {
      this.fetchData()
      localStorage.setItem('globalInfo', JSON.stringify(store.getters.getGlobalInfo))
    }
  },
  computed: {
    getGlobalStyle () {
      return store.getters.getGlobalInfo
    },
    imageInfo () {
      return store.getters.getFixedImageInfo
    },
    imageSetting () {
      return store.getters.getFixedBgInfo
    },
    getIsBingBg () {
      return store.getters.getGlobalInfo.showBingImage
    },
    getShowLoading () {
      return store.getters.getShowLoading
    },
    getCurrentMusic () {
      return store.getters.getCurrentAudio
    },
    getVersionList () {
      return store.getters.getVersionList
    }
  },
  mounted () {
    // 输出信息
    DGlobal.utils.setConsole('基于vue2.0的网站新版更新', 1, '未曾遗忘的青春')
    // 设置audio 的refs
    this.setAudioRef()
    // this.$nextTick(() => {
    // 是否在线加载壁纸
    this.shouldLoadingBg()
    // 获取用户地址
    this.getPlace()
    // 是否小于768
    this.isApp()
    // 加载数据
    this.fetchData()
    // 挂载 onresize事件
    window.onresize = () => {
      this.isApp()
    }
    localStorage.setItem('globalInfo', JSON.stringify(store.getters.getGlobalInfo))
    // })
  }
}
</script>

<style lang="stylus" rel="stylesheet/stylus">
@import '~common/stylus/border-1px/index.styl'
@import '/static/font-icon/style.css'
  audio
    display:none
  body,html
    margin:0
    padding:0
    box-sizing:border-box
    background:#f0f0f0f0
    user-select:none
    -webkit-tap-highlight-color: transparent
  #app 
    position: fixed
    top: 0
    left: 0
    bottom: 0
    right:0
    background:transparent
</style>
