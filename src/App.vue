<template>
  <div id="app">
    <c-dialog ref="loginDialog" title="请输入你的昵称" confirmBtn="开始聊天" @confirm="login">
      <input class="nickname" v-model="nickname" type="text" placeholder="请输入你的昵称">
    </c-dialog>
    <div class="web-im dis-flex">
      <div class="left">
        <div class="aside content">
          <div class="header">联系人</div>
          <div class="user" @click="triggerGroup">
            群1
            <span class="tips-num">{{getMsgNum()}}</span>
          </div>
          <div
            class="user"
            @click="triggerPersonal(item)"
            v-for="(item, index) in users"
            :key="index"
          >
            <div v-if="item.uid!=uid">
              <span>{{item.nickname}}</span>
              <span class="tips-num">{{getMsgNum(item)}}</span>
            </div>
          </div>
        </div>
      </div>
      <div class="right">
        <div class="header im-title">{{title}}</div>
        <div class="body im-record">
          <div class="ul">
            <div
              class="li"
              :class="{user: item.uid == uid}"
              v-for="(item, index) in currentMessage"
              :key="index"
            >
              <template v-if="item.type===1">
                <p class="join-tips">{{item.msg}}</p>
              </template>
              <template v-else>
                <p class="message-date">
                  <span class="m-nickname">{{item.nickname}}</span>
                  {{item.date}}
                </p>
                <p class="message-box">{{item.msg}}</p>
              </template>
            </div>
          </div>
        </div>
        <div class="footer im-input dis-flex">
          <input type="text" v-model="msg" placeholder="请输入内容">
          <button @click="send">发送</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import moment from "moment"
import CDialog from "@/components/dialog";

export default {
  name: "app",
  components: {
    CDialog
  },
  data() {
    return {
      uid: "",
      nickname: "",
      socket: "",
      msg: "",
      messageList: [],
      users: [],
      isShow: false,
      bridge: [],
      title: '聊天室'
    };
  },
  mounted() {
    let vm = this;
    let user = localStorage.getItem("WEB_IM_USER");
    user = (user && JSON.parse(user)) || {};
    vm.uid = user.uid;
    vm.nickname = user.nickname;

    if (!vm.uid) {  
      vm.$refs.loginDialog.show();
      // this.isShow = true;
    } else {
      vm.conWebSocket();
    }
    document.onkeydown = function(event) {
      var e = event || window.event;
      if (e && e.keyCode == 13) {
        //回车键的键值为13
        vm.send();
      }
    };
  },
  computed: {
    currentMessage() {
      let vm = this;
      let data = vm.messageList.filter(item=>{
        return item.bridge.sort().join(',') == vm.bridge.sort().join(',')
      })
      data.map(item=>{
        item.status = 0
        return item;
      })
      return data;
    },
  },
  methods: {
    // 切换到群聊
    triggerGroup() {
      this.bridge = [];
      this.title = '群聊';
    },
    // 切到具体个人
    triggerPersonal(item) {
      if(this.uid === item.uid){
        return;
      }
      // 将当前用户uid，和需要对话的uid放入bridge
      this.bridge = [this.uid, item.uid];
      this.title = '和' + item.nickname + '聊天';
    },
    send() {
      if (!this.msg) {
        return;
      }
      this.sendMessage(2, this.msg);
    },
    /* type 1为新用户，2为已经在聊天室的用户 */
    sendMessage(type, msg) {
      console.log(this.socket);
      this.socket.send(
        JSON.stringify({
          uid: this.uid,
          type: type,
          nickname: this.nickname,
          msg: msg,
          bridge: this.bridge
        })
      );
      this.msg = "";
    },
    getMsgNum(user){
      if(!user){
        return this.messageList.filter(item=>{
          return !item.bridge.length && item.status === 1
        }).length
      }
      return this.messageList.filter(item=>{
        return item.bridge.length && item.uid === user.uid && item.status === 1
      }).length
    },    
    conWebSocket() {
      let vm = this;
      if (window.WebSocket) {
        vm.socket = new WebSocket("ws://localhost:8001");
        let socket = vm.socket;
        socket.onopen = function(e) {
          console.log("连接服务器成功");
          if (!vm.uid) {
            // 生成新的用户id,并存入localStorage
            vm.uid = "web_im_" + moment().valueOf();
            localStorage.setItem(
              "WEB_IM_USER",
              JSON.stringify({
                uid: vm.uid,
                nickname: vm.nickname
              })
            );
            vm.sendMessage(1);
          }
        };
        socket.onclose = function(e) {
          console.log("服务器关闭");
        };
        socket.onerror = function() {
          console.log("连接出错");
        };
        // 接收服务器的消息
        socket.onmessage = function(e) {
          console.log("接收服务器的消息");
          let message = JSON.parse(e.data);
          vm.messageList.push(message);
          if(message.users.length) {
            vm.users = message.users;
          }
        };
      }
    },
    login() {
      this.$refs.loginDialog.hide();
      this.conWebSocket();
    }
  }
};
</script>

<style lang="stylus">
@import './app.styl'
.web-im
  .left
    box-shadow: 1px 1px 1px #eee
    .tips-num
      display: block
      margin-left: 20px
      font-size: 10px
      width: 10px
      height: 10px
      border-radius: 50%
  .right
    display: flex
    flex-direction: column
    .header
      text-align: center
    .body
      flex: 1
      padding-bottom: 50px
    .footer
      position: fixed
      bottom: 0px
      right: 0px
      left: 220px
      display: flex
      input
        flex: 1
        padding: 0 20px
        font-size: 24px
      button
        width: 200px
        color: #fff
        font-size: 24px
        background: #00b7a3
</style>
