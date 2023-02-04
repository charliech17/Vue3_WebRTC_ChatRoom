<template>
  <div>{{ state.isconnect ? '已連接' : '未連接' }}</div>
  <p>你的連接ID: {{ peerId }}</p>
  <input type="text" v-model="state.inputConnectID">
  <div>
    <button @click="() => connFunction(state.inputConnectID)" :disabled="!state.inputConnectID">連接視訊</button>
  </div>
  <div>
    <video id="streamVideo" autoplay muted playsinline></video>
  </div>
  <div>
    <textarea cols="30" rows="10" @input="handleTextareaInput" v-model="state.textAreaValue"></textarea>
  </div>
</template>

<script setup lang="ts">
import { Peer } from "peerjs";
import { reactive,ref, VideoHTMLAttributes } from 'vue'

const state:{isconnect: boolean,inputConnectID: string,textAreaValue: string} = reactive({
  isconnect: false,
  inputConnectID: '',
  textAreaValue: '',
})

const peer = new Peer();
const peerId = ref('')

// 1 接收連接
peer.on('open',(myId)=> {
    peerId.value = myId
})

peer.on("connection", (conn) => {
    conn.on("data", (data) => {
      console.log('data!!!',data)
      // @ts-ignore
      state.textAreaValue = data
    });
    conn.on("open", () => {
      conn.send(state.textAreaValue);
    });
});

peer.on("call", (call) => {
  console.log('call!!!',call)
  // @ts-ignore
  const getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia; 
	getUserMedia(
		{ video: true, audio: true },
    (stream: MediaStream) => {
			call.answer(stream); // Answer the call with an A/V stream.
			call.on("stream", (remoteStream) => {
        console.log(remoteStream)
				const video = document.querySelector("#streamVideo") as HTMLVideoElement
        video.srcObject = remoteStream;
        video.onloadedmetadata = () => {
          video.play();
        };
			})
    },
    (err: Error) => {
      console.error("Failed to get local stream", err);
    }
	)
})

// 2 始連接
const connFunction = (remoteID: string) => {
  console.log(state.inputConnectID)
  const conn = peer.connect(remoteID);
  conn.on("open", () => {
    conn.send("hi!");
  });
  conn.on("data", (data) => {
      // Will print 'hi!'
      console.log(data);
  });

  const constraints = {
    audio: true,
    video: {  facingMode: "user", width: 1280, height: 720 }
  };

  // @ts-ignore
  const getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia;
  getUserMedia(
		{ video: true, audio: true },
    (stream: MediaStream) => {
			console.log('start Stream!!!')
      const call = peer.call(state.inputConnectID, stream);
      call.on("stream", (remoteStream) => {
        // Show stream in some <video> element.
          const video = document.querySelector('#streamVideo') as HTMLVideoElement
          video.srcObject = remoteStream;
          video.onloadedmetadata = () => {
            video.play();
          };
      });
    },
    (err: Error) => {
      console.error("Failed to get local stream", err);
    }
	)
}

let debounce: ReturnType<typeof setTimeout>
const handleTextareaInput = () => {
  clearTimeout(debounce)
  debounce = setTimeout(()=> {
    const conn = peer.connect(state.inputConnectID);
    conn.on("open", () => {
      console.log('sendValue!!!',state.textAreaValue)
      conn.send(state.textAreaValue);
    });
  },500)
}

</script>

<style scoped>
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
</style>
