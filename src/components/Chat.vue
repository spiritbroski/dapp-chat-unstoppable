<template>
    <div>
        <div class="container">
            <div class="connectionDetails">
                <div class="level is-mobile">
                    <StatBox heading="Connected" :content="peerCount" footer="Peer" v-on:click.native="toggleShowPeers" />
                    <StatBox heading="WebSocket" :content="webSocketPeerCount" footer="Super Peer" v-on:click.native="toggleShowPeers" />
                    <StatBox v-if="webrtcCompatible" heading="WebRTC" :content="webRTCPeerCount" footer="Peer" v-on:click.native="toggleShowPeers" />
                    <StatBox heading="Message" :content="messageCount" footer="Count" notplural v-on:click.native="toggleShowPeers" />
                </div>
                <div v-if="peerCount">
                    <PeerList v-if="showPeers" :peers="peers" />
                </div>
                <div v-else class="notification is-info">
                    Offline
                </div>
            </div>
        </div>
        <div class="container chat" id="chat">
            <div class="box messages-box">
                <div v-for="(msg, name) in messages" :key="msg.name">
                    <div class="box chatmessages" v-if="msg" :id="name">
                        <div class="columns">
                            <div v-if="msg.user" class="column is-narrow message-user" :style="{backgroundColor: stringToColor(msg.user)}">
                                {{ msg.user }}
                            </div>
                            <div class="column">
                                {{ msg.msg }}
                            </div>
                            <div class="column is-narrow timestamp-column">
                                <Timestamp class="timestamp" :forceUpdate="incrementor" :time="msg.when" />
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <div class="container">
            <div class="form" v-if="isUnstoppable || isLogin">
                <div class="columns is-gapless">
                    <div class="column is-narrow">
                        <input class="input is-medium username" :disabled="isUnstoppable" v-model="user" :style="{backgroundColor: stringToColor(user)}" style="color: white">
                    </div>
                    <div class="column">
                        <input class="input is-medium" v-model="newMessage" placeholder="say hello..." v-on:keyup.enter="send" />
                    </div>
                    <div class="column is-narrow">
                        <div class="button is-success is-medium" :class="[{ 'is-loading': sending }]" v-on:click="send">{{ sendButtonText }}</div>
                    </div>
                </div>
            </div>
             <div class="columns">
                    <div class="column mt-5">
                        <button v-if="!isLogin" @click="loginAsGuest" class="button button2 is-dark">Login As Guest</button>
                      
                    </div>
                    <div class="column mt-5">
                        <button v-if="!isUnstoppable && !isLogin" @click="login" class="button button2 is-link" style="background-color:#4b47ee;"><img style="height:30px;" :src="require('../images/a.svg')" />Login with Unstoppable</button>
                    </div>
                   
                </div>
                 <div class="column mt-5">
                        <button v-if="isUnstoppable || isLogin" class="button button2 is-danger" @click="logout">Log Out</button>
                    </div>
            <div v-if="error" class="notification is-warning sendingError">Error: {{ error }}</div>
        </div>
        <div class="chatroom"><span class="chatroom-label-chat">Chat</span>room {{chatroom}}</div>
        <div class="kill-peers" v-on:click="closeConnection('all')">Go Offline</div>
    </div>
</template>

<script>
import Timestamp from '@/components/Timestamp'
import PeerList from '@/components/PeerList'
import StatBox from '@/components/StatBox'
import Gun from 'gun/gun'
import {animals} from '@/misc/animals'
import {adjectives} from '@/misc/adjectives'
import UAuth from '@uauth/js'
export default {
    components: {
        Timestamp,
        PeerList,
        StatBox
    },
    name: 'Chats',
    gun: new Object, //non-reactive, handle changes with .on
    data: function() {
        return {
            isUnstoppable:false,
            isLogin:false,
            //gun: null,
            messages: {},
            mesh: null,
            newMessage: "",
            sendButtonText: "Send",
            sending: false,
            error: "",
            root: 'dchat',
            devprod: process.env.NODE_ENV === 'development' ? 'dev' : 'p',
            chatroom: location.hash.slice(1) || 1,
            peers: {},
            user: '',
            showPeers: false,
            incrementor: 1,
            webrtcCompatible: window.RTCPeerConnection || false
        }
    },
    computed: {
        rootNode: function(){
            return this.root+'/'+this.devprod+'/'+this.chatroom
        },
        sortedMessages: function(){
            //todo
            return '';
        },
        messageCount: function(){
            return Object.keys(this.messages).length
        },
        peerCount: function(){
            return Object.keys(this.peers).length
        },
        webSocketPeerCount: function() {
            let c = 0;
            // eslint-disable-next-line no-unused-vars
            for(const[key,value] of Object.entries(this.peers)){
                if(value.wire !== undefined && value.wire !== null){
                    if(value.wire.constructor.name == "WebSocket"){
                        c++
                    }
                }
            }
            return c
        },
        webRTCPeerCount: function() {
            let c = 0;
            // eslint-disable-next-line no-unused-vars
            for(const[key,value] of Object.entries(this.peers)){
                if(value.wire !== undefined && value.wire !== null){
                    if(value.constructor.name == "RTCPeerConnection"){
                        c++
                    }
                }
            }
            return c
        }
    },
    methods: {
        loginAsGuest(){
            this.isLogin = true
                    this.isUnstoppable = false
                     this.user = this.generateUsername()
        },
        async login(){
              try {
                const uauth = new UAuth({
                  clientID: 'e23e7fbe-75fb-42fd-a935-483305535225',redirectUri:"https://dapp-chat-virid.vercel.app"
                })
                const authorization = await uauth.loginWithPopup()
 if(authorization){
                    this.isLogin = true
                    this.isUnstoppable = true
                    this.user = authorization.idToken.sub
                    
 }
                // console.log(authorization)
              } catch (e) {
                console.log(e.message);
                         this.isLogin = false
                    this.isUnstoppable = false
                // onJoinUnstoppable(false, null);
              }
        },
        async logout(){
            localStorage.user='';
            this.isLogin = false
                    this.isUnstoppable = false
        },
        send(){
            if(this.newMessage.trim() == ''){
                return
            }
            let msgId = this.generateId(20)
            if(this.user == ''&& this.isUnstoppable===false){
                this.user = this.generateUsername()
            }
            //this.$options.gun.get(msgId).put(this.newMessage)// XXX WORKING WITH WEBRTC XXX

            if(this.webSocketPeerCount){
                this.$options.gun.get(msgId).put(JSON.stringify({user: this.user, msg: this.newMessage, when: Gun.state()}), this.sendCallback)
                this.sending = true
            } else {
                //no callback if no websocket connection
                this.$options.gun.get(msgId).put(JSON.stringify({user: this.user, msg: this.newMessage, when: Gun.state()}))
                this.newMessage = ""
            }
        },
        sendCallback(ack){
            //console.log(ack)
            if(ack.ok == 1){
                this.newMessage = ""
                this.sending = false
            }
            if(!ack.ok){
                console.log("failed ack")
                console.log(ack)
                if(ack.err == undefined){
                    this.error = "Couldn't send message - possibly sent to dead webrtc peer"
                    //this.newMessage = ""
                    this.sending = false
                } else {
                    this.error = ack.err
                }
                this.sending = true
            }
        },
        closeConnection(peer){
            if(peer === 'all'){
                this.$gun.opt({peers: []})
                return
            }
            /*
            //TODO handle closing individual peers

            console.log("peers",this.peers)
            //console.log(this.$gun.back('opt.peers'))
            let newPeerList = []
            if(peer.id !== undefined){
                //console.log(peer)
                // eslint-disable-next-line no-unused-vars
                for(const[key,value] of Object.entries(this.peers)){
                    if(peer.id === value.id){
                        //console.log("key",key)
                        //console.log("value",value)
                        //console.log("gunval", this.$gun.back('opt.peers')[value.id])
                        if(this.$gun.back('opt.peers')[value.id]){
                            if(this.$gun.back('opt.peers')[value.id].wire){
                                this.$gun.back('opt.peers')[value.id].wire.close()
                            }
                        }
                    }
                    else {
                        newPeerList.push(value)
                    }
                }
                console.log("newpeerlist",newPeerList)
                this.$gun.opt({peers: []})
                this.$gun.opt({peers: newPeerList})
            }
            */
        },
        connectionDetails(){
            this.peers = Object.assign({}, this.$gun.back('opt.peers'))

        },
        getGunUpdates(){
            this.$options.gun.map().on(this.processGunUpdate, true)
        },
        //processGunUpdate(value, key, x, y){
        processGunUpdate(value, key){
            //console.log('got update', value, key)
            if(value){
                this.$set(this.messages, key, JSON.parse(value))
                this.scrollToMessage(key)
            }
        },
        failedPeer(peer){
            console.log("failed peer in chat.vue",peer)
            this.closeConnection(peer)
        },
        getSoul(data){
            if(Gun.node.is(data)){
                return(Gun.node.soul(data))
            }
            return false
        },
        getTimestamp(json){
            // eslint-disable-next-line
            if(Gun.node.is(json)){
                let data = JSON.parse(JSON.stringify(json))
                let timestamp = data["_"][">"].msg
                if(isNaN(timestamp)){
                    return 0
                } else {
                    return timestamp
                }
            } else {
                return 0
            }
        },
        toggleShowPeers(){
            this.showPeers = this.showPeers ? false : true
        },
        stringToColor(str){
            let hash = 0;
            for (let i = 0; i < str.length; i++) {
                hash = str.charCodeAt(i) + ((hash << 5) - hash);
            }
            let colour = '#';
            for (let i = 0; i < 3; i++) {
                let value = (hash >> (i * 8)) & 0xFF;
                colour += ('00' + value.toString(16)).substr(-2);
            }
            return colour;
        },
        generateId(len){
            let arr = new Uint8Array((len || 40) / 2)
            window.crypto.getRandomValues(arr)
            return Array.from(arr, this.dec2hex).join('')
        },
        dec2hex(dec){
            return(dec.toString(16).padStart(2, '0'))
        },
        scrollToMessage(id){
            this.$nextTick(() => {
                let element = document.getElementById(id)
                if(element){
                    element.scrollIntoView({behavious: "smooth", block: "end", inline: "nearest"})
                }
            })
        },
        generateUsername(){
            let animal = animals[Math.floor(Math.random() * animals.length)]
            animal = animal.charAt(0).toUpperCase() + animal.slice(1)
            let adjective = adjectives[Math.floor(Math.random() * adjectives.length)]
            adjective = adjective.charAt(0).toUpperCase() + adjective.slice(1)
            return adjective+" "+animal
        }
    },
    mounted: function(){
        this.$options.gun = this.$gun.get(this.rootNode)
        //console.log(this.$options.gun)

        this.$gun.on('hi', (peer) => {
            console.log('hi', peer)
            this.connectionDetails()
        })
        this.$gun.on('bye', (peer) => {
            console.log('bye', peer)
            this.connectionDetails()
        })
if(this.isUnstoppable){
    console.log()
}
       else {
            this.user = this.generateUsername()
        }
        this.getGunUpdates()
        this.connectionDetails()

        setInterval(() => {
            this.connectionDetails()
        }, 2500)

        //update relative timestamps every 30 seconds
        setInterval(() => {
            this.incrementor += 1
        }, 30000)
    },
    created: function(){
    },
    watch: {
        user(newName){
            localStorage.user = newName
        },
        error(err){
            if(String(err).includes('Error: ')){
                this.error = String(err).replace('Error: ','')
            }
        }
    }

}
</script>

<style scoped>
.timestamp {
    font-size: 0.666rem;
}
.chatmessages {
    text-align: left;
    margin-bottom: 2px;
}
.chat {
    max-height: 70vh;
    overflow-y: auto;
    margin: 1.25rem auto;
}
@media screen and (max-width: 768px){
    .chat {
        max-height: 45vh;
        overflow-y: auto;
        margin: 1.25rem auto;
    }
}
.messages-box {
    background-color: transparent;
    padding: 0 2px 0 0;
}
.input {
    border-color: transparent;
    border-radius: 0;
}
.username {
    border-radius: 4px 0 0 4px;
    text-align: center;
    padding-left: 0.75em;
    padding-right: 0.75em;
}
@media screen and (max-width: 768px){
    .username {
        width: 100%;
        border-radius: 4px 4px 0 0 ;
    }
}
.button {
    border-radius: 0 4px 4px 0;
}
.button2 {
    border-radius: 4px 4px 4px 4px;
}
.button2:hover{
    background-color:#0b24b3;
}

@media screen and (max-width: 768px){
    .button {
        border-radius: 0 0 4px 4px;
        padding-left: 2em;
        padding-right: 2em;
        width: 100%;
    }
     .button2 {
        border-radius: 4px 4px 4px 4px;
        padding-left: 2em;
        padding-right: 2em;
        width: 100%;
    }
}
@media screen and (max-width: 768px){
    .input {
        text-align: center;
    }
}
.level .level-item .box {
    width: 8em;
}
@media screen and (max-width: 768px){
    .level .level-item .box {
        padding: 0.25rem;
        width: auto;
        min-width: 5em;
    }
}
.sending-error {
    margin-top: 2px;
}
.message-user {
    min-width: 6em;
    text-align: center;
    color: #fff;
}
@media screen and (max-width: 768px){
    .message-user {
        text-align: left;
        padding: 0 0.75rem;
    }
}
.timestamp-column {
    border-left: 1px dashed lightgrey;
}
@media screen and (max-width: 768px){
    .timestamp-column {
        border-left: none;
        border-top: 1px dashed lightgrey;
        text-align: right;
        padding: 0 0.75rem;
    }
}
.chatroom {
    position: fixed;
    top: 0;
    right: 0;
    border-left: 1px dashed lightgrey;
    border-bottom: 1px dashed lightgrey;
    color: #fff;
    padding: 0.5rem;
}
@media screen and (max-width: 768px){
    .chatroom {
        padding:0 0.15rem;
        text-transform:capitalize;
    }
    .chatroom-label-chat {
        display: none;
    }
}
.kill-peers {
    position: fixed;
    top: 0;
    left: 0;
    border-right: 1px dashed lightgrey;
    border-bottom: 1px dashed lightgrey;
    color: #fff;
    padding: 0.5rem;
    cursor: pointer;
}
@media screen and (max-width: 768px){
    .kill-peers {
        padding:0 0.15rem;
        text-transform:capitalize;
    }
}
</style>
