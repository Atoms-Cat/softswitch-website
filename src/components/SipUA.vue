<template>
  <div>
    <el-button @click="register">register</el-button>
    <br/>
    <el-input v-model="callSipAdder" placeholder="sip地址"></el-input>
    <br/>
    <el-button @click="call">call</el-button>
    <br/>
    <el-input v-model="msg" placeholder="信息"></el-input>
    <br/>
    <el-button @click="sendMsg">sendMsg</el-button>
    <br/>
    <el-button @click="getMediaState">getMediaState</el-button>
    <br/>
    <video id="video" ref="localVideoView" autoplay height="320px" width="420px"/>
    <video ref="remoteVideoStream" autoplay height="320px" width="420px"/>
    <audio ref="remoteAudioView" autoplay controls/>
  </div>
</template>

<script>
import JsSIP from "jssip";

export default {
  name: "SipUA",
  data() {
    return {
      ua: Object,
      callSipAdder: "sip:1001@atomscat.com",
      msg: "",
      incomingSession: Object,
      outgoingSession: Object,
      localStream: Object,
      constraints: {
        audio: true,
        video: true,
        mandatory: {
          maxWidth: 640,
          maxHeight: 360
        }
      }
    }
  },
  mounted() {
    this.register();
  },
  methods: {
    init() {
      //const socket = new JsSIP.WebSocketInterface('ws://192.168.10.109:5062');
      //const socket = new JsSIP.WebSocketInterface('wss://192.168.10.109:5063');
      const socket = new JsSIP.WebSocketInterface('ws://127.0.0.1:5061');
      const configuration = {
        sockets: [socket],
        uri: 'sip:1001@atomscat.com',
        contact_uri: 'sip:1001@192.168.10.140;transport=ws',
        authorization_user: '1001',
        password: '1234',
        display_name: '1001',
        // 计时器
        session_timers: false,
        // 注册会话超时时间
        register_expires: 30
      };
      this.ua = new JsSIP.UA(configuration);
      this.ua.on("registered", function (data) {
        console.info("registered: ", data.response.status_code, ",", data.response.reason_phrase);
      })
    },
    getLocalMedia(stream) {
      console.info('Received local media stream', stream);
      const video = this.$refs.localVideoView;
      video.srcObject = stream;
      video.onloadedmetadata = function (e) {
        video.play();
        console.log(e);
      };
      this.localStream = stream;
    },
    initMedia() {
      navigator.mediaDevices.getUserMedia(this.constraints).then(this.getLocalMedia).catch(function (err) {
        console.error(err.name + ": " + err.message);
      });
    },
    //
    getMediaState() {
      // const mediaRecorder = new MediaRecorder(this.localStream);
      console.log(this.localStream)
    },
    // 接电话
    answer() {
      this.incomingSession.answer({
        mediaConstraints: {
          audio: true,
          video: true
        },
        // rtcOfferConstraints: {'offerToReceiveAudio': true, 'offerToReceiveVideo': false},
        // sessionTimersExpires: 3600 * 24,
      });
      this.addRemoteStream();
    },
    //  add remote stream
    addRemoteStream() {
      console.log(this.incomingSession)
      this.incomingSession.connection.addEventListener('onaddstream', (event) => {
        console.log('onaddstream', event);
        this.setRemoteStream(event);
      });
    },
    setRemoteStream(event) {
      const video = this.$refs.remoteVideoStream;
      const audio = this.$refs.remoteAudioView;
      video.srcObject = event.stream;
      video.onloadedmetadata = function (e) {
        video.play();
        console.log(e)
      };
      audio.srcObject = event.stream;
      audio.onloadstart = (e) => {
        audio.play();
        console.log(e);
      };
      audio.onerror = () => {
        alert('录音加载失败...');
      };
    },
    register() {
      this.init();
      this.initMedia();
      this.ua.registrator().setExtraHeaders([
        'X-Foo: bar'
      ]);
      const that = this;
      // 监听通话
      this.ua.on("newRTCSession", function (data) {
        console.log(data);
        if (data.originator === 'remote') { //incoming call
          console.info("incomingSession, answer the call");
          that.incomingSession = data.session;
          that.$confirm('是否接听?', '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning'
          }).then(() => {
            // 接电话
            that.answer();
          }).catch(() => {
            //
            that.ua.terminateSessions();
          });
        } else {
          console.info("outgoingSession");
          that.outgoingSession = data.session;
          that.outgoingSession.on('connecting', function (data) {
            console.info(data);
          });
        }
      });

      // 监听短信
      this.ua.on("newMessage", function (data) {
        console.log(data);
        if (data.originator === 'local') {
          console.info('onNewMessage , OutgoingRequest - ', data.request);
        } else {
          console.info('onNewMessage , IncomingRequest - ', data.request);
        }
      });

      this.ua.on("sipEvent", function (data) {
        console.log(data);
      });

      this.ua.start();
    },
    // 拨打电话
    call() {
      const that = this;
      // Register callbacks to desired call events
      const eventHandlers = {
        'progress': function (e) {
          console.log('call is in progress: ' + e);
        },
        'failed': function (e) {
          console.log('call failed with cause: ' + e);
        },
        'ended': function (e) {
          console.log('call ended with cause: ' + e);
        },
        'confirmed': function (e) {
          console.log(e);
        },
        'peerconnection': function (event) {
          console.log(event);
          event.peerconnection.onaddstream = function (ev) {
            console.info('onaddstream from remote - ', ev);
            that.setRemoteStream(ev);
          };
        }
      };

      const options = {
        'eventHandlers': eventHandlers,
        // 配置请求头信息
        'extraHeaders': ["X-set-id: 1"],
        'mediaConstraints': {'audio': true, 'video': true},
      };
      const session = this.ua.call(this.callSipAdder, options);
      console.log(session);
    },
    // 发信息
    sendMsg() {
      this.ua.sendMessage(this.callSipAdder, this.msg);
    }
  }
}
</script>

<style scoped>

</style>