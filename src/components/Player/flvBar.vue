<template>
  <!-- 层级说明：缓冲、加载、切换中途，会伴随层级调整 -->
  <!--            视频video: z-index: 默认1 -->
  <!--            弹幕danmu: z-index: 10 -->
  <!--        播放/暂停pause: z-index: 20 -->
  <!--          加载loading: z-index: 30 -->
  <!-- 底部、快进退、中部音量: z-index: 40 -->
  <div class="pl" :style="`background: ${options.theme}`">
    <video
      id="lee-player"
      :controls="false"
      :style="`max-width: 100%; max-height:100%`"
      :poster="options.poster"
      :muted="options.muted"
      @seeking="videoSeeking"
      @seeked="videoSeeked"
    >
      {{options.lang === 'zh' ? '您的浏览器不支持当前格式，请升级或更换谷歌浏览器。' : 'Your browser does not support the current format. Please upgrade or replace Google browser.'}}
    </video>
    <!-- 外部遮罩 loading -->
    <div class="loading" v-if="loading">
      <img :src="urlStatus ? LoadingImg : ErrorImg" alt="" />
      <div class="tip">
        <i :class="urlStatus ? 'el-icon-loading' : 'el-icon-error'"></i>
        {{ urlStatusText }}
      </div>
    </div>
    <!-- 弹幕层 -->
    <Danmu :openDanmu="openDanmu" :danmuData="danmuData" :danmuOpacity="danmuOpacity" :danmuArea="danmuArea" :playStatus="playStatus" :hasPlayedTime="hasPlayedTime" />
    <!-- 键盘事件 音量提醒 -->
    <div class="volume-keycode">
      <i :class="`iconfont ${volume ? 'icon-mn_shengyin_fill' : 'icon-mn_shengyinwu_fill'}`"></i>
      <span>{{volume}}%</span>
    </div>
    <!-- 播放器中部播放 暂停 -->
    <transition name="bounce">
      <div
        class="center-btn"
        :class="playStatus ? '' : 'center-btn-check'"
        @click="debounce(playOrPause(), 100, true)"
      >
        <i
          :class="
            playStatus ? 'pbtn el-icon-video-pause' : 'el-icon-video-play'
          "
        ></i>
      </div>
    </transition>
    <!-- 键盘事件 播放提醒 -->
    <div :class="fastPlayStep ? 'playtime-keycode' : 'playtime-keycode pt-keycode-none'">
      {{`${fastPlayStep ? fastPlayStep === 'right' ? `快进 ${dragVolumeLen}秒` : `快退 ${dragVolumeLen}秒` : ''}`}}
    </div>
    <div class="pl-control">
      <!-- 进度条 -->
      <div class="process" @click="changePlayedTime($event)">
        <div class="preload" :style="`width: ${(preloadTime >= fullTime ? fullTime : preloadTime) * 100 / fullTime}%`"></div>
        <div
          class="playing"
          :style="`width: ${
            options.url ? ((hasPlayedTime >= fullTime ? fullTime : hasPlayedTime) * 100) / fullTime : 0
          }%`"
        ></div>
        <div
          class="point"
          @click="($event) => $event.stopPropagation()"
          :style="`display: ${isLive ? 'none' : 'block'}; left: calc(${
            options.url ? ((hasPlayedTime >= fullTime ? fullTime : hasPlayedTime) * 100) / fullTime : 0
          }% - 5px)`"
        ></div>
      </div>
      <!-- 操作按钮 -->
      <div class="opt-buttons">
        <!-- 左 播放暂停按钮 及时长 -->
        <div class="lf-play">
          <span class="play-pause-btn" @click="debounce(playOrPause(), 100, true)"
            ><i
              :class="
                playStatus ? `iconfont icon-zanting` : `iconfont icon-bofang1`
              "
            ></i
          ></span>
          <span>
            {{ filterTime(hasPlayedTime) }}
            {{ !isLive ? options.url ? ' / ' + filterTime(fullTime) : " / 00:00" : '' }}
            <span v-if="isLive" class="live">直播</span>
            <i v-if="isLive" class="refresh iconfont icon-shuaxin" title="刷新" @click="refreshPlayTime"></i>
          </span>
        </div>
        <div class="rt-screen">
          <!-- 右 打赏 -->
          <div class="part" v-if="options.reward">
            打赏
            <div class="part-cover reward">
              <div style="display: flex; flex-direction: column; justify-content: center; align-items: center; background: #333; font-size: 10px; padding: 20px 10px">
                <i class="iconfont icon-jifen1"></i>
                积分
              </div>
            </div>
          </div>
          <!-- 右 倍速 -->
          <div style="width: 28px" class="part">
            {{ speedType == 1 ? "倍速" : speedType + "x" }}
            <div class="part-cover speed">
              <span
                v-for="i in options.speedOptions"
                :key="i"
                :class="speedType === i ? 'play-span' : ''"
                @click="changeSpeed($event, i)"
                >{{ i + "x" }}</span
              >
            </div>
          </div>
          <!-- 右 清晰度 -->
          <div class="part">
            {{ renderQuality(qualityType) }}
            <div class="part-cover quality">
              <span
                v-for="i in [
                  { v: 'lossless', n: '原画', t: '4K' },
                  { v: 'superhigh', n: '超清', t: '1080P' },
                  { v: 'high', n: '高清', t: '720P' },
                  { v: 'auto', n: '自动', t: '' },
                ]"
                :key="i.v"
                :class="qualityType === i.v ? 'play-span' : ''"
                @click="changeQuality($event, i)"
                >{{ i.n }}{{ i.t }}</span
              >
            </div>
          </div>
          <!-- 右 音量 -->
          <div class="part">
            <i @click="toggleVolumeStatus" :class="`iconfont ${openVolume ? 'icon-mn_shengyin_fill' : 'icon-mn_shengyinwu_fill'}`"></i>
            <div class="part-cover volume">
              <span class="num">{{openVolume ? parseInt(volume) : 0}}</span>
              <div class="volume-line" @click="changeVolume($event)">
                <div
                  class="volume-straight"
                  :style="`height: ${openVolume ? volume : 0}px`"
                ></div>
                <div
                @click="($event) => $event.stopPropagation()"
                  class="volume-check"
                  :style="`bottom: ${(openVolume ? volume : 0) - 5}px`"
                ></div>
              </div>
            </div>
          </div>
          <!-- 右 网页全屏 -->
          <div class="part" @click="cssScreenFunc" v-if="options.cssFullScreen">
            <el-tooltip class="pitem" effect="dark" :content="`${cssScreen ? '退出' : '进入'}网页全屏`" placement="top">
              <i :class="cssScreen
                ? `iconfont icon-24gf-fullScreenExit3`
                : `iconfont icon-24gf-fullScreenEnter3`
              "
              ></i>
            </el-tooltip>

          </div>
          <!-- 右 设备全屏 -->
          <div class="part" @click="fullScreenFunc" v-if="options.fullScreen">
            <el-tooltip class="pitem" effect="dark" :content="`${fullScreen ? '退出' : '进入'}全屏`" placement="top">
              <i
                class="full"
                :class="
                  fullScreen
                    ? `iconfont icon-cancel-full-screen`
                    : `iconfont icon-quanping1`
                "
              ></i>
            </el-tooltip>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import LoadingImg from "@/assets/loading.png";
import ErrorImg from "@/assets/error.png";
import Hls from 'hls.js'
import Danmu from '../Danmu/index.vue'
export default {
  name: "lee-player-flv",
  props: {
    options: {
      type: Object,
      default: () => {}
    }
  },
  components: {
    Danmu
  },
  data() {
    return {
      LoadingImg: LoadingImg,
      ErrorImg: ErrorImg,
      loading: true, // 数据加载阶段
      urlStatus: true, // 播放链接是否有效
      urlStatusText: '视频加载中', // 链接状态说明
      vplayer: null, // 播放器
      playStatus: true, // 播放状态，播放/暂停
      showPlayBtn: false, // 视频中部播放/暂停按钮
      cssScreen: false, // 网页全屏
      fullScreen: false, // 显示器全屏
      isLive: this.options.isLive, // 是否直播
      fullTime: 0, // 视频总时长
      hasPlayedTime: 0, // 已播放时长（秒）
      preloadTime: 0, // 缓冲区
      hasPlayedPoint: 0, // 直播条件下 记录最远播放值
      countTime: null, // 计时器
      speedType: this.options.speed, // 当前倍速
      qualityType: this.options.quality, // 当前清晰度
      volume: this.options.volume, // 当前音量
      dragVolumeLen: 0, // 拖动长度
      openVolume: !this.options.muted, // 开启音量
      interval: null, // 底部栏 显隐定时器
      fastPlayStep: null, // 快进或快退
      keyVolInterval: null, // 键盘事件 音量处理定时器
      danmuData: [],
      openDanmu: false,
      danmuArea: 'all',
      danmuOpacity: 1
    };
  },
  watch: {
    'options.url'(n, o) {
      if (n !== o && n) {
        this.qualityType = this.options.quality;
        this.isLive = this.options.isLive;
        this.speedType = this.options.speed;
        this.volume = this.options.volume;
        this.openVolume = !this.options.muted;
        this.danmuData = this.options.danmuList;
        this.openDanmu = this.options.danmu;
        this.danmuArea = this.options.danmuSettings.position;
        this.danmuOpacity = this.options.danmuSettings.opacity;
        this.createPlayer();
      }
    },
    'options.danmu'(n, o) {
      if (n !== o) {
        this.openDanmu = n;
      }
    },
    'options.danmuSettings.position'(n, o){
      if (n !== o) {
        this.danmuArea = n;
      }
    },
    'options.danmuSettings.opacity'(n, o){
      if (n !== o) {
        this.danmuOpacity = n;
      }
    }
  },
  mounted() {
    this.createPlayer();
  },
  beforeDestroy() {
    this.resetPlayer();
  },
  methods: {
    // 重置播放器
    resetPlayer(){
      this.vplayer && this.vplayer.destroy();
      this.vplayer && this.vplayer.detachMedia();
      this.vplayer = null;
      const dqPlayer = document.getElementById("lee-player");
      if(dqPlayer){
        dqPlayer.style.height = this.height + "px";
        dqPlayer.src = "";
        dqPlayer.innerHTML = "";
      }
      this.loading = true;
      this.urlStatus = false;
      this.playStatus = false;
      this.showPlayBtn = false;
      this.hasPlayedTime = 0;
      this.hasPlayedPoint = 0;
      this.dragVolumeLen = 0;
      this.fastPlayStep = null;
      this.fullTime = 0;
      this.preloadTime = 0;
      this.countTime = null;
    },
    // 播放或暂停
    playOrPause() {
      if (this.hasPlayedTime === this.fullTime) {
        this.hasPlayedTime = 0;
        this.preloadTime = 0;
        this.vplayer ? this.vplayer._media.currentTime = 0 : null;
        this.createPlayer();
        return;
      }
      this.showPlayBtn = true;
      this.playStatus = !this.playStatus;
      this.playStatus && this.vplayer
        ? this.vplayer._media.play()
        : this.vplayer._media.pause();
      this.computePlayedTime();
      // 控制 播放\暂停 按钮显隐
      let timm = setTimeout(() => {
        this.showPlayBtn = !this.playStatus;
        clearTimeout(timm);
      }, 1500);
    },
    // 时长显示方式
    filterTime(t) {
      const hou = t ? parseInt(t / 3600) : 0;
      const min = t ? parseInt((t - (hou * 3600)) / 60) : 0;
      const sec = t ? parseInt(t % 60) : 0;
      let str = t ? `${min < 10 ? "0" + min : min}:${sec < 10 ? "0" + sec : sec}` : "00:00";
      str = hou ? `${hou < 10 ? "0" + hou : hou}:` + str : str;
      return str;
    },
    // 播放时长计算
    computePlayedTime() {
      this.countTime = setTimeout(() => {
        if (this.hasPlayedTime >= this.fullTime) {
          if(this.isLive){
            this.fullTime = this.fullTime + this.vplayer.bufferController.details.totalduration;
            clearTimeout(this.countTime);
            this.countTime = null;
            this.computePlayedTime();
          } else {
            this.hasPlayedTime = this.fullTime;
            this.playStatus = false;
            clearTimeout(this.countTime);
            this.countTime = null;
          }
        } else {
          if(this.hasPlayedTime >= this.preloadTime){
            this.loading = true;
            this.urlStatusText = '视频加载中';
            this.playStatus = false;
          }
          if (this.playStatus) {
            this.hasPlayedTime += this.speedType;
            this.hasPlayedTime = this.hasPlayedTime >= this.fullTime ? this.fullTime : this.hasPlayedTime;
          }
          clearTimeout(this.countTime);
          this.countTime = null;
          this.computePlayedTime();
        }
        this.hasPlayedPoint = this.hasPlayedTime > this.hasPlayedPoint ? this.hasPlayedTime : this.hasPlayedPoint;
      }, 1000);
    },
    // 点击指定播放位置
    changePlayedTime(ev) {
      if(this.isLive){
        return;
      }
      const process = document.querySelector(".process");
      const e = ev || window.event;
      if (process) {
        const w = process.offsetWidth;
        const multiple = e.offsetX / w; // 选中点与总时长比例
        this.hasPlayedTime = multiple * (this.options.url ? this.fullTime : 0);
        this.vplayer ? this.vplayer._media.currentTime = this.hasPlayedTime : null; // 原生dom指定播放位置
      }
    },
    // 点击 更改音量
    changeVolume(ev) {
      const volume = document.querySelector(".volume-line");
      const e = ev || window.event;
      if (volume) {
        this.openVolume = true;
        const h = volume.offsetHeight;
        const multiple = (100 - e.layerY) / h; // 选中点与总音量比例
        this.volume = multiple * 100;
        this.vplayer ? this.vplayer._media.volume = multiple : null; // 原生dom指定音量
      }
    },
    // 静音/非静音
    toggleVolumeStatus(){
      this.openVolume = !this.openVolume;
      this.volume = this.volume || 75;
      this.vplayer ? this.vplayer._media.volume = this.openVolume ? this.volume / 100 : 0 : null; // 原生dom指定音量
    },
    // 开始触发播放位置
    videoSeeking() {
      console.log("video Seeking");
    },
    // 开始触发播放位置
    videoSeeked() {
      console.log("video Seeked");
    },
    // 初始化 M3U8 播放器
    createPlayer() {
      this.resetPlayer();
      setTimeout(() => {
        this.initPlayer();
      }, 222);
    },
    // 初始化 M3U8 播放器
    initPlayer() {
      const video = document.getElementById("lee-player");
      const l = this.options.url.filter((k) => {
        return k.quality === this.qualityType;
      });
      const source = this.qualityType ? l[0].url : '';
      if (!source) {
        this.createPlayer();
        return;
      }
      this.urlStatusText = "视频加载中";
      this.urlStatus = true;
      if (Hls.isSupported()) {
        this.vplayer = new Hls();
        this.vplayer.attachMedia(video);
        // 完成 video 节点绑定
        this.vplayer.on(Hls.Events.MEDIA_ATTACHED, (e, data) => {
          // console.log("完成 video 节点绑定", Hls.Events, e, data);
          this.vplayer.loadSource(source);
          // [m3u8链接、*.ts链接]成功
          this.vplayer.on(Hls.Events.MANIFEST_LOADED, (e, mdata) => {
            // console.log("[m3u8链接、*.ts链接]成功", mdata);
            this.fullTime = mdata.levels[0].details.totalduration; // 初始化视频总时长
            this.isLive = mdata.levels[0].details.live; // 初始化播放类型：[点播 false, 直播 true]
            // 直播，获取缓冲区
            this.isLive && this.vplayer.on(Hls.Events.LEVEL_LOADED, (e , ldata) => {
              // console.log("直播，获取缓冲区", ldata);
              this.preloadTime = ldata.details.fragments[0].appendedPTS;
            })
            // 点播，获取缓冲区
            !this.isLive && this.vplayer.on(Hls.Events.FRAG_BUFFERED, (e , fdata) => {
              // console.log("点播，获取缓冲区", fdata);
              this.preloadTime = fdata.frag.appendedPTS;
              // console.log('this.preloadTime::this.hasPlayedTime::', this.preloadTime, this.hasPlayedTime)
              if(this.preloadTime >= this.hasPlayedTime){
                this.startPlay();
              }
              return;
            })
            this.startPlay();
          })
          // [m3u8链接、*.ts链接]失败
          this.vplayer.on(Hls.Events.ERROR, (e, data) => {
            if (data.fatal) {
              switch (data.response.code) {
                case 404:
                  this.urlStatusText = "视频不存在";
                  break;
                case 0:
                  this.urlStatusText = "视频加载异常";
                  break;
                default:
                  this.urlStatusText = "视频加载异常";
                  break;
              }
            } else {
              if(data && data.response){
                switch (data.response.code) {
                  case 403:
                    this.urlStatusText = "无权限访问该视频";
                    break;
                  default:
                    break;
                }
              } else {
                this.urlStatusText = "视频加载异常";
              }
            }
            this.resetPlayer();
          });
        })
      } else if (video.canPlayType("application/vnd.apple.mpegurl")) {
        video.src = source;
        this.loading = false;
      }
      this.listenKeyEvent();
    },
    // start play 预加载完毕 开始播放
    startPlay(){
      let timm = setTimeout(() => {
        if(this.vplayer && (this.preloadTime > this.hasPlayedTime || !this.isLive)){
          this.vplayer._media.playbackRate = this.speedType;  // 初始化播放倍速
          !this.playStatus && this.fadeOperationBar(5);  // 底部栏 隐藏倒计时5秒
          this.loading = false;  // 链接有效 隐藏loading
          if(this.vplayer._media.paused){
            this.playStatus = false;
          } else {
            this.computePlayedTime();    // 计算已播放时间
            this.playStatus = true;  // 开启 可播放状态
            this.vplayer._media.play();  // 触发原生播放事件
          }
          clearTimeout(timm);
        } else {
          this.startPlay();
        }
      }, 333);
    },
    // 刷新当前播放资源
    refreshPlayTime(){
      this.createPlayer();
    },
    // 网页全屏
    cssScreenFunc() {
      this.fullScreen = false;
      const dE = document.querySelector('.pl');
      this.closeFullscreen(dE);
      // 若已开启 则关闭↓；已关闭，则开启↓
      //W3C、FireFox、Chrome等、IE11
      if(dE.requestFullscreen || dE.mozRequestFullScreen || dE.webkitRequestFullScreen || dE.mscssScreenRequestFullscreen){
        this.cssScreen ? this.closeFullscreen(dE, 'page') : this.renderCssFull(dE);
      }
    },
    // 渲染网页全屏
    renderCssFull(el){
      el.style.position = 'fixed';
      el.style.left = 0;
      el.style.top = 0;
      el.style.zIndex = 999;
      el.style.width = '100vw';
      el.style.height = '100vh';
      this.cssScreen = true;
    },
    // 退出全屏（样式全屏、显示器全屏）
    closeFullscreen(el, type = null) {
      if(type === 'page'){
        el.style.position = 'relative';
        el.style.left = 'auto';
        el.style.top = 'auto';
        el.style.zIndex = 1;
        el.style.width = this.options.width + 'px';
        el.style.height = this.options.height ? this.options.height + 'px' : this.options.width * 550 / 818 + 'px';
        this.cssScreen = false;
        return;
      }
      if (document && document.exitFullscreen) {
         document.exitFullscreen();
       } else if (document && document.msExitFullscreen) {
         document.msExitFullscreen();
       } else if (document && document.mozCancelFullScreen) {
         document.mozCancelFullScreen();
       } else if(document && document.oRequestFullscreen){
            document.oCancelFullScreen();
        }else if (document && document.webkitExitFullscreen){
         document.webkitExitFullscreen();
       }else{
        const docHtml = document.documentElement;
        const docBody = document.body;
        docHtml.style.cssText = "";
        docBody.style.cssText = "";
        el.style.cssText = "";
        document.IsFullScreen = false;
      }
    },
    // 显示器全屏
    fullScreenFunc() {
      this.cssScreen = false;
      const dE = document.querySelector('.pl');
      this.closeFullscreen(dE, 'page')
      //W3C
      if(dE.requestFullscreen){
        this.fullScreen ? this.closeFullscreen(dE) : dE.requestFullscreen();
        this.fullScreen = !this.fullScreen;
      }
      //FireFox
      else if (dE.mozRequestFullScreen) {
        this.fullScreen ? this.closeFullscreen(dE) : dE.mozRequestFullScreen();
        this.fullScreen = !this.fullScreen;
      }
      //Chrome等
      else if (dE.webkitRequestFullScreen) {
        this.fullScreen ? this.closeFullscreen(dE) : dE.webkitRequestFullScreen();
        this.fullScreen = !this.fullScreen;
      }
      //IE11
      else if (dE.msRequestFullscreen) {
        this.fullScreen ? this.closeFullscreen(dE) : dE.msRequestFullscreen();
        this.fullScreen = !this.fullScreen;
      }
    },
    // 底部栏展示当前清晰度
    renderQuality(q) {
      switch (q) {
        case "lossless":
          return "原画";
        case "superhigh":
          return "超清";
        case "high":
          return "高清";
        case "auto":
          return "自动";
        default:
          return "自动";
      }
    },
    // 切换倍速
    changeSpeed(e, s) {
      e.stopPropagation();
      this.speedType = s;
      this.vplayer ? this.vplayer._media.playbackRate = this.speedType : null;
    },
    // 切换清晰度
    changeQuality(e, q) {
      e.stopPropagation();
      this.qualityType = q.v;
      this.createPlayer();
    },
    // 底部操作栏 淡入淡出
    fadeOperationBar(ht){
      this.interval ? clearInterval(this.interval) : '';
      this.interval = null;
      const plControl = document.querySelector('.pl-control');
      plControl.className = plControl.className.includes('show-control') ? plControl.className : plControl.className + ' show-control';
      let hideTime = ht;
      this.interval = setInterval(() => {
        hideTime--;
        this.fastPlayStep = hideTime < 3 ? null : this.fastPlayStep;
        if(hideTime <= 0){
          plControl.className = plControl.className.includes('show-control') ? plControl.className.split(' show-control')[0] : plControl.className;
          clearInterval(this.interval);
          this.interval = null;
          this.fastPlayStep = null;
        }
      }, 1000);
    },
    // 键盘音量 淡入淡出
    keyCodeVol(){
      let tm = 3;
      this.keyVolInterval && clearInterval(this.keyVolInterval);
      this.keyVolInterval = null;
      const vk = document.querySelector('.volume-keycode');
      vk.className = vk.className.includes('volume-keycode-show') ? vk.className : vk.className + ' volume-keycode-show';
      this.keyVolInterval = setInterval(() => {
        tm--;
        if(tm <= 0){
          vk.className = vk.className.includes('volume-keycode-show') ? vk.className.split(' volume-keycode-show')[0] : vk.className;
          clearInterval(this.keyVolInterval);
        }
      }, 1000)
    },
    // 监听键盘事件
    listenKeyEvent(){
      const _this = this;
      _this.$nextTick(() => {
        document.onkeydown = function (e){
          e.stopPropagation();
          // 网页全屏模式下 监听esc
          if (e.keyCode === 27 || e.key === 'Escape') {
            const dE = document.querySelector('.pl');
            _this.closeFullscreen(dE, 'page');
            _this.fullScreen = false;
          }
          // 监听空格键
          if (e.keyCode === 32 || e.key === ' ') {
            // 禁止默认页面移动
            if(e.preventDefault){
                e.preventDefault();
                _this.debounce(_this.playOrPause(), 100, false);
            }else{
                window.event.returnValue = false;
            }
          }
          // 监听方向键 上箭头 38
          if (e.keyCode === 38 || e.key === 'ArrowUp') {
            const vol = _this.volume + 10;
            _this.volume = vol > 100 ? 100 : vol;
            _this.vplayer ? _this.vplayer._media.volume = _this.volume / 100 : null;
            e.stopPropagation()
            _this.keyCodeVol();
            return false;
          }
          // 监听方向键 下箭头 40
          if (e.keyCode === 40 || e.key === 'ArrowDown') {
            const vol = _this.volume - 10;
            _this.volume = vol < 0 ? 0 : vol;
            _this.vplayer ? _this.vplayer._media.volume = _this.volume / 100 : null;
            _this.keyCodeVol();
            return false;
          }
          // 监听方向键 左箭头 37
          if (e.keyCode === 37 || e.key === 'ArrowLeft') {
            _this.dragVolumeLen = 5;
            const ct = _this.hasPlayedTime - _this.dragVolumeLen;
            _this.hasPlayedTime = ct < 0 ? 0 : ct;
            _this.vplayer ? _this.vplayer._media.currentTime = _this.hasPlayedTime : null;
            if(_this.hasPlayedTime === 0){
              return false;
            }
            _this.fastPlayStep = 'left';
            _this.debounce(_this.fadeOperationBar(5), 1000, true);
            return false;
          }
          // 监听方向键 右箭头 39
          if (e.keyCode === 39 || e.key === 'ArrowRight') {
            if(_this.hasPlayedTime >= _this.hasPlayedPoint && _this.isLive){
              return false;
            }
            _this.dragVolumeLen = 5;
            const ct = _this.hasPlayedTime + _this.dragVolumeLen;
            _this.hasPlayedTime = ct  > _this.fullTime ? _this.fullTime : ct;
            _this.vplayer ? _this.vplayer._media.currentTime = _this.hasPlayedTime : null;
            if(_this.hasPlayedTime === _this.fullTime){
              return false;
            }
            _this.fastPlayStep = 'right';
            _this.debounce(_this.fadeOperationBar(5), 1000, true);
            return false;
          }
        }
        // 显示器全屏模式下 无法监听esc按钮事件
        window.onresize = () => {
          const isFull = document.fullscreenElement || document.mozFullScreenElement || document.webkitFullscreenElement;
          if(!isFull){
            _this.fullScreen = false;
          }
        }
        // 播放进度条拖动监听
        let pointleft = 0, dragVolumeLen = 0;
        let point = document.querySelector('.point');
        point.onmousedown = function(event){
          const process = document.querySelector('.process');
          point = document.querySelector('.point');
          const processW = process.offsetWidth;
          const eve = event || window.event;
          const leftVal = eve.clientX - this.offsetLeft;
          dragVolumeLen = _this.hasPlayedTime;
          document.onmousemove = function(ev){
            this.loading = false;
            const e = ev || window.event;
            pointleft = e.clientX - leftVal;
            if(pointleft < 0){
              pointleft = 0;
            } else if(pointleft > scroll.offsetWidth - point.offsetWidth){
              pointleft = scroll.offsetWidth - point.offsetWidth;
            }
            const nowTime = (pointleft / processW) * _this.fullTime;
            _this.fastPlayStep = _this.hasPlayedTime > nowTime ? 'left' : _this.hasPlayedTime < nowTime ? 'right' : null;
            _this.hasPlayedTime = nowTime;
            _this.vplayer ? _this.vplayer._media.currentTime = _this.hasPlayedTime : null;
            _this.dragVolumeLen = parseInt(Math.abs(dragVolumeLen - _this.hasPlayedTime));
            //防止拖动鼠标过快，弹起鼠标，point受影响移动
            window.getSelection ? window.getSelection().removeAllRanges() : document.selection.empty();
          }
        }
        // 音量进度条拖动监听
        const volumeLine = document.querySelector('.volume-line');
        const volumeCheck = document.querySelector('.volume-check');
        let volumeCheckTop = 0;
        const volumeLineH = volumeLine.offsetHeight;
        volumeCheck.onmousedown = function(event){
          const eve = event || window.event;
          const topVal = eve.clientY - this.offsetTop;
          document.onmousemove = function(ev){
            const e = ev || window.event;
            volumeCheckTop = e.clientY - topVal;
            if(volumeCheckTop < 0){
              volumeCheckTop = 0;
            } else if(volumeCheckTop > 100){
              volumeCheckTop = 100;
            }else if(volumeCheckTop > scroll.offsetHeight - volumeCheck.offsetHeight){
              volumeCheckTop = scroll.offsetHeight - volumeCheck.offsetHeight;
            }
            const vol = (100 - volumeCheckTop) / volumeLineH;
            _this.openVolume = !!vol;
            _this.volume = vol * 100;
            _this.vplayer ? _this.vplayer._media.volume = vol : null;
            //防止拖动鼠标过快，弹起鼠标，volumeCheck受影响移动
            window.getSelection ? window.getSelection().removeAllRanges() : document.selection.empty();
          }
        }
        document.onmouseup = function(){
          document.onmousemove = null;
        }
        // 鼠标进入 监听下方操作栏淡入淡出
        const pl = document.querySelector('.pl');
        pl.onmouseenter = () => {
          _this.debounce(_this.fadeOperationBar(5), 1000, true);
          pl.onmousemove = () => {
            _this.debounce(_this.fadeOperationBar(5), 1000, true);
          }
        }
        pl.onmouseleave = () =>{
          _this.debounce(_this.fadeOperationBar(1), 1000, true);
        }
      })
    },
    // 函数防抖处理 (事件, 防抖间隔, immediate[true 立即执行，false 延迟执行])
    debounce(func, wait, immediate) {
      let timer;
      return function() {
        let context = this, args = arguments;
        if (timer) clearTimeout(timer);
        if (immediate) {
          let callNow = !timer;
          timer = setTimeout(() => {
            timer = null;
          }, wait);
          if (callNow) func.apply(context, args);
        } else {
          timer = setTimeout(() => {
            func.apply(context, args);
          }, wait)
        }
      }
    }
  },
};
</script>

<style lang="scss" scoped>
.pl {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  margin: 0 auto;
  width: 100%;
  height: 100%;
  .loading {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 30;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    background: #000;
    img {
      width: 120px;
    }
    .tip {
      margin-top: 15px;
      font-size: 12px;
      letter-spacing: 1px;
      color: #00d5d5;
      display: flex;
      justify-content: center;
      align-items: center;
      i {
        font-size: 14px;
        margin-right: 5px;
      }
    }
  }

  /* volume-keycode */
  .volume-keycode{
    position: absolute;
    top:65px;
    left: 50%;
    width: 120px;
    height: 40px;
    margin-left: -60px;
    z-index: 40;
    background: rgba(0, 0, 0, .1);
    font-size: 20px;
    display: flex;
    justify-content: center;
    align-items: center;
    color: rgba(255, 255, 255, .35);
    border-radius: 4px;
    transition: all .3s;
    opacity: 0;
    i{
      font-size: 20px;
      margin-right: 12px;
    }
  }
  .volume-keycode-show{
    opacity: 1!important;
  }

  /* center-btn */
  .center-btn {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    transition: all 0.5s;
    z-index: 20;
    i {
      font-size: 75px;
      color: #fff;
      transition: all 0.5s;
      cursor: pointer;
      opacity: 1;
      margin-bottom: 20px;
    }
    .pbtn {
      opacity: 0;
      transform: scale(1.2);
    }
  }
  .center-btn-check {
    background: rgba(0, 0, 0, 0.2);
  }
  /* playtime-keycode */
  .playtime-keycode{
    width: auto;
    position: absolute;
    left: 8px;
    bottom: 55px;
    padding: 2px 10px;
    border-radius: 3px;
    z-index: 40;
    color: #fff;
    transition: all .5s;
    font-size: 11px;
    opacity: 1;
    background: rgba(255, 255, 255, .2);
  }
  .pt-keycode-none{
     opacity: 0!important;
  }
  /* pl-control */
  .pl-control {
    width: 100%;
    height: 50px;
    position: absolute;
    left: 0;
    bottom: -47px;
    background: rgba(0, 0, 0, 0.8);
    z-index: 40;
    transition: all .5s;
    .process {
      width: calc(100% - 14px);
      height: 4px;
      margin: 0 auto;
      background: #333;
      position: relative;
      cursor: pointer;
      transition: all 0.3s;
      &:hover {
        .point {
          transform: scale(1.1);
          opacity: 1;
        }
      }
      .preload{
        position: absolute;
        top: 0;
        left: 0;
        // width: 50%;
        height: 100%;
        background: #717171;
        z-index: 6;
      }
      .playing {
        position: absolute;
        top: 0;
        left: 0;
        width: 0%;
        height: 100%;
        background: linear-gradient(
          to right bottom,
          #10c8af 0%,
          #10c8af 60%,
          #00d5d5 100%
        );
        z-index: 6;
      }
      .point {
        position: absolute;
        top: -3px;
        left: -5px;
        width: 10px;
        height: 10px;
        border-radius: 100%;
        background: #fff;
        z-index: 7;
        opacity: 0;
        &:hover {
          transform: scale(1.1);
          opacity: 1;
        }
      }
    }
    .opt-buttons {
      width: calc(100% - 24px);
      height: calc(100% - 4px);
      margin: 0 auto;
      display: flex;
      justify-content: space-between;
      align-items: center;
      .lf-play {
        display: flex;
        justify-content: flex-start;
        align-items: center;
        span,
        i {
          color: #fff;
          height: 17px;
          font-size: 12px;
          font-family: PingFangSC-Regular, PingFang SC;
          font-weight: 400;
          line-height: 17px;
        }
        .play-pause-btn {
          margin-right: 15px;
          cursor: pointer;
          i {
            font-size: 20px;
          }
        }
        .live{
          margin-left: 10px;
          border: 1px solid #10c8af;
          color: #10c8af;
          font-size: 9px;
          padding: 0 6px;
          border-radius: 2px;
        }
        .refresh{
          display: inline-block;
          margin-left: 10px;
          transition: all 0.7s;
          cursor: pointer;
          &:hover{
            color: #00d5d5;
            transform: rotate(360deg);
          }
        }
      }
      .rt-screen {
        display: flex;
        justify-content: flex-end;
        align-items: center;
        height: 100%;
        .part,
        i {
          color: #fff;
          font-family: PingFangSC-Regular, PingFang SC;
          font-weight: 400;
        }
        .part {
          margin-left: 20px;
          font-size: 14px;
          height: 100%;
          display: flex;
          align-items: center;
          line-height: 100%;
          cursor: pointer;
          transition: all 0.1s;
          position: relative;
          &:hover {
            color: #10c8af;
            .part-cover {
              opacity: 1;
              bottom: 50px;
              background: #1d1b19;
              z-index: 2;
              transform: scale(1);
            }
            /deep/.process,
            /deep/.playing,
            /deep/.preload {
              pointer-events: unset;
            }
          }
          i {
            font-size: 20px;
          }
          .full {
            font-weight: 600;
            font-size: 17px;
          }
          .part-cover {
            position: absolute;
            bottom: -46px;
            left: 50%;
            height: auto;
            background: rgba(0, 0, 0, 0);
            z-index: -2;
            transition: all 0.3s;
            opacity: 0;
            border-top-left-radius: 3px;
            border-top-right-radius: 3px;
            transform: scale(0);
            span {
              display: block;
              width: 100%;
              height: 34px;
              font-size: 14px;
              font-family: PingFangSC-Regular, PingFang SC;
              font-weight: 400;
              color: #fff;
              line-height: 34px;
              text-align: center;
              transition: all 0.3s;
              &:hover {
                color: #10c8af;
                background: #3e3e40;
              }
            }
            .num{
              height: 30px;
              line-height: 30px;
              padding: 0;
              margin: 0;
              font-size: 12px;
              cursor: initial;
              &:hover {
                color: #fff;
                background: rgba(0, 0, 0, 0);
              }
            }
            .play-span {
              color: #10c8af;
            }
            .volume-line {
              width: 4px;
              height: 100px;
              margin: 0 16px 10px;
              border-radius: 2px;
              background: #fff;
              position: relative;
              display: flex;
              flex-direction: column;
              justify-content: flex-end;
              align-items: center;
              .volume-straight {
                width: 4px;
                height: 0px;
                border-radius: 2px;
                background: #10c8af;
              }
              .volume-check {
                position: absolute;
                bottom: -5px;
                left: 50%;
                margin-left: -5px;
                width: 10px;
                height: 10px;
                border-radius: 100%;
                background: #fff;
                box-shadow: 0 -1px 1px #aaa;
                box-sizing: border-box;
                z-index: 4;
              }
            }
          }
          .reward {
            margin-left: -35px;
            width: 70px;
          }
          .speed {
            margin-left: -35px;
            width: 70px;
          }
          .quality {
            margin-left: -50px;
            width: 100px;
          }
          .volume {
            margin-left: -20px;
            width: 36px;
          }
        }
        .check-part {
          color: #10c8af;
        }
      }
    }
  }
  .show-control{
    opacity: 1;
    bottom: 0;
  }
}
#lee-player {
  width: 100% !important;
  height: 100%!important;
}
</style>
