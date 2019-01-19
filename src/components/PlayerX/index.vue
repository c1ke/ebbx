<template>
  <div
    id="coral-root"
    class="container"
    @contextmenu="handleContextMenu"
  >
    <video ref="video" />
    <big-play-button @click="$refs.video.play()" v-if="!hasPlayed" />
    <div class="loading" v-if="isBuffering">
      <loader />
    </div>
    <div class="resource-loading" v-if="resourceLoading">
      <div class="caption">全速載入中⋯</div>
      <div class="longfazers">
        <span />
        <span />
        <span />
        <span />
      </div>
    </div>
    <div
      :class="{ 'controls': true, 'idle': idleClassActive }"
      @mouseenter="onMouseEnterListener"
      @mousemove="onMouseMoveListener"
      @mouseleave="onMouseLeaveListener"
      v-if="$refs.video"
    >
      <player-ads />
      <overlay-controls />
      <up-next-overlay v-if="!isMobile && showUpNextOverlay" />
      <seek-bar />
      <portal-target name="player-settings" />
    </div>
    <portal-target slim name="danmaku" />
  </div>
</template>

<script>
import { mapState } from 'vuex'
import Loader from '@/components/Loader'

import BigPlayButton from './BigPlayButton'
import PlayerAds from './PlayerAds'
import UpNextOverlay from './UpNextOverlay'
import OverlayControls from './OverlayControls'
import SeekBar from './SeekBar'

const HISTORY_SYNC_DURATION = 30 * 1000

window.shaka.polyfill.installAll()

export default {
  name: 'player-x',
  components: {
    Loader,
    BigPlayButton,
    PlayerAds,
    UpNextOverlay,
    OverlayControls,
    SeekBar,
  },
  props: {
    onReference: {
      type: Function,
      required: true,
    },
    vdata: {
      type: Object,
      required: false,
    },
  },
  data() {
    return {
      player: null,
      resourceLoading: false,
      sourceLoader: null,
      hasPlayed: false,
      isFullscreen: false,
      isLightOff: false,
      isBuffering: false,
      paused: false,
      isIdle: false,
      stayAwake: false,
      lastHistorySynced: Date.now() + HISTORY_SYNC_DURATION,
      showUpNextOverlay: false,
      // from page
      pageComponent: null,
      autoPlay: false,
      startTime: null,
    }
  },
  computed: {
    ...mapState({
      isMobile: (state) => state.isMobile,
    }),
    peekSrc() {
      const [_, path] = this.vdata.src.match(/(\/[0-9a-f]{64,}\/.+)$/)
      return `https://ebb.io/resload/p${path}`.replace('/master.csv', '.jpg')
    },
    video() {
      return this.$refs.video
    },
    idleClassActive() {
      const { isIdle, paused, stayAwake, showUpNextOverlay } = this
      return isIdle && !paused && !showUpNextOverlay && !stayAwake
    },
  },
  methods: {
    handleContextMenu(event) {
      if (!event.target.hasAttribute('data-playback-capture')) {
        event.preventDefault()
      }
    },
    syncWatchHistory(force = false) {
      const { watchHistory } = this.pageComponent
      if (!watchHistory) {
        return
      }
      const now = Date.now()
      if (force || (now - this.lastHistorySynced) >= HISTORY_SYNC_DURATION) {
        const { seasonId, title } = watchHistory
        this.pageComponent.updateWatchHistory(seasonId, title, this.video.currentTime)
        this.lastHistorySynced = now
      }
    },
    onPlayListener() {
      if (!this.hasPlayed) {
        this.hasPlayed = true
      }
      this.paused = false
    },
    onPauseListener() {
      this.paused = true
    },
    onTimeUpdateListener() {
      const { currentTime, duration } = this.video
      if (currentTime >= duration) {
        this.showUpNextOverlay = true
      } else {
        this.showUpNextOverlay = false
      }
      // sync history
      this.syncWatchHistory()
    },
    onKeydownListener(event) {
      if (event.target.nodeName === 'INPUT') {
        return
      }
      const { video } = this.$refs
      switch (event.which) {
        case 32: {
          // space
          event.preventDefault()
          if (video.paused) {
            video.play()
          } else {
            video.pause()
          }
          break
        }
        case 37: {
          // arrow right
          event.preventDefault()
          video.currentTime = Math.max(0, video.currentTime - 5)
          break
        }
        case 39: {
          // arrow right
          event.preventDefault()
          video.currentTime = Math.min(video.duration, video.currentTime + 5)
          break
        }
        case 70: {
          // char f
          event.preventDefault()
          this.$emit('signal-toggle-fullscreen')
          break
        }
        case 77: {
          // char m
          event.preventDefault()
          this.$emit('signal-toggle-mute')
          break
        }
        default:
          break
      }
    },
    onMouseEnterListener() {
      this.isMouseOverOnControls = true
    },
    onMouseMoveListener() {
      if (!this.isMouseOverOnControls) {
        return
      }
      this.isIdle = false
      clearTimeout(this.idleTimeout)
      // if user inactive for 3 seconds, set to idle
      this.idleTimeout = setTimeout(() => { this.isIdle = true }, 3000)
    },
    onMouseLeaveListener() {
      if (this.initialIdleFlag) {
        // do not trigger idle on mouse leave for first 3 seconds of video play
        return
      }
      clearTimeout(this.idleTimeout)
      this.isIdle = true
    },
    togglePlay() {
      if (this.paused) {
        this.video.play()
      } else {
        this.video.pause()
      }
    },
  },
  watch: {
    async vdata({ src }) {
      // reset flags
      this.hasPlayed = false

      // update source
      this.video.autoplay = this.autoPlay
      await this.sourceLoader(src)

      // restore start time
      this.video.currentTime = this.startTime

      // trigger idle event after 3 seconds from video play
      const initialIdleListener = () => {
        this.idleTimeout = setTimeout(() => { this.isIdle = true }, 3000)
        // clear flag after 3 seconds
        this.initialIdleFlag = true
        setTimeout(() => { this.initialIdleFlag = false }, 3000)
        // force sync history on first play
        this.syncWatchHistory(true)
        // remove listener after playing
        this.video.removeEventListener('play', initialIdleListener)
      }
      this.video.addEventListener('play', initialIdleListener)
    },
    isIdle(idle) {
      // hide cursor in fullscreen if idle
      if (idle && this.isFullscreen) {
        document.body.setAttribute('style', 'cursor: none')
      } else {
        document.body.removeAttribute('style')
      }
    },
  },
  mounted() {
    this.onReference(this)

    this.video.crossOrigin = 'anonymous'
    this.video.playsInline = true
    this.video.addEventListener('play', this.onPlayListener)
    this.video.addEventListener('pause', this.onPauseListener)
    this.video.addEventListener('timeupdate', this.onTimeUpdateListener)
    window.addEventListener('keydown', this.onKeydownListener)

    if (window.shaka.Player.isBrowserSupported()) {
      this.player = new window.shaka.Player(this.video)
      this.player.configure({
        streaming: {
          bufferBehind: 25,
          bufferingGoal: 25,
          rebufferingGoal: 25,
          retryParameters: {
            timeout: 1000 * 5,
            maxAttempts: 5,
          },
        },
      })
      this.player.addEventListener('loading', () => {
        this.resourceLoading = true
      })
      this.player.addEventListener('trackschanged', () => {
        this.resourceLoading = false
      })
      this.player.addEventListener('buffering', (event) => {
        this.isBuffering = event.buffering
      })
      this.sourceLoader = async (src) => this.player.load(src)
    } else if (/iPad|iPhone|iPod/.test(navigator.platform)) {
      this.sourceLoader = async (src) => { this.video.src = src }
    }
  },
  beforeDestroy() {
    this.video.removeEventListener('play', this.onPlayListener)
    this.video.removeEventListener('pause', this.onPauseListener)
    this.video.removeEventListener('timeupdate', this.onTimeUpdateListener)
    window.removeEventListener('keydown', this.onKeydownListener)
  },
}
</script>

<style lang="scss" scoped>
.container {
  background: #000;
  height: 100%;
  flex: 1;
  flex-shrink: 0;
  position: relative;
  -webkit-tap-highlight-color: transparent;
  user-select: none;

  video {
    height: 100%;
    left: 0;
    position: absolute;
    top: 0;
    width: 100%;
  }

  // tablet, break comment section
  @media (max-width: 1135px) {
    height: auto;
    width: 100%;
  }
}

.loading {
  align-items: center;
  bottom: 0;
  display: flex;
  justify-content: center;
  left: 0;
  pointer-events: none;
  position: absolute;
  right: 0;
  top: 0;
  z-index: 99;
}

@keyframes lf1 {
  0% {
    left: 200%;
  }
  to {
    left: -200%;
    opacity: 0;
  }
}

@keyframes lf2 {
  0% {
    left: 200%;
  }
  to {
    left: -200%;
    opacity: 0;
  }
}

@keyframes lf3 {
  0% {
    left: 200%;
  }
  to {
    left: -100%;
    opacity: 0;
  }
}

@keyframes lf4 {
  0% {
    left: 200%;
  }
  to {
    left: -100%;
    opacity: 0;
  }
}

@keyframes speeder {
  0% { transform: translate(2px, 1px) rotate(0deg); }
  10% { transform: translate(-1px, -3px) rotate(-1deg); }
  20% { transform: translate(-2px, 0) rotate(1deg); }
  30% { transform: translate(1px, 2px) rotate(0deg); }
  40% { transform: translate(1px, -1px) rotate(1deg); }
  50% { transform: translate(-1px, 3px) rotate(-1deg); }
  60% { transform: translate(-1px, 1px) rotate(0deg); }
  70% { transform: translate(3px, 1px) rotate(-1deg); }
  80% { transform: translate(-2px, -1px) rotate(1deg); }
  90% { transform: translate(2px, 1px) rotate(0deg); }
  to { transform: translate(1px, -2px) rotate(-1deg); }
}

.resource-loading {
  background: #000;
  align-items: center;
  bottom: 0;
  display: flex;
  justify-content: center;
  left: 0;
  position: absolute;
  right: 0;
  top: 0;
  z-index: 110;

  .caption {
    animation: speeder .75s linear infinite;
    font-size: 1.5rem;
  }

  .longfazers {
    bottom: 0;
    left: 0;
    overflow: hidden;
    position: absolute;
    right: 0;
    top: 0;

    span {
      background: #888;
      background: linear-gradient(to right, transparent, #888 20%, #888 80%, transparent 100%);
      height: 2px;
      position: absolute;
      width: 20%;

      &:nth-child(1) {
        animation: lf1 .6s linear infinite;
        animation-delay: -5s;
        top: 20%;
      }

      &:nth-child(2) {
        animation: lf2 .8s linear infinite;
        animation-delay: -1s;
        top: 40%;
      }

      &:nth-child(3) {
        animation: lf3 .6s linear infinite;
        top: 60%;
      }

      &:nth-child(4) {
        animation: lf4 .5s linear infinite;
        animation-delay: -3s;
        top: 80%;
      }
    }
  }
}

// for click-to-seek
@keyframes fade-delay {
  to { opacity: 1; }
}

.controls {
  height: 100%;
  position: relative;
  transition: opacity .15s linear;
  width: 100%;

  &.idle {
    opacity: 0;
  }
}
</style>
