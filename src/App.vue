<template>
  <h1>試用ChatApp</h1>
  <div class="status-bg mx-auto">
    <div class="d-flex align-items-center">
      <h2 class="mr-8 width-180px flex-shrink-0 txt-align pl-8">你的連線狀態:</h2>
      <div class="flex-1">
        <img  :src="state.isconnect ? getImageUrl('connect.png')
                                    : getImageUrl('disconnect.png')" 
                  style="height: 15px; display: inline; margin-right: 4px;"
        />
        <span>{{ state.isconnect ? '已連接' : '未連接' }}</span>
      </div>
    </div>
    <div class="d-flex align-items-center">
      <h2 class="mr-8 width-180px flex-shrink-0 txt-align pl-8">你的連接碼:</h2>
      <div class="flex-1 d-flex mr-4 align-items-center justify-content-center" > 
        <div v-if="!peerId" class="text-center">取得連接碼中...</div>
        <div v-if="peerId" class="mobile-flex-1">{{ peerId }}</div>
        <div v-if="peerId" class="copy_img_style" @click="copyText"></div>
      </div>
    </div>
  </div>
  <div class="d-flex mt-20 justify-content-center align-items-center">
    <h4 class="mr-8">輸入別人的連接碼</h4>
    <input 
      class="ipt_style mr-8" 
      type="text" 
      placeholder="輸入別人的連結碼，或提供連接碼給別人"
      v-model="state.inputConnectID"
    >
    <button @click="() => connFunction(state.inputConnectID)" :disabled="!state.inputConnectID">
      <span>連接</span>
    </button>
  </div>
  <main 
    class="main_bg opacity_disable"
    :class="{'opacity_normal': state.isconnect}"
  >
    <h2>視訊介面</h2>
    <div class="control_part mTop-8">
        <!-- **音訊開關按鈕** -->
      <button @click="closeMuted" :disabled="!state.isconnect">
        <img  :src="videoMuted  ? getImageUrl('volumeOff.png')
                                : getImageUrl('volumeOn.png')" 
              style="height: 15px; display: inline; margin-right: 4px;"
        />
        {{videoMuted ? "連接音訊":"斷開音訊"}}
      </button>
      <button @click="() => toggleOutput('video')" :disabled="!state.isconnect">
        <img  :src="state.isShowCamera  ? getImageUrl('showVideo.png')
                                        : getImageUrl('noVideo.png')" 
              style="height: 15px; display: inline; margin-right: 4px;"
        />
        {{state.isShowCamera ? '視訊關閉': '視訊開啟'}}
      </button>
      <!-- **麥克風開關按鈕** -->
      <button @click="() => toggleOutput('audio')" :disabled="!state.isconnect">
        <img :src="state.isShowSound  ? getImageUrl('unmute.png') 
                                      : getImageUrl('mute.png')" 
              style="height: 15px;display: inline;margin-right: 4px;"
        />
        {{state.isShowSound ? '麥克風關': '麥克風開'}}
      </button>
    </div>
    <div class="videoSection">
      <div>
        <h3>別人的影像</h3>
        <video poster="../assets/picture/posterImage.jpg" id="otherVideo" autoplay muted playsinline></video>
      </div>
      <div>
        <h3>您的影像</h3>
        <video poster="../assets/picture/posterImage.jpg" id="myVideo" autoplay muted playsinline></video>
      </div>
    </div>
  </main>
  <div 
    class="mTop-8 opacity_disable" 
    :class="{'opacity_normal': state.isconnect}"
  >
    <h2>即時文字</h2>
    <textarea 
      :disabled="!state.isconnect"
      cols="30" 
      rows="10" 
      @input="handleTextareaInput" 
      v-model="state.textAreaValue"
    ></textarea>
  </div>
</template>

<script setup lang="ts">
import { Peer } from "peerjs";
import { reactive,ref} from 'vue'

const state:{
  isconnect: boolean,
  inputConnectID: string,
  textAreaValue: string,
  isError: boolean,
  isMutedDisable: boolean,
  myMediaStream: MediaStream | null, 
  isShowCamera: boolean, 
  isShowSound: boolean
} = reactive({
  isconnect: false,
  inputConnectID: '',
  textAreaValue: '',
  isError: false,
  isMutedDisable: true,
  myMediaStream: null,
  isShowCamera: true,
  isShowSound: true
})

// let peer = new Peer();
let peer = new Peer()
const peerId = ref('')
initEventListenr()



// 2 始連接
const connFunction = (remoteID: string) => {
  console.log(state.inputConnectID,peer)
  if(state.inputConnectID == peerId.value) {
    alert("請勿輸入自己的連接碼")
    return
  } 
  const conn = peer.connect(remoteID)
  conn.on("open", () => {
    state.isconnect = true
    conn.send("hi!");
  });
  conn.on("data", (data) => {
      // Will print 'hi!'
      console.log(data);
  });

  const constraints = {
    audio: true,
    video: {  facingMode: "user" }
  };

  // @ts-ignore
  const getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia;
  getUserMedia(
		constraints,
    (localstream: MediaStream) => {
			console.log('start Stream!!!')
      setVideoPlay('#myVideo',localstream)
      const call = peer.call(state.inputConnectID, localstream);
      call.on("stream", (remoteStream) => {
        // Show stream in some <video> element.
        setVideoPlay('#otherVideo',remoteStream)
      });
    },
    (err: Error) => {
      state.isMutedDisable = true
      console.error("Failed to get local stream", err);
    }
	)
}

let debounce: ReturnType<typeof setTimeout>
const handleTextareaInput = () => {
  state.isconnect = true
  clearTimeout(debounce)
  debounce = setTimeout(()=> {
    const conn = peer.connect(state.inputConnectID);
    console.log(peer)
    conn.on("open", () => {
      console.log('sendValue!!!',state.textAreaValue)
      conn.send(state.textAreaValue);
    });
  },500)
}

function stopStreamedVideo(videoElem: HTMLVideoElement) {
  const stream = videoElem.srcObject;
  if(!stream) return
  //@ts-ignore
  const tracks = stream.getTracks();
  //@ts-ignore
  tracks.forEach((track) => {
    track.stop();
  });

  videoElem.srcObject = null;
}

function reInitPeer(initId?: string) {
  peer = initId ? new Peer(initId) : new Peer()
}

let countErrorTime = 0
function initEventListenr() {
  // 1 接收連接
  peer.on('open',(myId)=> {
      peerId.value = myId
  })

  peer.on("connection", (conn) => {
      conn.on("data", (data) => {
        console.log('data!!!',data)
        state.isconnect = true
        // @ts-ignore
        state.textAreaValue = data
      });
      conn.on("open", () => {
        conn.send(state.textAreaValue);
      });
  });

  peer.on("call", (call) => {
    state.isconnect = true
    // @ts-ignore
    const getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia; 
    getUserMedia(
      { video: true, audio: true },
      (localStream: MediaStream) => {
        setVideoPlay('#myVideo',localStream)
        call.answer(localStream); // Answer the call with an A/V stream.
        call.on("stream", (remoteStream) => {
          console.log(remoteStream)
          setVideoPlay('#otherVideo',remoteStream)
        })
      },
      (err: Error) => {
        state.isMutedDisable = true
        console.error("Failed to get local stream", err);
      }
    )
  })

  peer.on('close', function() { 
    stopStreaming()
 });

 peer.on('disconnected', function() { 
    stopStreaming()
 })

 peer.on('error',(error)=> {
  // console.log(peer,error,error.name,error.toString())
  state.isMutedDisable = true
  countErrorTime++
  if(error.toString().includes('Lost connection to server')) {
    console.log('error Lost connection to server!!!')
    reConnectServer()
  }

  if(countErrorTime > 4) {
    countErrorTime = 0
    const myVideo = document.querySelector('#myVideo') as HTMLVideoElement
    stopStreamedVideo(myVideo)
    alert('錯誤發生：'+error)
  }
 })
}

function stopStreaming() {
    state.isMutedDisable = true
    state.isconnect = false
    const otherVideo = document.querySelector('#otherVideo') as HTMLVideoElement
    stopStreamedVideo(otherVideo)
}

function reConnectServer() {
  reInitPeer(peerId.value)
  initEventListenr()
}

let videoMuted = ref(true)
const closeMuted = () => {
  const otherVideo = document.querySelector('#otherVideo') as HTMLVideoElement
  otherVideo.muted = !otherVideo.muted
  videoMuted.value  = otherVideo.muted
}

const setVideoPlay = (id: string,streamData: MediaStream) => {
  const video = document.querySelector(id) as HTMLVideoElement
  video.srcObject = streamData;
  video.onloadedmetadata = () => {
    video.play();
    if(id === "#myVideo") {
      state.myMediaStream = streamData
    } else if(id === '#otherVideo') {
      state.isMutedDisable = false
    }
  };
}

const toggleOutput = (inputType: 'video' | 'audio') => {
  if(!state.myMediaStream) return 
  const myStream = state.myMediaStream
  const videoTrack = myStream.getTracks().find(track => track.kind === inputType)
  console.log('vdTrack',videoTrack,myStream)
  
  if(videoTrack?.enabled) {
    videoTrack.enabled = false
    console.log(videoTrack,videoTrack.enabled)
    inputType === 'video' ? state.isShowCamera = false : state.isShowSound = false
  } else {
    if(!videoTrack) return
    videoTrack.enabled = true
    inputType === 'video' ? state.isShowCamera = true : state.isShowSound = true
  }
}


function getImageUrl(imageName: string) {
  return new URL(`../assets/image/${imageName}`, import.meta.url).href
}

function copyText() {
  const text = peerId.value
  // 判斷瀏覽器支援
  if (!navigator.clipboard) {
      alert("瀏覽器不支援 Clipboard API")
      // 這裡可以改用 document.execCommand('copy') 的方法
  }

  // 非同步複製至剪貼簿
  let resolve = () => { 
      console.log('已複製連接碼'); 
      alert('已複製連接碼')
  }
  let reject = (err: Error) => { 
      console.error('複製失敗' + err.toString() ); 
      alert('複製失敗')
  }
  navigator.clipboard.writeText(text).then(resolve, reject);
}

</script>

<style>
#app {
  padding: 16px;
  overflow-x: hidden;
}

</style>

<style lang="scss" scoped>
.mTop-8 {
  margin-top: 16px;
}
.mr-4 {
  margin-right: 8px;
}
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}

.videoSection {
  display: flex;
  flex-wrap: wrap;
  width: 100%;
  gap: 8px;
  
  div{
    flex: 1;

    video {
      width: 100%;
    }
  }
}
.opacity_disable {
  opacity: 0.3;
}
.opacity_normal {
  opacity: 1;
}

.mobile-flex-1{
  flex: 1;
  @media(min-width: 768px) {
    flex: auto;
  }
}

.copy_img_style{
  height: 20px;
  width: 20px;
  cursor: pointer;
  background-image: url("/copy.png");
  background-size: contain;
  background-repeat: no-repeat;
}
</style>
