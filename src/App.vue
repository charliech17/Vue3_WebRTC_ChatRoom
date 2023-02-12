<template>
  <div>
    <img  :src="state.isconnect ? getImageUrl('connect.png')
                                : getImageUrl('disconnect.png')" 
              style="height: 15px; display: inline; margin-right: 4px;"
    />
    {{ state.isconnect ? '已連接' : '未連接' }}
  </div>
  <p>你的連接ID: {{ peerId }}</p>
  <input type="text" v-model="state.inputConnectID">
  <div>
    <!-- ***按鈕區塊一*** -->
    <div class="mTop-8">
      <!-- **連接按鈕** -->
      <button @click="() => connFunction(state.inputConnectID)" :disabled="!state.inputConnectID">
        <span>連接視訊</span>
      </button>
       <!-- **音訊開關按鈕** -->
      <button @click="closeMuted" :disabled="!state.isconnect">
        <img  :src="videoMuted  ? getImageUrl('volumeOff.png')
                                : getImageUrl('volumeOn.png')" 
              style="height: 15px; display: inline; margin-right: 4px;"
        />
        {{videoMuted ? "連接音訊":"斷開音訊"}}
      </button>
    </div>
    <!-- ***按鈕區塊二*** -->
    <div class="mTop-8">
      <!-- **視訊開關按鈕** -->
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
  <div class="mTop-8">
    <textarea cols="30" rows="10" @input="handleTextareaInput" v-model="state.textAreaValue"></textarea>
  </div>
</template>

<script setup lang="ts">
import { Peer } from "peerjs";
import { reactive,ref} from 'vue'

const state:{isconnect: boolean,inputConnectID: string,textAreaValue: string,isError: boolean,isMutedDisable: boolean,myMediaStream: MediaStream | null, isShowCamera: boolean, isShowSound: boolean} = reactive({
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
  state.isMutedDisable = true
  countErrorTime++
  reConnectServer()
  if(countErrorTime > 4) {
    countErrorTime = 0
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
      background-color: wheat;
    }
  }
}
</style>
