<template>
  <div class="home" id="whiteBoard"></div>
</template>

<script>
// @ is an alias to /src
import 'white-web-sdk/style/index.css'
var WhiteWebSdk = require('white-web-sdk').WhiteWebSdk
var whiteWebSdk = new WhiteWebSdk()

const EVENT_SSO_CHECK = 'EVENT_SSO_CHECK'
const EVENT_SSO_CONFLICT = 'EVENT_SSO_CONFLICT'

export default {
  name: 'home',
  components: {},
  data () {
    return {
      room: null,
      ssoHasChecked: false,
      userInfo: {
        id: 15
      }
    }
  },
  mounted () {
    window.addEventListener('resize', this.onWindowResize)
    console.log(this.$route.query)
    this.initWhiteRoom({
      uuid: this.$route.query.uuid,
      roomToken: this.$route.query.token,
      userPayload: { id: this.userInfo.id }
    })
  },
  beforeDestroy () {
    window.removeEventListener('resize', this.onWindowResize)
  },
  methods: {
    onWindowResize () {
      let room = this.room
      room && room.refreshViewSize()
    },
    initWhiteRoom (config) {
      whiteWebSdk
        .joinRoom(config, {
          onRoomStateChanged: (modifyState) => {
            if (modifyState.globalState) {
              // globalState 改变了
              console.log(modifyState.globalState, 'globalState 改变了')
            }
            if (modifyState.roomMembers) {
              console.log(modifyState.roomMembers, 'roomMembers 改变了')
            }
            if (modifyState.memberState) {
              // memberState 改变了
              console.log(modifyState.memberState, 'memberState 改变了')
            }
            if (modifyState.sceneState) {
              // sceneState 改变了
              console.log(modifyState.sceneState, 'sceneState 改变了')
            }
            if (modifyState.broadcastState) {
              // broadcastState 改变了
              console.log(modifyState.broadcastState, 'broadcastState 改变了')
            }
          }
        })
        .then(joinedRoom => {
          joinedRoom.bindHtmlElement(document.getElementById('whiteBoard'))
          this.room = joinedRoom
          this.subscribeWhiteEvents()
        })
    },
    // 白板相关的回调
    subscribeWhiteEvents () {
      let room = this.room
      room && room.addMagixEventListener(EVENT_SSO_CHECK, this.onSsoCheck)
      room && room.addMagixEventListener(EVENT_SSO_CONFLICT, this.onSsoConflict)
      if (!this.ssoHasChecked) {
        this.ssoHasChecked = true
        room && room.dispatchMagixEvent(EVENT_SSO_CHECK, {
          id: this.userInfo.id,
          userInfo: this.userInfo
        })
      }
    },
    // 检测重复登录的情况
    onSsoCheck (obj) {
      let data = obj.payload
      let room = this.room
      console.log('onSsoCheck', data, this.userInfo, this.userInfo !== data.userInfo)
      if (data && data.id === this.userInfo.id && // 判断id一致
        this.userInfo !== data.userInfo) { // 判断不是自己发的自定义事件
        console.log('onSsoCheck send conflict event')
        room && room.dispatchMagixEvent(EVENT_SSO_CONFLICT, {
          id: this.userInfo.id,
          ts: new Date().getTime(),
          userInfo: this.userInfo
        })
      }
    },
    // 处理重复登录的情况
    onSsoConflict (obj) {
      let data = obj.payload
      console.log('onSsoConflict', data, this.userInfo, this.userInfo !== data.userInfo)
      if (data && data.id === this.userInfo.id && // 判断id一致
        this.userInfo !== data.userInfo) { // 判断不是自己发的自定义事件
        console.log('重复登录确认，关闭后进入的页面')
        this.exitWhiteBoard()
        // ... 弹窗提醒用户
      }
    },
    // 退出白板
    exitWhiteBoard () {
      let room = this.room
      room && room.disconnect().then(() => {
        room.bindHtmlElement(null)
        console.log('离开白板房间')
        alert('重复打开标签，离开白板房间')
      })
    }
  }
}
</script>

<style>
  .home {
    width: 100%;
    height: 900px;
  }
</style>
