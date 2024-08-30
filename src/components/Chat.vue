<template>

  <div class="common-layout">
    <el-container>
      <el-aside width="500px">

        <div class="greetings">
          <h3>聊天测试</h3>
        </div>
        <el-space direction="horizontal" alignment="start" :size="10">
          <el-button type="success" @click="connectWs()">建立ws连接</el-button>
        </el-space>
        <div style="margin-top: 20px;"></div>
        <!-- 表单  -->
        <el-form :model="form" label-width="auto" style="max-width: 600px">
          <el-form-item label="发送者ID(fromUserId)">
            <el-input v-model="form.fromUserId" />
          </el-form-item>

          <el-form-item label="接收者ID(toUserId)">
            <el-input v-model="form.toUserId" />
          </el-form-item>

          <el-form-item label="消息内容(content)">
            <el-input v-model="form.content" type="textarea" />
          </el-form-item>

          <el-form-item label="发送时间(sendTime)">
            <el-col :span="17">
              <el-date-picker
                  v-model="form.sendTime"
                  type="datetime"
                  placeholder="格式(YYYY-MM-DD HH:mm:ss)"
                  style="width: 100%"
                  value-format="YYYY-MM-DD HH:mm:ss"
              />
            </el-col>
          </el-form-item>

          <el-form-item label="消息类型(type)">
            <el-radio-group v-model="form.type">
              <el-radio value=0>0登录并建立连接</el-radio>
              <el-radio value=1>1文本消息</el-radio>
              <el-radio value=2>2图片消息</el-radio>
              <el-radio value=3>3文件消息</el-radio>
              <el-radio value=4>4语言消息</el-radio>
              <el-radio value=5>5视频消息</el-radio>
              <el-radio value=6>6推送消息</el-radio>
              <el-radio value=7>7系统消息</el-radio>
              <el-radio value=8>8心跳检测</el-radio>
              <el-radio value=9>9撤回类型</el-radio>
              <el-radio value=10>10表情包</el-radio>
              <el-radio value=11>11名片</el-radio>
            </el-radio-group>
          </el-form-item>

          <el-form-item label="消息状态(status)">
            <el-radio-group v-model="form.status">
              <el-radio value=0>0已发送</el-radio>
              <el-radio value=1>1已读</el-radio>
              <el-radio value=2>2发送失败</el-radio>
              <el-radio value=3>3已撤回</el-radio>
              <el-radio value=4>4待发送</el-radio>
              <el-radio value=5>5已读的语音</el-radio>
            </el-radio-group>
          </el-form-item>

          <el-form-item label="引用内容">
            <el-input v-model="form.quote" />
          </el-form-item>

          <el-form-item label="扩展字段内容">
            <el-input v-model="form.extend" />
          </el-form-item>

          <el-form-item label="消息唯一标识符(id)">
            <el-input v-model="form.id" />
          </el-form-item>

          <el-form-item>
            <el-button type="primary" @click="sendMsg()">发送自定义消息</el-button>
            <el-button type="info" @click="clearServerMessage()">清空服务器消息</el-button>
            <el-button type="danger" @click="close()">断开连接</el-button>
          </el-form-item>
        </el-form>
      </el-aside>

      <el-main>
        <!-- 显示服务器消息列表 -->
        <el-table :data="serverMessages" height="650" show-overflow-tooltip border stripe style="width: 100%; margin-top: 20px;">
          <el-table-column prop="fromUserId" label="发送者ID"  />
          <el-table-column prop="toUserId" label="接收者ID"  />
          <el-table-column prop="content" label="内容" min-width="200" />
          <el-table-column prop="sendTime" label="发送时间" width="150" />
          <el-table-column prop="type" label="类型" width="50" />
          <el-table-column prop="status" label="状态" width="50" />
          <el-table-column prop="quote" label="引用内容" width="50" />
          <el-table-column prop="extend" label="扩展内容" width="50" />
          <el-table-column prop="id" label="ID"/>
        </el-table>
      </el-main>
    </el-container>
  </div>


</template>

<script setup lang="ts">
import { load } from "protobufjs";
import {onMounted, reactive, ref} from 'vue'

// const wsUrl = ref("ws://192.168.0.244:16851/api/user/ws")
const wsUrl = ref("ws://127.0.0.1:16851/api/user/ws")

// 登录成功登录后获取到的sessionId
const sessionId = ref('')
let socket: WebSocket | null = null;
let ChatMessage: any = null;

// 服务器消息列表
let serverMessages = ref<ServerMessage[]>([]);
// 消息数据结构
interface ServerMessage {
  fromUserId: string;
  toUserId: string;
  content: string;
  sendTime: string;
  type: number;
  status: number;
  quote: string;
  extend: string;
  id: string;
}

let form = reactive({
  fromUserId: '',
  toUserId: '',
  content: '',
  sendTime: '',
  type: '',
  status: '',
  quote: '',
  extend: '',
  id: '',
})

onMounted(() => {
  // 加载proto文件
  load("/ChatMessage.proto", function (err, root) {
    if (err) throw err;
    if (!root) {
      alert("无法读取到proto文件");
      return;
    }
    ChatMessage = root.lookupType("ChatMessage");
  });

})
const sendMsg = () => {
  // 转换响应式对象为普通对象
  let customMessage = {
    fromUserId: form.fromUserId,
    toUserId: form.toUserId,
    content: form.content,
    sendTime: parseInt(form.sendTime), // 确保sendTime为整数
    type: parseInt(form.type), // 确保类型为整数
    status: parseInt(form.status), // 确保状态为整数
    quote: form.quote,
    extend: form.extend,
    id: form.id,
  };

  let message = ChatMessage.create(customMessage);

  console.log(`chatMessage = ${JSON.stringify(message)}`);

  // 编码为传输数据
  let buffer = ChatMessage.encode(message).finish();
  console.log(`编码后buffer = ${Array.prototype.toString.call(buffer)}`);

  // 发送编码后的数据
  if (socket == null) alert("未连接到服务器,请登录连接");
  socket!.send(buffer);
}
// 发送消息
function connectWs(){
  console.log("sessionId=" + sessionId.value)
  // 创建WebSocket连接
  socket = new WebSocket(wsUrl.value);
  // 连接打开事件
  socket.onopen = function (event) {
    console.log("WebSocket 已连接.");
  };
  // 接收到消息事件
  socket.onmessage = function (event) {
    console.log("收到服务端数据")
    handleMessage(event.data);
  };
  // 错误事件
  socket.onerror = function (event) {
    console.error("WebSocket 发生错误:", event);
  };
  // 连接关闭事件
  socket.onclose = function (event) {
    if (event.wasClean) {
      console.log(`Closed cleanly, code=${event.code}, reason=${event.reason}`);
    } else {
      console.error("WebSocket 连接已关闭");
    }
  };
  // // 开始登录
  // let message = ChatMessage.create({ content: "登录"
  // , type: 1, status: 0, id: 1, fromUserId: 1, toUserId: 2, sendTime: 1 });
  //
  // console.log(`登录的chatMessage = ${JSON.stringify(message)}`);
  // // 编码为传输数据
  // let buffer = ChatMessage.encode(message).finish();
  //
  // // 发送编码后的数据
  // socket!.send(buffer);
}

// 处理服务器传来的数据
// 异步处理消息
async function handleMessage(data: Blob) {
  try {
    console.log("收到服务端数据")
    const blob: Blob = data;
    const buffer = await blob.arrayBuffer(); // 等待 Promise 解析
    const byteArray = new Uint8Array(buffer);
    const message = ChatMessage.decode(byteArray);
    console.log(`服务端传来的消息: ${JSON.stringify(message)}`);

    // 添加到消息列表
    serverMessages.value.push({
      fromUserId: message.fromUserId,
      toUserId: message.toUserId,
      content: message.content,
      sendTime: message.sendTime,
      type: message.type,
      status: message.status,
      quote: message.quote,
      extend: message.extend,
      id: message.id,
    });

  } catch (e) {
    console.error("无法解析服务端消息:", e);
    alert("解析服务端消息错误:");
  }
}

// 清空服务器消息列表
const clearServerMessage = () => {
  serverMessages.value = [];
};

// 断开连接
function close(){
  if (socket) {
    socket.close();
    socket = null;
  } else {
    alert("WebSocket 并未连接");
  }
}

</script>



<style scoped>
</style>
