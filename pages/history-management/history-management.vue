<template>
	<view class="container">
		<!-- #ifndef H5 --> <statusBar></statusBar> <!-- #endif -->
		<!-- 页面主体 -->
		<view class="history-page">
			<!-- 头部 -->
			<view class="header">
				<image src="/static/icons/back.png" mode="aspectFit" class="back-icon" @click="goBack" />
				<view class="title-group">
					<text class="title">历史记录管理</text>
					<text class="desc">可编辑设备备注或删除历史记录</text>
				</view>
				<text class="clear-btn" @click="clearAllHistory">清空全部</text>
			</view>

			<!-- 空状态 -->
			<view v-if="devices.length === 0" class="empty-state">
				<image src="/static/icons/empty-box.png" mode="aspectFit" class="empty-icon" />
				<br>
				<text class="empty-text">暂无历史记录</text>
			</view>

			<!-- 历史列表 -->
			<scroll-view scroll-y class="device-list" :scroll-with-animation="true">
				<block v-for="(device, index) in devices" :key="index">
					<view class="card">
						<view class="card-body">
							<text class="mac-label">MAC地址：</text>
							<text class="mac-address">{{ device.macAddress }}</text>
						</view>
						<view class="card-footer">
							<text class="remark-text">{{ device.remark || '未填写备注' }}</text>
							<image src="/static/icons/edit.png" mode="aspectFit" class="edit-icon"
								@click="openEditModal(device)" />
							<image src="/static/icons/delete.png" mode="aspectFit" class="delete-icon"
								@click="deleteDevice(index)" />
						</view>
					</view>
				</block>
			</scroll-view>

			<!-- 编辑模态框 -->
			<view v-if="isEditModalVisible" class="modal-mask" @click.self="closeEditModal">
				<view class="modal-content">
					<text class="modal-title">编辑备注</text>
					<input 
						type="text" 
						v-model="currentDevice.remark" 
						placeholder="请输入备注（如：办公室主机）"
						class="remark-input-modal"
						@input="onRemarkInput"
						focus
					/>
					<button class="save-btn" @click="saveRemark(currentDevice)">保存</button>
					<button class="cancel-btn" @click="closeEditModal">取消</button>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			statusBarHeight: 0,
			devices: [],
			isEditModalVisible: false,
			currentDevice: {},
			hasUnsavedChanges: false // 是否有未保存的修改
		};
	},
	onLoad() {
		const systemInfo = uni.getSystemInfoSync();
		this.statusBarHeight = systemInfo.statusBarHeight;

		const history = uni.getStorageSync('wolHistory') || [];
		this.devices = history.map(device => ({
			...device,
			remark: device.remark || ''
		}));
	},
	methods: {
		openEditModal(device) {
			this.currentDevice = { ...device };
			this.isEditModalVisible = true;
		},

		closeEditModal() {
			this.isEditModalVisible = false;
			this.hasUnsavedChanges = false;
		},

		onRemarkInput() {
			this.hasUnsavedChanges = true;
		},

		saveRemark(device) {
			const storedHistory = uni.getStorageSync('wolHistory') || [];
			const updatedHistory = storedHistory.map(h => {
				if (h.macAddress === device.macAddress) {
					return { ...h, remark: device.remark };
				}
				return h;
			});
			uni.setStorageSync('wolHistory', updatedHistory);
			this.closeEditModal();

			this.devices = this.devices.map(d => {
				if (d.macAddress === device.macAddress) {
					return { ...d, remark: device.remark };
				}
				return d;
			});

			uni.showToast({ title: '已保存备注' });
		},

		deleteDevice(index) {
			uni.showModal({
				title: '确认删除',
				content: `是否删除该 MAC 地址：${this.devices[index].macAddress}？`,
				success: (res) => {
					if (res.confirm) {
						this.devices.splice(index, 1);
						uni.setStorageSync('wolHistory', this.devices);
						uni.showToast({ title: '删除成功' });
					}
				}
			});
		},

		clearAllHistory() {
			uni.showModal({
				title: '确认清空',
				content: '确定要清空所有历史记录吗？此操作不可恢复。',
				success: (res) => {
					if (res.confirm) {
						uni.setStorageSync('wolHistory', []);
						this.devices = [];
						uni.showToast({ title: '已清空' });
					}
				}
			});
		},

		goBack() {
			if (this.hasUnsavedChanges) {
				uni.showModal({
					title: '注意',
					content: '您有未保存的修改，确定要离开吗？',
					success: (res) => {
						if (res.confirm) {
							uni.navigateBack();
						}
					}
				});
			} else {
				uni.navigateBack();
			}
		}
	}
};
</script>

<style>
/* 全局容器 */
.container {
	min-height: 100vh;
	background-color: #f5f5f5;
}

/* 状态栏占位 */
.status-bar-placeholder {
	width: 100%;
	background-color: transparent;
}

/* 页面主区域 */
.history-page {
	padding: 20rpx;
	box-sizing: border-box;
	flex-direction: column;
	display: flex;
	height: 100%;
}

/* 头部样式 */
.header {
	display: flex;
	align-items: center;
	margin-bottom: 30rpx;
}

.back-icon {
	width: 40rpx;
	height: 40rpx;
	margin-right: 20rpx;
}

.title-group {
	display: flex;
	flex-direction: column;
	margin-right: auto;
}

.title {
	font-size: 36rpx;
	font-weight: bold;
	color: #333;
}

.desc {
	font-size: 26rpx;
	color: #666;
	margin-top: 8rpx;
}

.clear-btn {
	font-size: 26rpx;
	padding: 12rpx 24rpx;
	border-radius: 40rpx;
	background-color: #f1f1f1;
	color: #333;
}

.clear-btn:hover {
	background-color: #e0e0e0;
}

/* 空状态 */
.empty-state {
	flex-direction: column;
	align-items: center;
	justify-content: center;
	padding: 100rpx 0;
	text-align: center;
}

.empty-icon {
	width: 160rpx;
	height: 160rpx;
	margin-bottom: 20rpx;
}

.empty-text {
	font-size: 28rpx;
	color: #999;
}

/* 滚动列表 */
.device-list {
	flex: 1;
	max-height: 80vh;
	overflow-y: auto;
}

.card {
	background-color: #fff;
	border-radius: 16rpx;
	padding: 30rpx;
	margin-bottom: 20rpx;
	box-shadow: 0 4rpx 16rpx rgba(0, 0, 0, 0.05);
	transition: all 0.3s ease-in-out;
}

.card:hover {
	transform: translateY(-4rpx);
	box-shadow: 0 8rpx 20rpx rgba(0, 0, 0, 0.1);
}

.card-body {
	margin-bottom: 20rpx;
}

.mac-label {
	font-size: 26rpx;
	color: #666;
}

.mac-address {
	font-size: 30rpx;
	font-weight: bold;
	color: #333;
}

.card-footer {
	display: flex;
	justify-content: space-between;
	align-items: center;
}

.remark-text {
	font-size: 26rpx;
	color: #888;
	flex: 1;
}

.edit-icon,
.delete-icon {
	width: 36rpx;
	height: 36rpx;
	margin-left: 20rpx;
}

/* 模态框 */
.modal-mask {
	position: fixed;
	top: 0;
	left: 0;
	right: 0;
	bottom: 0;
	background-color: rgba(0, 0, 0, 0.5);
	display: flex;
	justify-content: center;
	align-items: center;
	z-index: 999;
}

.modal-content {
	background-color: #fff;
	padding: 40rpx;
	border-radius: 20rpx;
	width: 80%;
	max-width: 600rpx;
	z-index: 1001;
	position: relative;
}

.modal-title {
	font-size: 32rpx;
	font-weight: bold;
	color: #333;
	margin-bottom: 30rpx;
	text-align: center;
}

.remark-input-modal {
	width: 100%;
	min-height: 80rpx;
	padding: 24rpx 30rpx;
	font-size: 28rpx;
	border: 1rpx solid #ddd;
	border-radius: 12rpx;
	margin-bottom: 20rpx;
	box-sizing: border-box;
	z-index: 1002;
	position: relative;
}

.save-btn,
.cancel-btn {
	width: 100%;
	font-size: 28rpx;
	padding: 24rpx 30rpx;
	margin-top: 10rpx;
	border-radius: 12rpx;
	box-sizing: border-box;
}

.save-btn {
	background-color: #4caf50;
	color: #fff;
}

.cancel-btn {
	background-color: #f1f1f1;
	color: #333;
}
</style>
