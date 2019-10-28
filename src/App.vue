<template>
  <div id="app">
    <div class="title">
      <div class="logo-bg-wrap">
        <div class="logo-bg"></div>
      </div>
      <span class="title-name">调用摄像头录像</span>
    </div>
    <div class="media-wrap">
      <div class="media-right">
        <div class="key-point-wrap">
          <div class="key-title">求救词</div>
          <ul class="key-list-wrap">
            <li class="key-list" v-for="(item, index) in keyWords" :key="index">
              <span class="key-span" :class="{'key-span-active': (item === key_word)}">{{item}}</span>
            </li>
          </ul>
        </div>
        <div class="demo-show-wrap">
          <span class="demo-show" :class="{'demo-show-active': !isStart}" @click.stop.prevent="showDemo">案例演示</span>
        </div>
      </div>
      <div class="media-left">
        <div class="video-wrap" ref="videoWrap">
          <!--录像的video-->
          <video :style="{display: !demoShow ? 'block' : 'none'}" class="video" :width="videoWidth"
                 :height="videoHeight" ref="video" preload="auto">
            <source :src="videoBlobSrc" type="video/mp4">
          </video>
          <!--播放视频的video-->
          <video :style="{display: demoShow ? 'block' : 'none'}" class="video" :width="videoWidth" :height="videoHeight"
                 ref="demoVideo" preload="auto" autoplay="autoplay">
          </video>
          <div class="start-btn-wrap">
            <img src="./images/start.svg" alt="" class="start-btn" v-show="!isStart" @click.stop.prevent="handleStart">
            <span class="countdown" v-show="!(countdown === 0)">{{this.countdown}} S后开始</span>
          </div>
          <div class="progress-bar-wrap">
            <span class="progress-time">{{playTime}}s</span>
            <span class="progress-div">
              <span class="progress-div-inner" :class="{'progress-div-inner-active' : playTime !== 0}"></span>
            </span>
          </div>
        </div>
        <div class="image-wrap">
          <div class="show-key-word" v-show="key_word">用户发出求救信号：<span class="key_word-color">{{this.key_word}}</span></div>
          <ul class="image-list" v-if="imagesList.length > 0">
            <li v-for="(imgItem, index) in imagesList" :key="index" class="image-list-inner">
              <div class="image"
                   :style="{backgroundImage: `url(${baseUrl}pictures?picture_dir=${imgItem.path})`}"></div>
            </li>
          </ul>
          <div v-else class="message-show">{{msg}}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  import axios from 'axios'
  const localUrl = 'http://localhost:8080/'
  const baseUrl = process.env.NODE_ENV === 'development' ? urlapp : localUrl
  let pageTimer = null
  export default {
    name: 'App',
    data() {
      return {
        videoWidth: '',
        videoHeight: '',
        isStart: false, // 是否开始录制
        countdown: 0, // 倒计时开始
        mediaStream: '', // 视频流文件
        video: '', // 录像视频，
        demoVideo: '', // 播放案例演示视频
        videoBlobSrc: '', // 视频路径
        keyWords: ['救命啊', '救救我', '报警啊', '110',],
        key_word: '', // 得到的唇语
        imagesList: [],
        baseUrl: baseUrl,
        demoShow: false,
        playTime: 0, // 视频播放的时间
        msg: '请点击录像或是点击案例演示' // 用户提示信息
      }
    },
    mounted() {
      this.video = this.$refs.video
      this.demoVideo = this.$refs.demoVideo
      this.videoHeight = this.$refs.videoWrap.offsetHeight
      this.videoWidth = this.$refs.videoWrap.offsetWidth
      //  视频是否已经播放完成
      this.demoVideo.onended = () => {
        this.isStart = false
        this.msg = '正在努力解析中......'
        axios.post(`${baseUrl}demo`)
          .then(async res => {
            const result = res.data
            if (result && result.code === 0) {
              const images = result.picture_dir
              await images.forEach((item, index) => {
                setTimeout(() => {
                  this.imagesList.push(item)
                }, 150 * index)
              })
              this.key_word = {
                jiujiuwo: '救救我',
                jiuminga: '救命啊',
                baojinga: '报警啊',
                '110': '110'
              }[result.key_word]
            } else {
              this.msg = '网络异常，请重新点击案例演示'
            }
          })
      }
      //  监听页面的变化
      window.onresize = () => {
        if (pageTimer) {
          clearTimeout(pageTimer)
        }
        pageTimer = setTimeout(() => {
          this.videoHeight = this.$refs.videoWrap.offsetHeight
          this.videoWidth = this.$refs.videoWrap.offsetWidth
        }, 300)
      }
    },
    methods: {
      // 点击开始录制屏幕按钮
      handleStart() {
        this.demoShow = false
        this.imagesList = []
        this.playTime = 0
        this.key_word = ''
        //  点击开始，调用摄像头
        this.getCamera()
          .then(() => {
            let timer = ''
            this.countdown = 3
            timer = setInterval(() => {
              this.countdown--
              if (this.countdown === 0) {
                clearInterval(timer)
                // 录制视频
                this.recording()
              }
            }, 1000)
          })
      },
      // 调用摄像头
      getCamera() {
        return new Promise((resolve) => {
          //  启动客户端的摄像头功能
          if (!navigator.mediaDevices.getUserMedia) {
            this.msg = '本浏览器暂不支持摄像功能，请移至Chrome浏览器'
            this.mediaStream.getTracks()[0].stop()
            return
          }
          navigator.mediaDevices.getUserMedia({
            audio: false,
            video: true
          })
            .then((mediaStream) => {
              this.isStart = true
              this.mediaStream = mediaStream
              this.video.srcObject = mediaStream
              this.video.onloadedmetadata = () => {
                this.video.play()
                this.msg = '请准备好，即将开始录像'
              }
              resolve()
            })
            .catch((err) => {
              console.log(err.name + ": " + err.message)
            })
        })
      },
      //  录制视频
      async recording() {
        //  创建一个媒体录制流文件
        if (!window.MediaRecorder) {
          this.msg = '本浏览器暂不支持录像功能，请移至Chrome浏览器'
          this.mediaStream.getTracks()[0].stop()
          return
        }
        this.threeTime()
        const mediaRecorder = await new MediaRecorder(this.mediaStream, {
          mimeType: 'video/webm',
        })
        await mediaRecorder.start()
        this.msg = '录像中'
        setTimeout(async () => {
          await this.video.pause()
          await mediaRecorder.stop()
          await this.mediaStream.getTracks()[0].stop() // 关闭摄像头
          this.isStart = false
        }, 3000)
        // 当停止录像时，获取到的录制媒体资源
        mediaRecorder.ondataavailable = async (event) => {
          await this.videoFormat(event.data)
        }
      },
      //   视频类型转化
      videoFormat(videoData) {
        const blob = new Blob([videoData], {
          'type': 'video/mp4'
        })
        this.videoBlobSrc = window.URL.createObjectURL(blob)
        const videoFormData = new FormData()
        videoFormData.append('file', blob)
        this.demoShow = false
        this.msg = '正在努力解析中......'
        //  调用接口分析得倒报警信号
        axios.post(`${baseUrl}`, videoFormData, {
          headers: {'Content-Type': 'multipart/form-data;charset=utf-8'}
        })
          .then(async res => {
            const result = res.data
            if (result && result.code === 0) {
              const images = result.picture_dir
              await images.forEach((item, index) => {
                setTimeout(() => {
                  this.imagesList.push(item)
                }, 150 * index)
              })
              this.key_word = {
                jiujiuwo: '救救我',
                jiuminga: '救命啊',
                baojinga: '报警啊',
                '110': '110'
              }[result.key_word]
            } else {
              this.msg = '未识别到求救信号'
            }
          })
      },

      //  案例演示
      showDemo() {
        if (!this.isStart) {
          this.playTime = 0
          this.demoShow = true
          this.imagesList = []
          this.$refs.demoVideo.src = `${baseUrl}demo?demo_type=110`
          this.isStart = true
          this.msg = '案例演示视频正在播放中'
          this.key_word = ''
          this.threeTime()
        }
      },

      //  3秒钟定时器
      threeTime() {
        let timer = ''
        timer = setInterval(() => {
          this.playTime = this.playTime + 1
          if (this.playTime === 2) {
            clearInterval(timer)
          }
        }, 1000)
      }


    },
  }
</script>

<style lang='stylus' rel='stylesheet/stylus' scoped>
  @import "./index.styl";
</style>
