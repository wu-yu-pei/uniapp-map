<template>
	<view class="content">
		<view>
			<uni-popup ref="inputDialog" type="dialog">
				<uni-popup-dialog ref="inputClose"  mode="input" title="输入组队号" value=""
					placeholder="请输入" @confirm="dialogInputConfirm"></uni-popup-dialog>
			</uni-popup>
		</view>
		<uni-popup ref="message" type="message">
			<uni-popup-message :type="msgType" :message="messageText" :duration="2000"></uni-popup-message>
		</uni-popup>
		
		<view class="roomNo">
			<span>角色: {{type}} </span>
			<span>房间: {{room}} </span>
			<span>人数: {{users.length + 1}} </span>
			<span>版本: 0.0.20</span>
			<span>坐标: {{longitude}}</span>
		</view>
		<map 
		name="map" 
		class="map" 
		show-location="true" 
		enable-3D="ture" 
		isHighAccuracy="true" 
		enable-indoorMap="ture"
		:markers="users"
		:longitude="longitude" 
		:latitude="latitude"></map>
		
		<view class="control">
			<button @click="create">创建房间</button>
			<button @click="join">加入房间</button>
			<button @click="out">退出房间</button>
		</view>
	</view>
</template>

<script>
	import io from '@hyoga/uni-socket.io';
	export default {
		data() {
			return {
				longitude:'',
				latitude:'',
				type:null,
				room:null,
				msgType:'success',
				messageText:'',
				users:[],
				userinfo:{},
				socket:'',
				id:Math.floor(Math.random()*1000)
			}
		},
		onLoad() {
			this.getLocation().then(res => {
				this.longitude = res[0]
				this.latitude = res[1]
			})
		},
		onShow() {
			if(this.room && this.type == 'creater') {
				console.log('creater重新加入房间。。。。');
				this.createRoom()
			}else if(this.room && this.type == 'joiner') {
				console.log('joiner重新加入房间。。。。');
				this.joinRoom()
			}
		},
		methods: {
			message(type, text) {
				this.msgType = type
				this.messageText = text
				this.$refs.message.open()
			},
			inputDialogToggle() {
				this.$refs.inputDialog.open()
			},
			dialogInputConfirm(val) {
				this.$refs.inputDialog.close()
				this.room = val
				if(this.type === 'creater') {
					this.createRoom()
				}else {
					this.joinRoom()
				}
			},
			// 获取当前位置
			getLocation() {
				return new Promise((resolve, reject) => {
					uni.getLocation({
						type: 'gcj02',
						success:  (res) =>  {
							resolve([res.longitude, res.latitude])
						}
					});
				})
			},
			// 获取用户信息
			getUserInfo() {
				return new Promise((resolve, reject) => {
					wx.getUserProfile({
						desc: '用于完善相关信息', 
						success: (res) => {
							console.log('用户信息:', res);
							this.userinfo = res.userInfo
							resolve(res)
						},
						fail(err) {
							reject(err)
						}
					})
				})
			},
			
			create() {
				if(this.isInRoom()) {
					this.message('error', '已经在房间里了!')
					return
				}
				this.type = 'creater'
				this.inputDialogToggle()
			},
			join() {
				if(this.isInRoom()) {
					this.message('error', '已经在房间里了!')
					return
				}
				this.type = 'joiner'
				this.inputDialogToggle()
			},
			
			isInRoom() {
				return this.room
			},
			// 输入房间号 创建房间
			async createRoom() {
				// 是否用用户信息
				if(Object.keys(this.userinfo).length === 0) {
					try{
						await	this.getUserInfo()
					}catch(e){
						this.message('error', '获取个人信息失败')
						
						this.room = null
						this.type = null
						return
					}
				}
				// 连接socket
			 	const socket =  io('https://wuyupei.top:8888', {
				  query: {
					  auth: this.room,
					  type: this.type,
					  id: this.id
				  },
				  transports: [ 'websocket', 'polling' ],
				  timeout: 5000,
				});
				Object.defineProperty(this, 'sockets', {
					value: socket,
					configurable : true,
					writable : true,
					enumerable : true,
				})
				socket.on('location',res => {
					res = typeof res === 'string' ? JSON.parse(res) : res
					if(res.type === 'creater') return
					const result =	this.users.findIndex((item, index) => item.id == res.id) 
					
					if(result == -1) {
						this.users.push(res)
					}else {
						this.users.splice(result, 1)
						this.users.push(res)
					}
					
				})
				
				socket.on('joner-disconnect', res => {
					this.message('warning', res.msg)
					const result =	this.users.findIndex((item, index) => item.id == res.id)
					
					if(result == -1) {
						return
					}else {
						this.users.splice(result, 1)
					}
					
				})
				
				socket.emit('location', {
					id:this.id,
					type:this.type,
					width: 30,
					height: 30,
					latitude:this.latitude,
					longitude:this.longitude,
					title: this.userinfo.nickName,
					iconPath:this.userinfo.avatarUrl
				})
				
				wx.startLocationUpdate({
					success: res=> {
						console.log('开启实时位置更新了...');
						wx.onLocationChange(({latitude, longitude}) => {
							if(this.latitude !== latitude || this.longitude !== longitude) {
									console.log('位置发送变化已经为你更新了...');
									// 只需要把位置发出去
									// this.longitude = longitude
									// this.latitude = latitude
									console.log(latitude, longitude);
									socket.emit('location', {
										id:this.id,
										type:this.type,
										width: 30,
										height: 30,
										latitude: latitude,
										longitude:longitude,
										title: this.userinfo.nickName,
										iconPath:this.userinfo.avatarUrl
									})
							}else {
								console.log('位置未发生变化未为你更新...');
							}
						})
					},
					fail() {
						this.message('error', '你已拒绝实时位置共享服务，请授权！')
					}
				})
			},
			async joinRoom() {
				if(Object.keys(this.userinfo).length === 0) {
					let res = await	this.getUserInfo()
					if(!res) {
						this.message('error', '获取个人信息失败')
						
						this.room = null
						this.type = null
						return
					}
				}
				// 连接socket
				const socket = io('https://wuyupei.top:8888', {
				  query: {
					  auth: this.room,
					  type: this.type,
					  id: this.id
				  },
				  transports: [ 'websocket', 'polling' ],
				  timeout: 5000,
				});
				
				Object.defineProperty(this, 'sockets', {
					value: socket,
					configurable : true,
					writable : true,
					enumerable : true,
				})

				socket.on('location',res => {
					res =typeof res === 'string' ? JSON.parse(res) : res
					if(res.type === 'creater') {
						this.users = [res]
					}
				})
				
				// 房间不存在
				socket.on('room-no-exit', (res) => {
					this.message('warning', res.msg)
					
					this.room = null
					
				})
				
				// 房主关闭房间 
				socket.on('homeowner-disconnect', res => {
					this.message('warning', res.msg)
					
					this.sockets.disconnect()
					delete this.sockets
					this.room = null
					this.type = null
					this.users = []
				})
				
				socket.emit('location', {
					id:this.id,
					type:this.type,
					width: 30,
					height: 30,
					latitude:this.latitude,
					longitude:this.longitude,
					title: this.userinfo.nickName,
					iconPath:this.userinfo.avatarUrl
				})
				
				wx.startLocationUpdate({
					success: res=> {
						console.log('开启实时位置更新了...');
						wx.onLocationChange(({latitude, longitude}) => {
							if(this.latitude !== latitude || this.longitude !== longitude) {
									console.log('位置发送变化已经为你更新了...');
									// 不需要更新
									// this.longitude = longitude
									// this.latitude = latitude
									console.log(latitude, longitude);
									socket.emit('location', {
										id:this.id,
										type:this.type,
										width: 30,
										height: 30,
										latitude: latitude,
										longitude: longitude,
										title: this.userinfo.nickName,
										iconPath:this.userinfo.avatarUrl
									})
							}else {
								console.log('位置未发生变化未为你更新...');
							}
						})
					},
					fail() {
						this.message('error', '你已拒绝实时位置共享服务')
					}
				})
			},
			out() {
				if(!this.sockets) return
				this.message('success', '你已退出房间')
				this.sockets.disconnect()
				delete this.sockets
				this.room = null
				this.type = null
				this.users = []
			}
		}
	}
</script>

<style>
	.content {
		width: 100vw;
		height: 100vh;
	}
	.map {
		width: 100%;
		height: 100%;
	}
	.roomNo {
		position: fixed;
		display: flex;
		justify-content: space-around;
		color: aliceblue;
		width: 100%;
		height: auto;
		top: 0;
		left: 0;
		background-color: rgba(0, 0, 0, 0.3);
		padding: 10rpx 0;
		z-index: 99999;
	}
	.roomNo span {
		font-size: 24rpx
	}
	.control {
		position: fixed;
		display: flex;

		justify-content: center;		
		height: 150rpx;
		align-items: center;
		background-color:rgba(0, 0, 0, 0.5);
		bottom: 0;
		left: 0;
		right: 0;
	}
</style>
