<template>
  <div class="player" v-show="playList.length>0">
    <!-- 正常大小的播放器界面-->
    <transition
      name="normal"
      @enter="enter"
      @after-enter="afterEnter"
      @leave="leave"
      @after-leave="afterLeave"
    >
      <!-- 充满屏幕时显示正常播放器 -->
      <div class="normal-player" v-show="fullScreen">
        <!-- 背景透明大图 -->
        <div class="background">
          <img width="100%" height="100%" :src="(currentSong.al && currentSong.al.picUrl) || (currentSong.artists && currentSong.artists[0].img1v1Url)">
        </div>
        <!-- 顶部 包括返回图标歌名 歌手名 -->
        <div class="top">
          <div class="back" @click="back">
            <i class="icon">&#xe8e2;</i>
          </div>
          <h1 class="title" v-html="currentSong.name"></h1>
          <h2 class="subtitle" v-html="(currentSong.ar && currentSong.ar[0].name) || (currentSong.artists && currentSong.artists[0].name)"></h2>
        </div>
        <!-- 中间  -->
        <div
          class="middle"
          @touchstart="middleTouchStart"
          @touchmove="middleTouchMove"
          @touchend="middleTouchEnd"
        >
          <!-- 图片界面 包括图片专辑和滚动的小歌词-->
          <div class="middle-l" ref="middleL">
            <div class="cd-wrapper" ref="cdWrapper">
              <div class="cd" ref="imageWrapper">
                <img
                  ref="image"
                  :class="cdCls"
                  class="image"
                  :src="(currentSong.al && currentSong.al.picUrl) || (currentSong.artists && currentSong.artists[0].img1v1Url)"
                >
              </div>
            </div>
            <!-- 获取高亮歌词 -->
            <div class="playing-lyric-wrapper">
              <div class="playing-lyric">{{playingLyric}}</div>
            </div>
          </div>
          <!-- 歌词界面 -->
          <v-scroll class="middle-r" ref="lyricList" :data="currentLyric && currentLyric.lines">
            <div class="lyric-wrapper">
              <div v-if="currentLyric">
                <p
                  ref="lyricLine"
                  class="text"
                  :class="{'current': currentLineNum ===index}"   
                  v-for="(line,index) in currentLyric.lines"
                  :key="index"
                >{{line.txt}}</p>
              </div>
              <div class="pure-music" v-show="isPureMusic">
                <p>{{pureMusicLyric}}</p>
              </div>
            </div>
          </v-scroll>
        </div>
        <!-- 底部 -->
        <div class="bottom">
          <!-- 两个点 -->
          <div class="dot-wrapper">
            <span class="dot" :class="{'active':currentShow==='cd'}"></span>
            <span class="dot" :class="{'active':currentShow==='lyric'}"></span>
          </div>
          <!-- 进度条 -->
          <div class="progress-wrapper">
            <span class="time time-l">{{format(currentTime)}}</span>
            <div class="progress-bar-wrapper">
              <!-- 引入v-progress-curve组件 -->
              <v-progress-curve :percent="percent" />
            </div>
            <span class="time time-r">{{format(duration)}}</span>
          </div>
          <div class="operators-box">
            <div class="operators">
              <!-- 循环播放键 -->
              <div class="icon-box i-left" @click="changeMode">
                <i class="icon" style="font-size: 20px">&#xe819;</i>
              </div>
              <!-- 上一首 -->
              <div class="icon-box i-left" :class="disableCls">
                <i @click="prev" class="icon">&#xe61e;</i>
              </div>
              <!-- 播放键 -->
              <div class="icon-box i-center" :class="disableCls">
                <div>
                  <i class="icon" v-if="playing" @click="togglePlaying">&#xe644;</i>
                  <i class="icon icon-pause" v-else @click="togglePlaying">&#xe630;</i>
                </div>
              </div>
              <!-- 下一首 -->
              <div class="icon-box i-right" :class="disableCls">
                <i @click="next" class="icon">&#xe604;</i>
              </div>
              <!-- 播放列表 -->
              <div class="icon-box i-right" @click="showPlaylist">
                <i class="icon" style="font-size: 28px">&#xe927;</i>
              </div>
            </div>
          </div>
        </div>
      </div>
    </transition>
    <!-- 小播放器 -->
    <transition name="mini">
      <div class="mini-player" v-show="!fullScreen" @click="open">
        <!-- 小图 -->
        <div class="picture">
          <div class="imgWrapper" ref="miniWrapper">
            <!-- 图片懒加载 先只加载页面上看到的图片 -->
            <img
              ref="miniImage"
              :class="cdCls"
              width="40"
              height="40"
              v-lazy="(currentSong.al && currentSong.al.picUrl) || (currentSong.artists && currentSong.artists[0].img1v1Url)"
            >
          </div>
        </div>
        <div class="text">
          <h2 class="name" v-html="currentSong.name"></h2>
          <p class="desc" v-html="(currentSong.ar && currentSong.ar[0].name) || (currentSong.artists && currentSong.artists[0].name)"></p>
        </div>
        <div class="control" @click.stop="togglePlaying">
          <i class="icon icon-mini" v-if="playing">&#xe60a;</i>
          <i class="icon icon-mini" v-else>&#xe606;</i>
        </div>
        <div class="control" @click.stop="next">
          <i class="icon" style="font-size: 24px">&#xe718;</i>
        </div>
        <div class="control" @click.stop="showPlaylist">
          <i class="icon">&#xe927;</i>
        </div>
        <!-- 进度条 -->
        <div class="bottom-progress-bar">
          <!-- 百分比三位数 -->
          <div class="bottom-progress" :style="{width: (currentTime / duration).toFixed(3)*100 + '%'}"></div>
        </div>
      </div>
    </transition>
    <!-- playlist组件 -->
    <v-playlist ref="playList"></v-playlist>
    <!-- 定义audio事件 -->
    <audio
      ref="audio"
      @playing="ready"
      @error="error"
      @timeupdate="updateTime"
      @ended="end"
      @pause="paused"
    ></audio>
  </div>
</template>

<script type="text/ecmascript-6">   /* 让编辑器识别es6语法 */
import Lyric from 'lyric-parser'
import animations from 'create-keyframe-animation'
import scroll from '@/components/scroll'
import progressCurve from '@/components/progressCurve'
import { playerMixin } from '@/common/js/mixin'
import { prefixStyle } from '@/common/js/dom'   /* 为了浏览器兼容，自动加前缀 */
import api from '@/api'
import playList from '@/components/playList'
import { mapGetters, mapMutations, mapActions } from 'vuex'

const transform = prefixStyle('transform')
const transitionDuration = prefixStyle('transitionDuration')

//[00:00.00] 匹配歌词中时间的部分 其中.后面的数字只匹配不获取
const timeExp = /\[(\d{2,}):(\d{2})(?:\.(\d{2,3}))?]/g

export default {
  name: 'play',
  mixins: [playerMixin],
  components: {
    'v-scroll': scroll,
    'v-progress-curve': progressCurve,
    'v-playlist': playList
  },
  data() {
    return {
      songReady: false,
      currentTime: 0,
      duration: 0,
      currentLyric: null,
      currentLineNum: 0,
      currentShow: 'cd',
      playingLyric: '',   //高亮歌词
      isPureMusic: false,
      pureMusicLyric: ''
    }
  },
  computed: {
    //设置图片的class值 播放时设置为play 给play设置成360°旋转
    cdCls() {
      return this.playing ? 'play' : ''    //playing值默认为false 在music.js中定义
    },
    //设置左中右播放键的class值
    disableCls() {
      return this.songReady ? '' : 'disable'   //songReady默认为false
    },
    //时间百分比
    percent() {
      return this.currentTime / this.duration
    },
    ...mapGetters([
      'currentIndex',
      'fullScreen',
      'playing'
    ])
  },
  //创建一个空对象 持续添加方法 用于cd和歌词的切换功能
  created() {
    this.touch = {}
  },
  methods: {
    /* 左上角返回键 */
    back() {
      this.setFullScreen(false)
    },
    /* 点击小播放器进入正常播放器 */
    open() {
      this.setFullScreen(true)
    },
    /* 进入正常播放器函数 */
    enter(el, done) {
      const { x, y, scale } = this._getPosAndScale()    /* 解构 */
      /* 引入animation使用create-keyframe-animation插件 */
      let animation = {
        //第0帧的时候先让图片缩小
        0: { transform: `translate3d(${x}px,${y}px,0) scale(${scale})` },
        //60%的时候让图片回到cd中心，变大
        60: { transform: `translate3d(0,0,0) scale(1.1)` },
        //100%变回原来的尺寸，会有一个弹回的效果
        100: { transform: `translate3d(0,0,0) scale(1)` }
      }
      //注册动画
      animations.registerAnimation({
        name: 'move',
        animation,   //插入自定义动画
        presets: {
          duration: 400,    //持续时间
          easing: 'linear'   //过度效果线性
        }
      })
      //执行动画 挂载元素 name 
      animations.runAnimation(this.$refs.cdWrapper, 'move', done)
    },
    afterEnter() {
      //取消动画
      animations.unregisterAnimation('move')
      this.$refs.cdWrapper.style.animation = ''
    },
    leave(el, done) {
      this.$refs.cdWrapper.style.transition = 'all 0.4s'
      const { x, y, scale } = this._getPosAndScale()
      //直接移动变小
      this.$refs.cdWrapper.style[transform] = `translate3d(${x}px,${y}px,0) scale(${scale})`
      const timer = setTimeout(done, 400)
      //监听transitionend 事件在 CSS 完成过渡后触发done回调  
      this.$refs.cdWrapper.addEventListener('transitionend', () => {
        clearTimeout(timer)
        done()
      })
    },
    afterLeave() {
      this.$refs.cdWrapper.style.transition = ''
      this.$refs.cdWrapper.style[transform] = ''
    },
    //播放 暂停
    togglePlaying() {
      if (!this.songReady) {
        return
      }
      //点击设成相反的播放状态
      this.setPlaying(!this.playing)
      //切换歌词播放/暂停状态
       if (this.currentLyric) {
        this.currentLyric.togglePlay()
      } 
    },
    //歌曲播放结束时触发
    end() {
      this.currentTime = 0
      this.next()
    },
    /* 循环播放时 */
    loop() {  
      this.$refs.audio.currentTime = 0
      this.$refs.audio.play()    //播放音乐
      this.setPlaying(true)
      if (this.currentLyric) {
        this.currentLyric.seek(0)     //歌词跳转
      }
    },
    /* 下一首 */
    next() {
      if (!this.songReady) {
        return
      }
      /* 只有一首歌时一直重新播放 */
      if (this.playList.length === 1) {
        this.loop()
        return
      } else {
        let index = this.currentIndex + 1
        /* 播放的是最后一首歌切换到第一首 */
        if (index === this.playList.length) {
          index = 0
        }
        this.setCurrentIndex(index)
        if (!this.playing) {
          this.togglePlaying()
        }
      }
    },
    /* 前一首 */
    prev() {
      if (!this.songReady) {
        return
      }
      if (this.playList.length === 1) {
        this.loop()
        return
      } else {
        let index = this.currentIndex - 1
        /* 如果播放的是第一首 切换到最后一首 */
        if (index === -1) {
          index = this.playList.length - 1
        }
        this.setCurrentIndex(index)
        if (!this.playing) {
          this.togglePlaying()
        }
      }
    },
    /* 循环播放键 */
    changeMode() {
      this.$toast('开发中，敬请期待...')
    },
    /* @playing事件 */
    ready() {
      clearTimeout(this.timer)
      // 监听 playing 这个事件可以确保慢网速或者快速切换歌曲导致的 DOM Exception
      this.songReady = true
      this.canLyricPlay = true   /* 已经播放了音乐 */
      this.duration = this.$refs.audio.duration    /* 获取当前歌曲总时间 */
      this.savePlayHistory(this.currentSong)    /* 保存播放历史 异步操作 */
      // 如果歌曲的播放晚于歌词的出现，播放的时候需要同步歌词
      if (this.currentLyric && !this.isPureMusic) {
        //偏移歌词到拖动时间的对应位置
        this.currentLyric.seek(this.currentTime * 1000)
      }
    },
    //暂停
    paused() {
      this.setPlaying(false)
      if (this.currentLyric) {
        this.currentLyric.stop()
      }
    },
    error() {
      clearTimeout(this.timer)
      this.songReady = true
    },
    /* 播放时时间的改变 */
    updateTime(e) {
      this.currentTime = e.target.currentTime
    },
    /* 显示时间 */
    format(interval) {
      /* 位运算符取整 */
      interval = interval | 0    
      const minute = interval / 60 | 0
      const second = this._pad(interval % 60)
      return `${minute}:${second}`
    },
    /* 获取解析后的歌词 */
    getLyric(id) {
      api.MusicLyric(id).then(res => {
        if (res.code === 200) {
          /* 实例化Lyric对象 */
          this.currentLyric = new Lyric(res.lrc.lyric, this.handleLyric)
          this.isPureMusic = !this.currentLyric.lines.length
          if (this.isPureMusic) {
            /* 查找时间部分并替换成空字符串 并去掉字符串两端多于的空格*/
            this.pureMusicLyric = this.currentLyric.lrc.replace(timeExp, '').trim()
            this.playingLyric = this.pureMusicLyric
          } else {
            if (this.playing && this.canLyricPlay) {
              // 这个时候有可能用户已经播放了歌曲，要切到对应位置
              this.currentLyric.seek(this.currentTime * 1000)
            }
          }
        }
      })
    },
    /* 得到当前currentLingNum值，判断如果歌曲播放，调用Lyric的play() */
    handleLyric({ lineNum, txt }) {
      if (!this.$refs.lyricLine) {
        return
      }
      this.currentLineNum = lineNum
      /* 判断如果行数大于5就滚动 保证歌词在中间播放 */
      if (lineNum > 5) {
        let lineEl = this.$refs.lyricLine[lineNum - 5]
        //从lineE1行开始 滚动播放
        this.$refs.lyricList.scrollToElement(lineEl, 1000)
      } else {
        this.$refs.lyricList.scrollTo(0, 0, 1000)   //滚动到底部
      }
      this.playingLyric = txt
    },
    /* 显示播放列表 */
    showPlaylist() {
      this.$refs.playList.show()
    },
    /* 中间图片和歌词的切换 */
    middleTouchStart(e) {
      /* 为touch添加一个initiated方法 true开始滑动 */
      this.touch.initiated = true
      // 用来判断是否是一次移动 否则在移动一次之后点击就可以切换
      this.touch.moved = false
      const touch = e.touches[0]    //取到返回的对象 以取到对象下的pageX
      this.touch.startX = touch.pageX
      this.touch.startY = touch.pageY
    },
    middleTouchMove(e) {
      if (!this.touch.initiated) {
        return
      }
      const touch = e.touches[0]
      //从右→往左滑动，deltaX是负数
      //从左→往右滑动，deltaX是正数
      const deltaX = touch.pageX - this.touch.startX   //移动的偏移量
      const deltaY = touch.pageY - this.touch.startY
      //当纵轴的偏移量大于横轴就不移动 此方法返回绝对值
      if (Math.abs(deltaY) > Math.abs(deltaX)) {
        return
      }
      if (!this.touch.moved) {
        this.touch.moved = true
      }
      const left = this.currentShow === 'cd' ? 0 : -window.innerWidth
      //临界值
      const offsetWidth = Math.min(0, Math.max(-window.innerWidth, left + deltaX))
      //控制透明度实现渐入渐出效果
      this.touch.percent = Math.abs(offsetWidth / window.innerWidth)
      this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px,0,0)`
      this.$refs.lyricList.$el.style[transitionDuration] = 0
      this.$refs.middleL.style.opacity = 1 - this.touch.percent
      this.$refs.middleL.style[transitionDuration] = 0
    },
    middleTouchEnd() {
      if (!this.touch.moved) {
        return
      }
      let offsetWidth
      let opacity
      //从右向左滑动
      if (this.currentShow === 'cd') {
        //percent临界值0.1  0.9
        if (this.touch.percent > 0.1) {
          offsetWidth = -window.innerWidth
          opacity = 0
          this.currentShow = 'lyric'
        } else {
          offsetWidth = 0
          opacity = 1
        }
      } else {
        if (this.touch.percent < 0.9) {
          offsetWidth = 0
          this.currentShow = 'cd'
          opacity = 1
        } else {
          offsetWidth = -window.innerWidth
          opacity = 0
        }
      }
      const time = 300
      this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px,0,0)`
      //设置过度时间
      this.$refs.lyricList.$el.style[transitionDuration] = `${time}ms`
      this.$refs.middleL.style.opacity = opacity
      this.$refs.middleL.style[transitionDuration] = `${time}ms`
      this.touch.initiated = false
    },
    /* 如果秒数是一位数就在前面加一个0 */
    _pad(num, n = 2) {
      /* 转化为字符串 */
      let len = num.toString().length
      while (len < n) {
        num = '0' + num
        len++
      }
      return num
    },
    //获得偏移量 以及缩放倍数 偏移量是两个中心点的距离
    _getPosAndScale() {
      const targetWidth = 40   /* 小图宽度 */
      const paddingLeft = 40    /* 左内边距 */
      const paddingBottom = 30    /* 下内边距 */
      const paddingTop = 80    /* 上内边距 */
      const width = window.innerWidth * 0.8     /* 获取窗口宽度 */
      const scale = targetWidth / width   /* 缩放倍数 */
      const x = -(window.innerWidth / 2 - paddingLeft)    /* x偏移量     */
      const y = window.innerHeight - paddingTop - width / 2 - paddingBottom    /* y偏移量 */
      return {
        x,
        y,
        scale
      }
    },
    /**
     * 计算内层Image的transform，并同步到外层容器
     * @param wrapper
     * @param inner
     */
    //将cd的内层图片的transform（旋转）同步到外层容器中
    syncWrapperTransform(wrapper, inner) {
      if (!this.$refs[wrapper]) {
        return
      }
      let imageWrapper = this.$refs[wrapper]
      let image = this.$refs[inner]
      let wTransform = getComputedStyle(imageWrapper)[transform]
      let iTransform = getComputedStyle(image)[transform]
      imageWrapper.style[transform] = wTransform === 'none' ? iTransform : iTransform.concat(' ', wTransform)
    },
    ...mapMutations({
      setFullScreen: 'SET_FULL_SCREEN'
    }),
    ...mapActions([
      'savePlayHistory'
    ])
  },
  /* 监测变动 */
  watch: {
    /* 监测歌曲变化 */
    async currentSong(newSong, oldSong) {
      /* 通过id判断当前歌曲没变，不执行任何操作 */
      if (!newSong.id || newSong.id === oldSong.id) {
        return
      }
      //url发生变化
      if (!newSong.url) {
        const { data, code } = await api.MusicUrl(newSong.id)
        //请求成功
        if (data && code === 200) {
          newSong = { ...newSong, url: data[0].url }
        } else {      //请求失败
          alert('请求音乐出错啦')
        }
      }
      //重置
      this.songReady = false
      this.canLyricPlay = false
      if (this.currentLyric) {
        this.currentLyric.stop()
        // 重置为null
        this.currentLyric = null
        this.currentTime = 0
        this.playingLyric = ''
        this.currentLineNum = 0
      }
      //播放音乐
      this.$refs.audio.src = newSong.url
      this.$refs.audio.play()
      // 若歌曲 5s 未播放，则认为超时，修改状态确保可以切换歌曲。
      clearTimeout(this.timer)
      this.timer = setTimeout(() => {
        this.songReady = true
      }, 5000)
      //调用getLyric函数
      this.getLyric(newSong.id)
    },
    /* 监测播放状态 */
    playing(newPlaying) {
      if (!this.songReady) {
        return
      }
      const audio = this.$refs.audio   //取到audio元素
      this.$nextTick(() => {
        newPlaying ? audio.play() : audio.pause()
      })
      if (!newPlaying) {
        if (this.fullScreen) {
          this.syncWrapperTransform('imageWrapper', 'image')
        } else {
          this.syncWrapperTransform('miniWrapper', 'miniImage')
        }
      }
    },
    /* 监测正常播放界面 拖动歌词20ms回到当前歌词*/
    fullScreen(newVal) {
      if (newVal) {
        setTimeout(() => {
          this.$refs.lyricList.refresh()
        }, 20)
      }
    }
  }
}
</script>

<style scoped lang="scss">
@import "../assets/css/function.scss";

.player {
  z-index: 150;
  .normal-player {
    position: fixed;
    left: 0;
    right: 0;
    top: 0;
    bottom: 0;
    z-index: 150;
    background: rgb(8, 5, 58);
    .background {
      position: absolute;
      left: 0;
      top: 0;
      width: 100%;
      height: 100%;
      z-index: -1;
      opacity: 0.6;
      filter: blur(20px);   //设置模糊程度 值越大越模糊
    }
    .top {
      position: relative;
      margin-bottom: 25px;
      .back {
        position: absolute;
        top: 0;
        left: px2rem(12px);
        z-index: 50;
        .icon {
          display: block;
          height: px2rem(100px);   /* 一个插件 用于px转换为rem */
          line-height: px2rem(100px);
          padding: 0 px2rem(30px);
          font-size: 22px;
        }
      }
      .title {
        width: 70%;
        margin: 0 auto;
        line-height: px2rem(100px);
        text-align: center;
        text-overflow: ellipsis;    /* 过长标题显示为省略号 */
        overflow: hidden;
        white-space: nowrap;   /* 不会换行 */
        font-size: 18px;
        color: #fff;
      }
      /* 歌手名 */
      .subtitle {
        line-height: px2rem(40px);
        text-align: center;
        font-size: 14px;
        color: #fff;
      }
    }
    .middle {
      position: fixed;
      width: 100%;
      top: px2rem(180px);
      bottom: px2rem(340px);
      white-space: nowrap;
      font-size: 0;
      .middle-l {
        display: inline-block;
        vertical-align: top;
        position: relative;
        width: 100%;
        height: 0;
        padding-top: 80%;
        .cd-wrapper {
          position: absolute;
          left: 10%;
          top: 0;
          width: 80%;
          box-sizing: border-box;
          height: 100%;
          .cd {
            width: 100%;
            height: 100%;
            border-radius: 50%;
            .image {
              position: absolute;
              left: 0;
              top: 0;
              width: 100%;
              height: 100%;
              box-sizing: border-box;
              border-radius: 50%;
              border: 10px solid rgba(255, 255, 255, 0.1);
            }
            /* cd转动 */
            .play {
              animation: rotate 20s linear infinite;
            }
          }
        }
        /* 小歌词 */
        .playing-lyric-wrapper {
          width: 80%;
          margin: 30px auto 0 auto;
          overflow: hidden;
          text-align: center;
          .playing-lyric {
            height: px2rem(40px);
            line-height: px2rem(40px);
            font-size: 14px;
            color: hsla(0, 0%, 100%, 0.5);
          }
        }
      }
      .middle-r {
        display: inline-block;
        vertical-align: top;
        width: 100%;
        height: 100%;
        overflow: hidden;
        .lyric-wrapper {
          width: 80%;
          margin: 0 auto;
          overflow: hidden;
          text-align: center;
          .text {
            line-height: px2rem(64px);
            color: hsla(0, 0%, 100%, 0.5);
            font-size: 14px;
            /* 高亮歌词 */
            &.current {
              color: #fff;
            }
          }
          .pure-music {
            padding-top: 50%;
            line-height: px2rem(64px);
            color: hsla(0, 0%, 100%, 0.5);
            font-size: 14px;
          }
        }
      }
    }
    .bottom {
      position: absolute;
      bottom: px2rem(200px);
      width: 100%;
      .dot-wrapper {
        text-align: center;
        font-size: 0;
        .dot {
          display: inline-block;
          vertical-align: middle;
          margin: 0 px2rem(8px);
          width: px2rem(16px);
          height: px2rem(16px);
          border-radius: 50%;
          background: hsla(0, 0%, 100%, 0.5);
          &.active {
            width: px2rem(40px);
            border-radius: px2rem(10px);
            background: hsla(0, 0%, 100%, 0.8);    /* 色调 饱和度 亮度 透明度 */
          }
        }
      }
      .progress-wrapper {
        display: flex;
        justify-content: space-between;
        align-items: center;
        width: 80%;
        margin: 0px auto;
        padding: 10px 0;
        .time {
          color: #fff;
          font-size: 12px;
          flex: 0 0 30px;
          line-height: px2rem(60px);
          width: px2rem(60px);
          &.time-l {
            text-align: left;
          }
          &.time-r {
            text-align: right;
          }
        }
        .progress-bar-wrapper {
          position: absolute;
          left: 0;
          right: 0;
          top: 0;
        }
      }
      .operators-box {
        width: px2rem(1200px);
        height: px2rem(1200px);
        position: absolute;
        top: px2rem(80px);
        left: 50%;
        transform: translate3d(-50%, 0, 0);
        overflow: hidden;
        z-index: -1;
        &::after {
          content: '';    //必须定义
          width: 100%;
          height: 100%;
          background: #ea2448;
          /* 裁剪前需绝对定位 */
          position: absolute;
          clip: rect(0 px2rem(600px) px2rem(1200px) 0);  
          transform: rotate(90deg);
          border-radius: 50%;
        }
      }
      .operators {
        position: absolute;
        top: px2rem(70px);
        display: flex;
        width: px2rem(660px);
        height: px2rem(132px);
        margin-left: 50%;
        transform: translate3d(-50%, 0, 0);
        align-items: center;
        z-index: 100;

        .icon-box {
          flex: 1;
          height: 100%;
          display: flex;
          justify-content: center;
          align-items: center;
          &.disable {
            color: #222;
          }
          i {
            font-size: 26px;
          }
        }
        .i-left {
          text-align: right;
        }
        .i-center {
          margin: 0 px2rem(20px);
          > div {
            width: px2rem(120px);
            height: px2rem(120px);
            display: flex;
            align-items: center;
            justify-content: center;
            border-radius: 50%;
            background: #fff;
            .icon {
              margin-top: px2rem(4px);
              font-size: 30px;
              display: inline-block;
              color: #4436b1;
              &.icon-pause {
                margin: px2rem(10px) 0 0 px2rem(10px);
              }
            }
          }
        }
        .i-right {
          text-align: left;
        }
      }
    }
    &.normal-enter-active,
    &.normal-leave-active {
      transition: all 0.4s;
      .top,
      .bottom {
        transition: all 0.4s cubic-bezier(0.86, 0.18, 0.82, 1.32);
      }
    }
    &.normal-enter,
    &.normal-leave-to {
      opacity: 0;
      .top {
        transform: translate3d(0, -100px, 0);
      }
      .bottom {
        transform: translate3d(0, 100px, 0);
      }
    }
  }

  .mini-player {
    display: flex;
    align-items: center;
    position: fixed;
    left: 0;
    bottom: px2rem(5px);
    z-index: 180;
    width: 100%;
    height: px2rem(105px);
    background: #ea2448;
    &.mini-enter-active,
    &.mini-leave-active {
      transition: all 0.4s;
    }
    &.mini-enter,
    &.mini-leave-to {
      opacity: 0;
    }
    .picture {
      flex: 0 0 px2rem(80px);
      width: px2rem(80px);
      height: px2rem(80px);
      padding: 0 px2rem(20px) 0 px2rem(40px);
      .imgWrapper {
        height: 100%;
        width: 100%;
        img {
          border-radius: 50%;
          &.play {
            animation: rotate 10s linear infinite;
          }
          &.pause {
            animation-play-state: paused;  /* 暂停动画 */
          }
        }
      }
    }

    .text {
      display: flex;
      flex-direction: column;
      justify-content: center;
      flex: 1;
      line-height: px2rem(40px);
      overflow: hidden;
      .name {
        margin-bottom: 2px;
        text-overflow: ellipsis;
        overflow: hidden;
        white-space: nowrap;
        font-size: 14px;
        color: #fff;
      }
      .desc {
        text-overflow: ellipsis;
        overflow: hidden;
        white-space: nowrap;
        font-size: 12px;
        color: hsla(0, 0%, 100%, 0.3);
      }
    }
    .control {
      flex: 0 0 px2rem(60px);
      width: px2rem(60px);
      text-align: center;
      padding: 0 px2rem(20px);
      .icon {
        font-size: 30px;
        color: #fff;
      }
    }
    .bottom-progress-bar {
      position: fixed;
      left: 0;
      right: 0;
      bottom: 0;
      height: px2rem(6px);
      background: #fe7498;
      .bottom-progress {
        height: 100%;
        background: linear-gradient(#902541, #902444);
      }
    }
  }
}

@keyframes rotate {
  0% {
    transform: rotate(0);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>