<template>
	<view class="content">
		<!--  -->
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
			<span>版本: 1.1.2</span>
		</view>
		<map 
		name="map" 
		class="map" 
		show-location="true" 
		enable-3D="ture" 
		isHighAccuracy="true" 
		enable-indoorMap="ture"
		:latitude="latitude"
		:longitude="longitude"
		:markers="users">
			<cover-view slot="callout">
				<template v-for="item in users">
					<cover-view :marker-id="item.id">
						<cover-image :src="item.img" class="cover-image" :key="item.img"></cover-image>
					</cover-view>
				</template>
			  </cover-view>
		</map>
		
		<view :class="{menu:true, show: isShowControl}">
			<view @click="create">创建</view>
			<view @click="join">加入</view>
			<view @click="out">退出</view>
		</view>
		<view :class="{control: true}" @click="isShowControl = !isShowControl">
			<img :class="{rotate: isShowControl}" src="../../static/add.png" alt="">
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
				relongitude:'',
				relatitude:'',
				zhizhen:0,
				type:null,
				room:null,
				msgType:'success',
				messageText:'',
				users:[],
				userinfo:{},
				socket:'',
				id:Math.floor(Math.random()*1000),
				isShowControl: false
			}
		},
		onLoad() {
			const { locationEnabled } = wx.getSystemInfoSync()
			wx.getSetting({
			  success: (res) => {
			    if (!res.authSetting['scope.userLocation']) {
			      wx.authorize({
			        scope: 'scope.userLocation',
			        success: ()  => {
						if(!locationEnabled) {
							this.message('success', '位置服务未开启,请开启后重新进入小程序')
							return
						} 
			          // 获取位置信息
			          this.getLocation().then(res => {
			          	this.latitude = res[1]
						this.longitude = res[0]
						this.relatitude =  res[1]
						this.relongitude = res[0]
			          })
			        }
			      })
			    }else {
					if(!locationEnabled) {
						this.message('success', '位置服务未开启,请开启后重新进入小程序')
						return
					} 
					// 获取位置信息
					this.getLocation().then(res => {
						this.latitude = res[1]
						this.longitude = res[0]
						this.relatitude =  res[1]
						this.relongitude = res[0]
					})
				}
			  }
			})

		},
		onShow() {
			if(this.room && this.type == 'creater') {
				
				this.createRoom()
			}else if(this.room && this.type == 'joiner') {
				
				this.joinRoom()
			}
		},
		methods: {
			hidden() {
				this.isShowControl = !this.isShowControl
			}, 
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
							console.log(res, '.....');
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
							
							this.userinfo = res.userInfo
							// 绘制图像
							// this.getImage(res.userInfo.avatarUrl)
							resolve(res)
						},
						fail(err) {
							reject(err)
						}
					})
				})
			},
			
			create() {
				this.hidden()
				if(this.isInRoom()) {
					this.message('error', '已经在房间里了!')
					return
				}
				this.type = 'creater'
				this.inputDialogToggle()
			},
			join() {
				this.hidden()
				if(this.isInRoom()) {
					this.message('error', '已经在房间里了!')
					return
				}
				this.type = 'joiner'
				this.inputDialogToggle()
			},
			getImage(url) {
				const query = wx.createSelectorQuery()
				    query.select('#myCanvas')
				      .fields({ node: true, size: true })
				      .exec((res) => {
				        const canvas = res[0].node
				        const ctx = canvas.getContext('2d')
						
				        const dpr = wx.getSystemInfoSync().pixelRatio
				        canvas.width = res[0].width * dpr
				        canvas.height = res[0].height * dpr
				        ctx.scale(dpr, dpr)
				
				        const image = canvas.createImage()
						const marker = canvas.createImage()
						
						image.src = url
						marker.src = 'https://wuyupei.top:8888/upload/02b8136651d753b7a551e1000.png'
						
						
						// ctx.drawImage(image, 0, 0)
						ctx.drawImage(marker, 0, 0)
						
						wx.canvasToTempFilePath({
							  x: 0,
							  y: 0,
							  width: 150,
							  height: 150,
							  destWidth: 100,
							  destHeight: 100,
							  canvas: canvas,
							  success(res) {
								console.log(res.tempFilePath)
							  }
						})
				      })
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
					console.log(this.users);
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
					id:this.id,img:this.userinfo.avatarUrl,
					type:this.type,
					width: 16,
					customCallout:{display:'ALWAYS', anchorX:0, anchorY:0},
					height: 16,
					anchor: {
						x: 0.5,
						y: 0.5
					},
					rotate: this.zhizhen,
					latitude:this.relatitude,
					longitude:this.relongitude,
					title: this.userinfo.nickName,
					iconPath:"../../static/location.png"
				})
				
				wx.startDeviceMotionListening({
					interval: 'normal',
					success: res => {
						wx.onDeviceMotionChange(res => {
							if(Math.abs(this.zhizhen - res.alpha) < 30) {
								return
							}else {
								console.log('发送了位置');
								this.zhizhen = res.alpha
								socket.emit('location', {
									id:this.id,
									img:this.userinfo.avatarUrl,
									type:this.type,
									width: 16,
									customCallout:{display:'ALWAYS', anchorX:0, anchorY:0},
									height: 16,
									anchor: {
										x: 0.5,
										y: 0.5
									},
									rotate: res.alpha <= 180 ?  res.alpha + 180 : res.alpha - 180,
									latitude: this.relatitude,
									longitude: this.relongitude,
									title: this.userinfo.nickName,
									iconPath:"../../static/location.png"
								})
							}
						})
					},
					fail: err => {
						
					}
				})
				
				wx.startLocationUpdate({
					success: res=> {
						
						wx.onLocationChange(({latitude, longitude}) => {
							if(Math.abs(this.relatitude - latitude) > 0.0005 || Math.abs(this.relongitude !== longitude) > 0.0005) {
									console.log(this.relongitude,latitude);
									// 优化部分
									this.relongitude = longitude
									this.relatitude = latitude
									 
									socket.emit('location', {
										id:this.id,
										img:this.userinfo.avatarUrl,
										type:this.type,
										width: 16,
										customCallout:{display:'ALWAYS', anchorX:0, anchorY:0},
										height: 16,
										anchor: {
											x: 0.5,
											y: 0.5
										},
										rotate: this.zhizhen,
										latitude: latitude,
										longitude: longitude,
										title: this.userinfo.nickName,
										iconPath:"../../static/location.png"
									})
							}else {
								
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
					res = typeof res === 'string' ? JSON.parse(res) : res
					console.log('接受到位置');
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
					img:this.userinfo.avatarUrl,
					type:this.type,
					width: 16,
					customCallout:{display:'ALWAYS', anchorX:0, anchorY:0},
					height: 16,
					anchor: {
						x: 0.5,
						y: 0.5
					},
					rotate:this.zhizhen,
					latitude:this.relatitude,
					longitude:this.relongitude,
					title: this.userinfo.nickName,
					iconPath:"../../static/location.png"
				})
				
				wx.startDeviceMotionListening({
					interval: 'normal',
					success: res => {
						wx.onDeviceMotionChange(res => {
							if(Math.abs(this.zhizhen - res.alpha) < 30) {
								return
							}else {
								console.log('发送了位置');
								this.zhizhen = res.alpha
								socket.emit('location', {
									id:this.id,
									img:this.userinfo.avatarUrl,
									type:this.type,
									width: 16,
									customCallout:{display:'ALWAYS', anchorX:0, anchorY:0},
									height: 16,
									anchor: {
										x: 0.5,
										y: 0.5
									},
									rotate: res.alpha <= 180 ?  res.alpha + 180 : res.alpha - 180,
									latitude: this.relatitude,
									longitude: this.relongitude,
									title: this.userinfo.nickName,
									iconPath:"../../static/location.png"
								})
							}
						})
					},
					fail: err => {
						
					}
				})
				
				wx.startLocationUpdate({
					success: res=> {
						
						wx.onLocationChange(({latitude, longitude}) => {
							if(Math.abs(this.relatitude - latitude) > 0.0005 || Math.abs(this.relongitude !== longitude) > 0.0005) {
									
									// 优化部分
									this.relongitude = longitude
									this.relatitude = latitude
									socket.emit('location', {
										id:this.id,
										img:this.userinfo.avatarUrl,
										type:this.type,
										width: 16,
										customCallout:{display:'ALWAYS', anchorX:0, anchorY:0},
										height: 16,
										anchor: {
											x: 0.5,
											y: 0.5
										},
										rotate: this.zhizhen,
										latitude: latitude,
										longitude: longitude,
										title: this.userinfo.nickName,
										iconPath:"../../static/location.png"
									})
							}else {
								
							}
						})
					},
					fail() {
						this.message('error', '你已拒绝实时位置共享服务')
					}
				})
			},
			out() {
				this.hidden()
				wx.stopLocationUpdate()
				wx.stopDeviceMotionListening()
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
		margin: 0;
		padding: 0;
		justify-content: space-around;
		color: aliceblue;
		width: calc(100% - 190rpx);
		text-align: center;
		line-height: 100rpx;
		border-radius: 20rpx;
		height: 100rpx;
		bottom: 50rpx;
		left: 160rpx;
		background-color: #fff;
		z-index: 2;
	}
	.roomNo span {
		font-size: 20rpx;
		color: black;
		margin: 0;
		padding: 0;
	}
	.menu {
		position: fixed;
		bottom: 130rpx;
		left: 30rpx;
		width: 100rpx;
		height: 400rpx;
		transition: all linear 0.3s;
		opacity: 0;
	}
	.menu view {
		text-align: center;
		line-height: 100rpx;
		width: 100rpx;
		height: 100rpx;
		background-color: #fff;
		margin: 20rpx 0;
		border-radius: 20rpx;
	}
	.show {
		opacity: 1;
	}
	.control {
		display: flex;
		position: fixed;
		justify-content: center;
		align-items: center;
		left: 30rpx;
		bottom: 50rpx;
		width: 100rpx;
		height: 100rpx;
		border-radius: 20rpx;
		align-items: center;
		background-color:rgba(255, 255, 255);
		transition: all linear 0.2s;
	}
	.control image {
		width: 60rpx;
		height: 60rpx;
		transition: all linear 0.2s;
	}
	.rotate {
		transform: rotateZ(45deg);
	}
	.cover-image {
		width: 30px;
		height: 30px;
		border-radius: 50%;
	}
</style>
