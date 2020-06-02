<template>
  <v-layout>
    <video id="yourVideo" autoplay muted></video>
    <video id="friendsVideo" autoplay muted></video>
    <v-btn @click="showFriendsFace">show friend</v-btn>
  </v-layout>
</template>
<script>
  export default {
    data () {
      return {
        database: null,
        yourVideo: null,
        friendsVideo: null,
        yourId: null,
        servers: {},
        pc: null
      }
    },
    methods: {
      sendMessage(senderId, data) {
        let msg = this.database.add({ sender: senderId, message: data })
      },
      showMyFace() {
        navigator.mediaDevices.getUserMedia({audio:true, video:true})
          .then(stream => this.yourVideo.srcObject = stream)
          .then(stream => this.pc.addStream(stream))
      },
      showFriendsFace () {
        this.pc.createOffer()
          .then(offer => this.pc.setLocalDescription(offer) )
          .then(() => this.sendMessage(this.yourId, JSON.stringify({'sdp': this.pc.localDescription})))
      },
      readMessage(data) {
        if (!data.message) {
          return
        }
        let msg = JSON.parse(data.message)
        let sender = data.sender
        if (sender !== this.yourId) {
          if (msg.ice !== undefined)
            this.pc.addIceCandidate(new RTCIceCandidate(msg.ice));
          else if (msg.sdp.type == 'offer')
            this.pc.setRemoteDescription(new RTCSessionDescription(msg.sdp))
              .then(() => this.pc.createAnswer())
              .then(answer => this.pc.setLocalDescription(answer))
              .then(() => this.sendMessage(this.yourId, JSON.stringify({'sdp': this.pc.localDescription})))
          else if (msg.sdp.type == 'answer')
            this.pc.setRemoteDescription(new RTCSessionDescription(msg.sdp));
        }
      }
    },
    mounted () {
      this.database = this.$fireStore.collection('messages')
      this.yourVideo = document.getElementById('yourVideo')
      this.friendsVideo = document.getElementById('friendsVideo')
      this.yourId = Math.floor(Math.random() * 1000000000)
      this.servers = {'iceServers': [
        {'urls': 'stun:stun.services.mozilla.com'},
          {'urls': 'stun:stun.l.google.com:19302'},
          {'urls': 'turn:numb.viagenie.ca','credential': 'webrtc','username': 'websitebeaver@mail.com'}]}
      let pc = new RTCPeerConnection(this.servers);
      pc.onicecandidate = (event => event.candidate ? this.sendMessage(this.yourId, JSON.stringify({'ice': event.candidate})) : console.log('Sent All Ice') )
      pc.onaddstream = (event => this.friendsVideo.srcObject = event.stream)
      this.pc = pc
      this.database.onSnapshot(snapshot => {
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
    transform: rotateY(180deg);
    -webkit-transform:rotateY(180deg); /* Safari and Chrome */
    -moz-transform:rotateY(180deg); /* Firefox */
  }
</style>
