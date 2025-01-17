<script setup lang="ts">
import { ElMessage } from "element-plus";
import Peer, { DataConnection } from "peerjs";
import { ref, watch } from "vue";
const id = ref("");
const remoteId = ref("");
const ensureId = ref(false);
const peer = ref<Peer>(new Peer());

const ensure = () => {
	if (id) {
		ensureId.value = true;
	}
};

watch(
	() => ensureId.value,
	() => {
		if (ensureId.value) {
			peer.value = new Peer(id.value);

			peer.value.on("open", () => {
				console.log("opened, my id is ", id.value);
			});

			peer.value.on("connection", (curConnect) => {
				console.log("has someone connect me!");
				curCon.value = curConnect;
				connected.value = true;

				addDataListen(curConnect);
			});
		}
	}
);

const caller = ref(false);
const receiver = ref(false);
const calling = ref(false);
const localVideo = ref<HTMLVideoElement>(document.createElement("video"));
const remoteVideo = ref<HTMLVideoElement>(document.createElement("video"));

const getMyselfVideoStream = async () => {
	const stream = await navigator.mediaDevices.getUserMedia({
		video: true,
		audio: true,
	});

	return stream;
};
const startCall = async () => {
	const stream = await getMyselfVideoStream();
	localVideo.value.srcObject = stream;
	localVideo.value.play();

	const call = peer.value?.call(remoteId.value, stream);
	call?.on("stream", (remoteStream) => {
		// const remoteVideo = document.getElementById('')
		remoteVideo.value.srcObject = remoteStream;
	});
	caller.value = true;
};

//@ts-ignore
const receiverCall = () => {
	peer.value.on("call", async (call) => {
		const stream = await getMyselfVideoStream();
		localVideo.value.srcObject = stream;

		call.answer(stream); // 将stream给到发送方
		call.on("stream", (remoteStream) => {
			remoteVideo.value.srcObject = remoteStream;
		});
	});
};

const transformMirror = (el: HTMLElement) => {
	if (el) {
		const transform = el.style.transform;
		el.style.transform = transform + "scaleX(-1)";
	}
};

const messageList = ref<{ id: string; data: any }[]>([]);
const message = ref("");
const connected = ref(false);
const curCon = ref<DataConnection>();
// const remoteCon = ref<any>();
const connect = () => {
	if (!remoteId.value) {
		ElMessage.error("please input remote name");
	}
	const curConnect = peer.value.connect(remoteId.value);
	curCon.value = curConnect;
	curConnect.on("open", () => {
		console.log("connected!!");
		connected.value = true;

		addDataListen(curConnect);

		send("hello");
	});
};

const addDataListen = (con: DataConnection) => {
	con.on("data", (data) => {
		console.log("receive data: ", data);
		messageList.value.push(data as any);
	});
};

const clickSend = () => {
	if (!message.value) return;
	send(message.value);
	message.value = "";
};
const send = (value: string) => {
	if (!value) return;
	curCon.value!.send({ id: id.value, data: value });

	messageList.value.push({
		id: id.value,
		data: value,
	});
};
</script>

<template>
	<div class="container p-5">
		<h4 class="mb-2">
			MY Name:
			<div class="inline-flex items-center w-fit">
				<el-input
					v-model="id"
					:disabled="ensureId"
					placeholder="please input your name"
					class="max-w-[200px]"></el-input>
				<el-button @click="ensure" v-if="!ensureId">确定</el-button>
			</div>
		</h4>

		<div class="my-5" v-if="false">
			<el-input
				v-model="remoteId"
				placeholder="please input your friend name"
				class="max-w-[200px] mr-2"></el-input>

			<el-button v-if="!receiver" type="primary" @click="startCall">发起视频</el-button>
			<el-button v-if="calling" class="bg-red-500 w-[100px] h-[30px] ml-5">结束视频</el-button>
		</div>

		<div class="flex justify-between mt-5 border p-5" v-show="calling">
			<div>
				<p>本地视频</p>
				<video ref="localVideo" controls autoplay muted></video>
				<el-button type="primary" class="mt-5" @click="transformMirror(localVideo)">镜像反转</el-button>
			</div>
			<div class="ml-[10px]">
				<p>远程视频</p>
				<video ref="remoteVideo" controls autoplay></video>
				<el-button type="primary" class="mt-5" @click="transformMirror(remoteVideo)">镜像反转</el-button>
			</div>
		</div>

		<div class="chat-container m-2 p-2 border" v-show="ensureId">
			<div class="connect-container" v-show="!connected">
				<el-input v-model="remoteId" placeholder="please input remote id" class="max-w-[200px] mr-2"></el-input>
				<el-button @click="connect">连接</el-button>
			</div>

			<div class="send-container my-2" v-show="connected">
				<el-input v-model="message" class="max-w-[200px] mr-2"></el-input>
				<el-button @click="clickSend">发送</el-button>
			</div>

			<div v-if="messageList.length" class="message-container border p-2">
				<ul>
					<li v-for="msg in messageList">{{ msg.id }}: {{ msg.data }}</li>
				</ul>
			</div>
		</div>
	</div>
</template>

<style>
video {
	transform: scaleX(1);
}
</style>
