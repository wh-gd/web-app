<style lang="scss">
  @import "./index";
</style>

<template>
  <div class="modules-chat-body">
    <scroll-top-load :load-more="loadMoreMessage" :loading-html='loadingHtml'>
      <message-item v-for="item in messagesWithUserInfo" :message.sync="item"></message-item>
    </scroll-top-load>
  </div>
</template>

<script>
  import $ from 'jquery'
  import array from 'lodash/array'
  import { scrollBottom } from '../../../utils/enhance'
  import ScrollTopLoad from '../../commons/ScrollTopLoad/'
  import MessageItem from '../../modules/MessageItem/'
  import constants from '../../../constants/'

  import {
    getMessages,
    getUserInfoByIds,
    getCurMessage,
    clearMessages,
    toggleChannelSend
  } from '../../../vuex/actions'

  export default {
    components: {
      ScrollTopLoad,
      MessageItem
    },

    data () {
      return {
        startLoadMore: false,
        isloadFirst: true, // 是否为 channel 的首次加载，用来判断滚动跳位置
        messagesWithUserInfo: [],
        loadingHtml: '<div class="message-loading"><div class="v-clip" style="height: 35px; width: 35px; border-width: 2px; border-style: solid; border-color: rgb(4, 201, 191) rgb(4, 201, 191) transparent; border-radius: 100%; background: transparent !important;"></div></div>'
      }
    },

    vuex: {
      actions: {
        getMessages,
        getUserInfoByIds,
        getCurMessage,
        clearMessages,
        toggleChannelSend
      },
      getters: {
        channels: state => state.channel.channels,
        curChannelId: state => state.channel.curChannelId,
        seqMessages: state => state.message.seqMessages,
        curMessage: state => state.message.curMessage,
        messages: state => state.message.allMessages,
        usersInfoStorage: state => state.user.usersInfoStorage,
        channelSendState: state => state.message.channelSendState
      }
    },
    watch: {
      // 切换 channel 时，触发更新 messages
      curChannelId () {
        this.startLoadMore = false
        this.isloadFirst = true
        this.$broadcast('startLoading')
        this.clearMessages() // 临时方案，每次切换 channel 都会把所有 messages 数据清空

        let curMessage = this.messages && this.messages[this.curChannelId]
        let curChannel = this.channels[this.curChannelId]
        let messageNumber = curChannel.message_number

        if (messageNumber > 0) { // 首次拉取聊天数据，且数据数量不为0
          typeof curMessage === 'undefined' && this.getMessages(this.curChannelId, messageNumber)
        } else { // 信息数据量为0
          this.$broadcast('endLoading')
          this.messagesWithUserInfo = []
          // todo: 显示没有发生过任何消息时的样式
        }

        if (curChannel.live_schema === 1) {
          this.$dispatch('openRightPannel')
          this.$dispatch('openLiveVideo', curChannel.live_play_url)
        } else {
          this.$dispatch('closeRightPannel')
        }
      },

      // 更新 allMessages 时触发更新
      curMessage () {
        if (this.curMessage.length === 0) {
          return
        }
        // step 1: 去内存获取已存储的用户信息（如果不存在，存入待查询变量中，注意去重处理）
        // step 2: 把待查询的用户 ids 丢到接口中，集中获取需要的用户信息，然后并入到本地库中
        // step 3: 将 users 信息合并到 allMessages 集合中去
        // step 4: 渲染用户 allMessages

        let newUserIds = this._getNewUserIds(this.curMessage)
        if (newUserIds.length > 0) {
          return this.getUserInfoByIds(newUserIds)
        }
        this._updateRender()
      },

      usersInfoStorage () {
        this._updateRender()
      }
    },

    methods: {

      _updateRender () {
        if (!this.curChannelId) {
          return
        }

        if (this._sendMesRender()) {
          return
        }

        if (this._firstMesRender()) {
          return
        }

        if (this._moreMesRender()) {
          return
        }
      },

      _sendMesRender () { // 在当前channel上发送消息时，需要回滚掉底部
        if (!this.channelSendState) {
          return false
        }
        this.messagesWithUserInfo = this.curMessage
        scrollBottom($('.scroll-con')[0])
        this.toggleChannelSend()
        return true
      },

      _firstMesRender () {
        if (!this.isloadFirst) {
          return false
        }
        $('.pannel-chat-body').css({visibility: 'hidden'}) // 防止滚动跳动闪屏
        this.messagesWithUserInfo = this.curMessage

        this.$nextTick(() => {
          this.$broadcast('endLoading')
          scrollBottom($('.scroll-con')[0])
          $('.pannel-chat-body').css({visibility: 'visible'})
          this.startLoadMore = true
        })
        return true
      },

      _moreMesRender () {
        if (this.isloadFirst) {
          return false
        }
        $('.pannel-chat-body').css({visibility: 'hidden'})   // 防止滚动跳动闪屏
        let preLast = $('.module-message-item').eq(0)
        this.messagesWithUserInfo = this.curMessage

        this.$nextTick(() => {
          this.$broadcast('endLoading')
          $('.scroll-con').scrollTop(preLast.position().top) // 还原位置
          $('.pannel-chat-body').css({visibility: 'visible'})
        })
        return true
      },

      _getNewUserIds (curMessage) {
        let userIds = []
        for (let item of curMessage) {
          if (item.message_type !== constants.MESSAGE_PLAIN || array.findIndex(this.usersInfoStorage, {id: item.creator}) !== -1) {
            continue
          }
          userIds.push(item.creator)
        }
        userIds = array.uniq(userIds)
        return userIds
      },

      loadMoreMessage (scrollEle, contentEle) {
        if (!this.startLoadMore) {
          return
        }
        let seq = this.seqMessages[this.curChannelId]
        seq && this.getMessages(this.curChannelId, seq)

        this.isloadFirst = false
        this.$broadcast('endLoading')
      }
    }
  }
</script>
