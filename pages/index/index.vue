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
			房间：{{room}} <br/>
							<br/>
			人数：{{users.length + 1}}
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
				type:'',
				room:null,
				msgType:'success',
				messageText:'',
				users:[
					
				],
				userinfo:{},
				socket:null,
				id:''
			}
		},
		onLoad() {
			this.getLocation().then(res => {
				this.longitude = res[0]
				this.latitude = res[1]
			})
		},
		methods: {
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
					if(!Object.keys(this.userinfo).length === 0) {
						return;
					}
					// 获取个人信息
					wx.getUserProfile({
						desc: '用于完善会员资料', 
						success: (res) => {
							console.log(res);
							this.userinfo = res.userInfo
							resolve()
						},
						fail(err) {
							console.log(err);
							reject(err)
						}
					})
				})
			},
			
			create() {
				if(this.isInRoom()) {
					console.log('已经在房间里了！');
					return
				}
				this.type = 'creater'
				this.inputDialogToggle()
			},
			join() {
				if(this.isInRoom()) {
					console.log('已经在房间里了！');
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
			    await	this.getUserInfo()
				this.id = Math.floor(Math.random() * 1000)
				// 连接socket
			 	const socket =  io('https://wuyupei.top:8888', {
				  query: {
					  auth: this.room,
					  type: this.type
				  },
				  transports: [ 'websocket', 'polling' ],
				  timeout: 5000,
				});
				
				socket.on('location',res => {
					res =typeof res === 'string' ? JSON.parse(res) : res
					if(res.type === 'creater') return
					const result =	this.users.findIndex((item, index) => item.id == res.id) 
					
					if(result == -1) {
						this.users.push(res)
					}else {
						this.users.splice(result, 1)
						this.users.push(res)
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
				
				wx.onLocationChange(({latitude, longitude}) => {
					this.longitude = latitude
					this.latitude = longitude
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
				})
			},
			async joinRoom() {
				await	this.getUserInfo()
				this.id = Math.floor(Math.random() * 1000)
				// 连接socket
				const socket = io('https://wuyupei.top:8888', {
				  query: {
					  auth: this.room,
					  type: this.type
				  },
				  transports: [ 'websocket', 'polling' ],
				  timeout: 5000,
				});
				
				socket.on('location',res => {
					res =typeof res === 'string' ? JSON.parse(res) : res
					if(res.type === 'creater') {
						this.users = [res]
					}
				})
				
				// 房间不存在
				socket.on('room-no-exit', (res) => {
					console.log(res);
					this.msgType = 'error'
					this.messageText = res.msg
					this.$refs.message.open()
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
				
				wx.onLocationChange(({latitude, longitude}) => {
						this.longitude = latitude
						this.latitude = longitude
						socket.emit('location', {
							id:this.id,
							type:this.type,
							width: 50,
							height: 50,
							latitude:this.latitude,
							longitude:this.longitude,
							title: this.userinfo.nickName,
							iconPath:this.userinfo.avatarUrl
						})
				})
			},
			out() {
				console.log('退出');
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
		top: 0;
		left: 0;
		background-color: transparent;
		z-index: 99999;
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
