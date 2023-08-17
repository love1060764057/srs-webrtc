<template>
    <div id="box">
        <!-- 设置自动播放，否则不会显示视频流画面 -->
        <video id="video" autoplay></video>
        <div id="btn">
            <button ref="button_one" @click="publish">开始直播</button>
            <button ref="button_two" @click="close" >停止直播</button>
            <button ref="button_three" @click="stopAudio" >关闭声音</button>
            <button ref="button_four" @click="startAudio" >开启声音</button>
            <button ref="button_five" @click="play" >播放直播</button>
        </div>
        
        <video id="video2" autoplay></video>
    </div>
</template>

<script setup>
import { onMounted,ref } from 'vue';
// 定义全局属性
let videoStream = null;
let videoElement = null;
// 全局的RTCPeerConnection
let pc = null;
// 全局音频轨道，用于RTCRtpSender发送和停止对应轨道
let audioTrack = null;
// 全局的RTCRtpSender
let audioSender = null;
// 获取按钮元素
let button_one = ref(null);
let button_two = ref(null);
let button_three = ref(null);
let button_four = ref(null);
let button_five = ref(null);

const publish = async()=>{
        if(pc!==null&& pc!==undefined){
            console.log("已开始推流");
            return ;
        }
            

        var httpURL = "http://192.168.5.104:1985/rtc/v1/publish/";
        var webRTCURL = "webRTC://192.168.5.104/live/1";
        var constraints = {
                audio: {
                    echoCancellation : true,    // 回声消除
                    noiseSuppression : true,    // 降噪
                    autoGainControl  : true     // 自动增益
                },
                video: {
                    frameRate   : { min : 30 },                // 最小帧率
                    width       : { min : 640, ideal : 1080}, // 宽度   
                    height      : { min : 360, ideal : 720},  // 高度  
                    aspectRadio : 16/9                        // 宽高比
                }
}
        // 通过摄像头、麦克风获取音视频流
        videoStream = await navigator.mediaDevices.getUserMedia(constraints);
        // 获取video元素
        videoElement = document.querySelector("#video")
        //video播放流数据
        videoElement.srcObject = videoStream;
        // 静音
        videoElement.volume=0;
        // 创建RTC连接对象
         pc = new RTCPeerConnection();
        
        // RTCPeerConnection方法addTransceiver()创建一个新的RTCRtpTransceiver，并将其添加到与RTCPeerConnection关联的收发器集中。
        // 每个收发器代表一个双向流，RTCRtpSender和RTCRtpReceiver都与之相关联。
        // 注意添加顺序为audio、video,后续RTCPeerConnection创建offer时SDP的m线顺序遵循此顺序创建，SRS自带的信令服务器响应的SDP中m线总是先audio后video。
        // 若本端SDP和远端SDP中的m线顺序不一直，则设置远端描述时会异常，显示offer中的m线与answer中的m线顺序不匹配
        pc.addTransceiver("audio", {direction: "recvonly"});
        pc.addTransceiver("video", {direction: "recvonly"});
        // 遍历getUserMedia（）获取到的流数据，拿到其中的音频轨道和视频轨道，加入到RTCPeerConnection连接的音频轨道和视频轨道中
        videoStream.getTracks().forEach((track)=>{
            pc.addTrack(track);
        });
        // 创建本端offer
        var offer = await pc.createOffer();
        // 设置本端
        await pc.setLocalDescription(offer);
        var data = {
            "api": httpURL,
            "streamurl":webRTCURL,
            "sdp":offer.sdp
        }
        // SDP交换，请求SRS自带的信令服务器
        httpApi(httpURL,data).then(async(data)=>{
            console.log("answer",data);
            // 设置远端描述，开始连接
            await pc.setRemoteDescription(new RTCSessionDescription({type: 'answer', sdp: data.sdp}));
                button_one.value.disabled=true;
                button_two.value.disabled=false;
                button_three.value.disabled=false;
                button_five.value.disabled=false;

        }).catch((data)=>{
            if(data.code===400){
                console.log("SDP交换失败");
            }
        });
        
    
}

const play = async()=>{
    var httpURL = "http://192.168.5.104:1985/rtc/v1/play/";
    var webRTCURL = "webRTC://192.168.5.104/live/1";
    // 创建RTCPeerConnection连接对象
    var pc = new RTCPeerConnection();
    // 创建媒体流对象
    var stream = new MediaStream();
    // 获取播放流的容器video
    var videoElement2 = document.querySelector("#video2");
    // 监听流
    pc.ontrack = (event)=>{
        // 监听到的流加入MediaStream对象中让video播放
        stream.addTrack(event.track);
        videoElement2.srcObject = stream;
    }
    // RTCPeerConnection方法addTransceiver()创建一个新的RTCRtpTransceiver，并将其添加到与RTCPeerConnection关联的收发器集中。
    // 每个收发器代表一个双向流，RTCRtpSender和RTCRtpReceiver都与之相关联。
    // 注意添加顺序为audio、video,后续RTCPeerConnection创建offer时SDP的m线顺序遵循此顺序创建，SRS自带的信令服务器响应的SDP中m线总是先audio后video。
    // 若本端SDP和远端SDP中的m线顺序不一直，则设置远端描述时会异常，显示offer中的m线与answer中的m线顺序不匹配
    pc.addTransceiver("audio", {direction: "recvonly"});
    pc.addTransceiver("video", {direction: "recvonly"});

    var offer =await pc.createOffer();
    await pc.setLocalDescription(offer)
    var data = {
            "api": httpURL,
            "streamurl":webRTCURL,
            "sdp":offer.sdp
    }
    // SDP交换，请求SRS自带的信令服务器
    httpApi(httpURL,data).then(async(data)=>{
            console.log("answer",data);
            // 设置远端描述，开始连接
            await pc.setRemoteDescription(new RTCSessionDescription({type: 'answer', sdp: data.sdp}));
            button_five.value.disabled=true;

    }).catch((data)=>{
            if(data.code===400){
                console.log("SDP交换失败");
            }
    });
}

// 关闭连接
const close = ()=>{
    if(pc!==null&&pc!==undefined){
        pc.close();
        pc = null;
        button_one.value.disabled=false;
        button_two.value.disabled=true;
        button_three.value.disabled=true;
        button_four.value.disabled=true;
        button_five.value.disabled=true;
    }
}
// 关闭音频
const stopAudio = ()=>{
    if(pc!==null&&pc!==undefined){
        // RTCPeerConnection方法getSenders()返回RTCRtpSender对象的数组，
        // 每个对象代表负责传输一个轨道的数据的RTP发送器。
        // sender对象提供了检查和控制音轨数据的编码和传输的方法和属性。
        pc.getSenders().forEach((sender)=>{
            if(sender.track!==null&&sender.track.kind==="audio"){
                // 拿到音频轨道
                audioTrack = sender.track;
                // 拿到音频轨道发送者对象RTCRtpSender
                audioSender = sender;
                // RTCRtpSender的replaceTrack()可以在无需重新媒体协商的情况下用另一个媒体轨道更换当前正在发送轨道
                // 参数为空则将当前正在发送的轨道停止，比如关闭音频，再次开启时将音频轨道作为参数传入
                audioSender.replaceTrack(null);
                button_three.value.disabled=true;
                button_four.value.disabled=false;
            }
        });
    }
}
// 开启音频
const startAudio = ()=>{
    console.log(audioSender);
    if(pc!==null&&pc!==undefined){
       if(audioSender.track===null){
        audioSender.replaceTrack(audioTrack);
        button_three.value.disabled=false;
        button_four.value.disabled=true;
       }
    }
}

const httpApi = (httpURL,data)=>{
    var promise = new Promise((resolve,reject)=>{
        var xhr = new XMLHttpRequest();
        xhr.open('POST', httpURL, true);
        xhr.setRequestHeader('Content-type', 'application/json');
        xhr.send(JSON.stringify(data));
        xhr.onload = ()=>{
                if (xhr.readyState !== xhr.DONE) reject(xhr);
                if (xhr.status !== 200 && xhr.status !== 201) reject(xhr) ;
                var data = JSON.parse(xhr.responseText);
                if(data.code===0){
                    resolve(data);
                }else{
                    reject(data)
                }
            }
    });
    return promise;
}

onMounted(()=>{
    button_one.value.disabled=false;
    button_two.value.disabled=true;
    button_three.value.disabled=true;
    button_four.value.disabled=true;
    button_five.value.disabled=true;
});

</script>

<style lang="scss">
*{
    margin: 0;
    padding: 0;
    border: 0;
    box-sizing: border-box;
}
#box{
    width: 100%;
    text-align: center;
}
video{
    background-color: black;
    width: 500px;
    height: 400px;
    object-fit: cover;
}
#btn{
    width: 80%;
    height: 100px;
    display: flex;
    margin:10px 10%;
}
button{
    flex: 1;
    height: 100px;
    background-color: aqua;
    border-radius: 20px;
    margin-left: 10px;
}
button:nth-child(1){
    margin-left: 0;
}
</style>
