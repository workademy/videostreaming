<template>
  <div>
    <video id="yourVideo" autoplay muted></video>
    <video id="friendsVideo" autoplay></video>
    <v-btn @click="showFriendsFace">Show friend</v-btn>
  </div>
</template>

<script>
  const SERVERS = {
    iceServers: [{urls: 'stun:stun.services.mozilla.com'},
      {urls: 'stun:stun.l.google.com:19302'},
      {urls: 'turn:numb.viagenie.ca', credential: 'webrtc', username: 'websitebeaver@mail.com'}
    ]
  }
  export default {
    data () {
      return {
        id: Math.floor(Math.random() * 1000000000),
        pc: new RTCPeerConnection(SERVERS),
        db: this.$fireStore.collection('messages')
      }
    },
    methods: {
      sendMessage (senderId, data) {
        let msg = this.db.add({ sender: senderId, message: data })
      },
      readMessage (data) {
        let msg = data.message ? JSON.parse(data.message) : ''
        if (!msg) {
          return
        }
        let sender = data.sender
        if (msg && msg.sdp) {
          console.log(msg.sdp.type)
        }
        if (sender === this.id) {
          return
        }
        if (msg.ice) {
          this.pc.addIceCandidate(new RTCIceCandidate(msg.ice))
        } else if (msg.sdp && msg.sdp.type === 'offer') {
          this.pc.setRemoteDescription(new RTCSessionDescription(msg.sdp))
            .then(() => this.pc.createAnswer())
            .then(answer => this.pc.setLocalDescription(answer))
            .then(() => this.sendMessage(this.id, JSON.stringify({sdp: this.pc.localDescription})))
        } else if (msg.sdp && msg.sdp.type === 'answer') {
          this.pc.setRemoteDescription(new RTCSessionDescription(msg.sdp))
        }
      },
      preparePeerConnections () {
        let $friendsVideo = document.getElementById('friendsVideo')
        this.pc.onicecandidate = event =>
          event.candidate ? this.sendMessage(this.id, JSON.stringify({'ice': event.candidate})) : console.log('Sent All Ice')
        this.pc.onaddstream = event => $friendsVideo.srcObject = event.stream
      },
      showMyFace () {
        let $yourVideo = document.getElementById('yourVideo')
        navigator.mediaDevices.getUserMedia({audio:true, video:true})
          .then(stream => $yourVideo.srcObject = stream)
          .then(stream => this.pc.addStream(stream))
      },
      showFriendsFace () {
        this.pc.createOffer()
          .then(offer => this.pc.setLocalDescription(offer))
          .then(() => this.sendMessage(this.id, JSON.stringify({'sdp': this.pc.localDescription})) )
      }
    },
    mounted () {
      this.preparePeerConnections()
      this.db.onSnapshot(snapshot => {
        snapshot.docChanges().forEach(change => {
          if (change.type === 'added') {
            this.readMessage(change.doc.data())
          }
        })
      })
      this.showMyFace()
    }
  }
</script>
<style>
  video {
    background-color: #ddd;
    border-radius: 7px;
    margin: 10px 0px 0px 10px;
    width: 320px;
    height: 240px;
  }
</style>
