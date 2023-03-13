<template>
  <div>
    <ul>
      <template v-for="(userinfo) in state.userlist">
        <li :key="userinfo.id" v-if="userinfo.username === state.username">
          {{userinfo.username}}</li>
        <li :key="userinfo.id" v-else>
          <a href="javascript:;" @click="selectUser(userinfo)">{{userinfo.username}}</a>
        </li>
      </template>
    </ul>

    <div v-if="state.targetUser">
      <h3>
        {{state.targetUser.username}}
      </h3>

      <input v-model="state.msgText" type="text" placeholder="Input some...">
      <button @click="sendMessage">Send</button>
    </div>

    <div>
      <ul>
        <li v-for="(item,index) in messageList" :key="index">
          <p>{{item.fromUsername}}: {{new Date(item.dateTime)}}</p>
          <p>{{item.msg}}</p>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { useRouter, useRoute } from 'vue-router';
import { io } from 'socket.io-client';
import { computed, reactive } from 'vue';

const $router = useRouter()
const $route = useRoute()
console.log($router.currentRoute.value.query)
console.log('useRoute: ', $route)
const state = reactive({
  username: $route.query.username || '',
  userlist: [],
  targetUser: null,
  msgText: "",
  // 前端存储用所有用户消息列表的容器
  messageBox: {}
});

const messageList = computed(() => {
  // 如果消息盒子中的用户名不存在，则默认赋值空数组
  // !state.messageBox[state.username] && (state.messageBox[state.username] = []);
  // 当前用户存在，并且已经选中了目标用户
  return (state.messageBox[state.username] && state.targetUser) ?
    state.messageBox[state.username].filter(item => {
      return item.fromUsername === state.targetUser.username ||
        item.toUsername === state.targetUser.username
    }) : []
})

const socket = io('http://localhost:3000', {
  query: {
    username: state.username
  }
});

const selectUser = (userinfo) => {
  console.log('当前选中的userinfo: ', userinfo);
  state.targetUser = userinfo;
}

const sendMessage = () => {
  if (!state.msgText.length) return;
  /* 
    数据包格式: 
    { 
      fromuserName: xxx,
      touserName: xxx,
      dateTime: xxx,
      msg: xxx
    }
  */
  appendMessage({
    fromUsername: state.username,
    toUsername: state.targetUser.username,
    msg: state.msgText,
    dateTime: new Date().getTime()
  });

  // 给服务器发送自定义事件
  socket.emit('send', {
    fromUsername: state.username,
    targetId: state.targetUser.id,
    msg: state.msgText
  })
}

socket.on('online', data => {
  // 接收服务器发送的用户列表数据
  console.log('接收socket传递的数据 >>>> ', data);
  state.userlist = data;
})

socket.on('receive', data => {
  console.log('receive data: ', data);
  appendMessage(data)
})

socket.on('error', err => {
  console.log('socket 出错啦!!! ', err);
})

function appendMessage(data) {
  !state.messageBox[state.username] && (state.messageBox[state.username] = []);
  state.messageBox[state.username].push(data);
}

</script>

<style lang="scss">
</style>