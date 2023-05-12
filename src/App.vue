<style lang="scss" scoped>
.my-chat-room {
    display: flex;
    flex-direction: column;

    .connect {
        display: flex;
    }

    .message {
        display: flex;
    }

    .mine {
        text-align: left;
    }
    .remote {
        text-align: right;
    }

    .video-box {
        width: 100%;
        min-height: 100vw;
        position: relative;
    }

    .video-remote {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        z-index: 9;
    }

    .video-local {
        position: absolute;
        right: 0;
        bottom: 0;
        width: 40%;
        z-index: 10;
    }
}
</style>

<template>
    <div class="my-chat-room">
        <span>My peer ID is: {{ state.myPeerId }}</span>
        <n-button @click="copyTextToClipboard(`https://sjw.xfbc.top/?id=${state.myPeerId}`)"
            >分享</n-button
        >
        <div class="connect">
            <n-input v-model:value="state.destPeerId" type="text" placeholder="请输入id" />
            <n-button @click="connect">连接</n-button>
        </div>
        <div class="message">
            <n-input v-model:value="state.msgInfo" type="text" placeholder="信息" />
            <n-button @click="send">发送</n-button>
        </div>
        <div>
            <p :class="`${item.type}`" v-for="item in state.msg">{{ item.data }}</p>
        </div>
        <div class="video-box">
            <video
                class="videos video-local"
                controls
                autoPlay
                muted
                ref="localCameraVideo"
            ></video>
            <video
                class="videos video-remote"
                controls
                autoPlay
                muted
                ref="remoteCameraVideo"
            ></video>
        </div>
    </div>
</template>

<script setup>
import Peer from 'peerjs';
import { nextTick, onMounted, reactive, ref } from 'vue';
const peer = new Peer()
const state = reactive({
        myPeerId: '',
        myPeerConnect: '',
        currentCall: '',
        msgInfo: '',
        msg: [],
        stream: '',
        destPeerId: '',
    }),
    localCameraVideo = ref(),
    remoteCameraVideo = ref()

const createChatRoom = async () => {
    peer.on('open', (id) => {
        console.log('My peer ID is: ' + id)
        state.myPeerId = id
    })

    // 获取本地流-调用摄像头
    state.stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true })
    localCameraVideo.value.srcObject = state.stream
    localCameraVideo.value.play()

    peer.on('connection', (connection) => {
        // 接受对方传来的数据
        connection.on('data', (data) => {
            state.msg.push({ id: state.msg.length + 1, type: 'remote', data })
        })
    })

    peer.on('call', async (call) => {
        if (window.confirm(`是否接受 ${call.peer}?`)) {
            // 响应
            call.answer(state.stream)

            // 监听视频流，并更新到 remoteCameraVideo 上
            call.on('stream', (stream) => {
                remoteCameraVideo.value.srcObject = stream
                remoteCameraVideo.value.play()
            })

            state.currentCall = call
        } else {
            call.close()
        }
    })
}

const endCall = () => {
    if (state.currentCall) {
        state.currentCall.close()
    }
}

const connect = async () => {
    state.myPeerConnect = peer.connect(state.destPeerId)
    state.myPeerConnect.on('open', () => {
        alert('连接成功')
    })

    // 获取本地流-分享屏幕
    // state.stream = await navigator.mediaDevices.getDisplayMedia({ video: true })

    // 多媒体传输
    const call = peer.call(state.destPeerId, state.stream)
    call.on('stream', (stream) => {
        remoteCameraVideo.value.srcObject = stream
        remoteCameraVideo.value.play()
    })
    call.on('error', (err) => {
        console.error(err)
    })
    call.on('close', () => {
        endCall()
    })

    state.currentCall = call
}

const send = () => {
    if (!state.myPeerConnect) {
        alert('还未建立链接')
        return false
    }
    state.msg.push({ id: state.msg.length + 1, type: 'mine', data: state.msgInfo })
    state.myPeerConnect.send(state.msgInfo)
}

const copyTextToClipboard = async (text) => {
    if (navigator.clipboard && window.isSecureContext) {
        // navigator clipboard 向剪贴板写文本
        await navigator.clipboard.writeText(text)
        alert('已复制到剪切板，请发送给需视频对象')
    } else {
        // 创建text area
        let textArea = document.createElement('textarea')
        textArea.value = text
        // 使text area不在viewport，同时设置不可见
        textArea.style.position = 'absolute'
        textArea.style.opacity = 0
        textArea.style.left = '-999999px'
        textArea.style.top = '-999999px'
        document.body.appendChild(textArea)
        textArea.focus()
        textArea.select()
        return new Promise((res, rej) => {
            // 执行复制命令并移除文本框
            document.execCommand('copy') ? res() : rej()
            textArea.remove()
        })
    }
}

onMounted(async () => {
    if (window.location.search.replace(/\?id\=/g, '')) {
        state.destPeerId = window.location.search.replace(/\?id\=/g, '')
    }
    await nextTick()
    await createChatRoom()
})
</script>
