<template>
	<view>
		
		<scroll-view id="scrollview" scroll-y :scroll-top="scrollTop" 
		:scroll-with-animation="true"
		refresher-enabled
		@refresherrefresh="scrollTopHandle"
		:refresher-triggered="triggered"
		:style="{height:style.contentH+'px'}">
			<!-- 聊天列表 -->
			<block v-for="(item,index) in currentChatMsgs" :key="index">
				<view class="chat-item">
				<user-chat-list 
					@goToUserInfo="goToUserInfo"
					:item="item" :index="index"></user-chat-list>
				</view>
			</block>
		</scroll-view>
		
		<!-- 输入框 -->
		<user-chat-bottom @submit="submit"></user-chat-bottom>
	</view>
</template>

<script>
	import userChatBottom from "../../components/user-chat/user-chat-bottom.vue";
	import time from "../../common/time.js";
	import userChatList from "../../components/user-chat/user-chat-list.vue";
	import {mapState,mapMutations,mapGetters} from 'vuex'
	import {pushMessage,createChat} from '@/api/user-chat.js'
	import Vue from 'vue'
	export default {
		components:{
			userChatBottom,
			userChatList
		},
		computed:{
			...mapState(['chatList','userInfo','msgIndex','msgPage']),
			...mapGetters(['currentChatMsgs'])
		},
		data() {
			return {
				scrollTop:0,
				triggered:false,
				index:-1,
				style:{
					contentH:0,
					itemH:0
				},
				list:[],
				cId:0,
				socket:null,
				fid: undefined,
				isShow:false

			};
		},
		onShow() {
			this.isShow = true
		},
		beforeDestroy() {
			this.isShow = false
			this.setIndex(-1)
			this.setMsgPage(1)
		},
		// 监听下拉刷新
		
		async onLoad(data) {
			if(data.index){
				this.setIndex(parseInt(data.index))
				this.index = data.index
				uni.setNavigationBarTitle({
					title:this.chatList[this.msgIndex].username
				})
			}
			 this.fid = data.fid
			 let fid = data.fid
			let flag = true
			if(!data.index&&!!fid){
				for(let i =0;i<this.chatList.length;i++){
					if(this.chatList[i].fid == fid){
						this.index = i
						this.setIndex(i)
						flag = false
						return
					}
				}
				if(flag){
					this.fid = fid
					let chat = await createChat({
						fromId: this.userInfo.id,
						toId:parseInt(fid),
					})
					this.cId = chat.id
					chat.time= time.gettime.gettime(chat.afterTime)
					this.setIndex(0)
					this.addChatList(chat);
				}

			}else{
				this.fid = this.chatList[data.index].fid
				this.cId = this.chatList[data.index].id
			}
	
		},

		onReady() {
			this.getdata();
			this.initdata();
			this.pageToBottom(true);
		},
		watch:{
			currentChatMsgs(old){
				if(this.triggered){
					
				}else{
					this.pageToBottom(true);
				}
			}
		},
		methods:{
			...mapMutations(['setChatMessage',
			'setIsPaper',
			'addChatList',
			'setIndex',
			'setMsgPage',
			'addChatMessage','addNoreadMessage']),
			// 初始化参数
			initdata(){
				try {
					const res = uni.getSystemInfoSync();
					this.style.contentH=res.windowHeight - uni.upx2px(120);
				} catch (e) { }
			},
			scrollTopHandle(){
				if (this.triggered) {
					setTimeout(()=>{
						if(this.msgPage>(this.currentChatMsgs.length/20)){
							uni.showToast({
								title:"没有更多消息了!",
								icon:'none'
							})
						}
						this.triggered = false;
					},200)
					return;
				}
				this.triggered = true;
				this.setMsgPage()
			},
			pageToBottom(isfirst = false){
				let q=uni.createSelectorQuery().in(this);
				let itemH = q.selectAll('.chat-item');
				if(this.currentChatMsgs.length!=0){
					this.$nextTick(()=>{
						itemH.fields({
							size:true
						},data => {
							if (data) {
								if (isfirst) {
									for (let i = 0; i < data.length; i++) {
										this.style.itemH += data[i].height;
									}
								}else{
									this.style.itemH += data[data.length-1].height;
								}
								this.scrollTop = (this.style.itemH > this.style.contentH) ? this.style.itemH : 0;
							}
						}).exec();
					})
				}
			},

			goToUserInfo(item){
				uni.navigateTo({
					url:'../../pages/user-space/user-space?uid='+item.uid
				})
			},
			// 获取聊天数据
			async getdata(){
				// 从服务器获取到的数据
				if( this.chatList.length==0){
					return
				}
			},
			async submit(data){
				// 构建数据
				let now=new Date().getTime();
				if(data==''){
					uni.showToast({
						title:"消息不能为空",
						icon:'none'
					})
					return
				}
				let msg =await pushMessage({
					cId: this.cId,
					fromId: this.userInfo.id,
					toId: this.fid,
					message: data
				}) 
				if(msg.code&&msg.code!=0){
					uni.showToast({
						title:'消息发送失败!',
						icon:'none'
					})
					return
				}
				let obj={
						fromId:msg.fromId,
						toId:msg.toId,
						index:this.index,
						isme:msg.fromId==this.userInfo.id,
						userpic:this.userInfo.authorUrl,
						type:"text",
						message:msg.message,
						time:  time.gettime.gettime(msg.sendTime),
						sendTime:msg.sendTime
					}
				this.addChatMessage(obj)
				this.pageToBottom(true);
			}
		}
	}
</script>

<style>

</style>
