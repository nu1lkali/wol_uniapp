<template>
	<view class="container">
		<!-- #ifndef H5 --> <statusBar></statusBar> <!-- #endif -->
		<scroll-view scroll-y class="content-scroll">
			<!-- Header -->
			<view class="header">
				<text class="title">WOL 远程唤醒</text>
				<text class="subtitle">Wake-on-LAN 工具</text>
			</view>

			<!-- 更多按钮 -->
			<view class="menu-trigger" @click="toggleMenu">
				<image src="/static/icons/more.png" mode="aspectFit" class="icon-more" />
			</view>

			<!-- 弹出菜单 -->
			<view v-if="isMenuVisible" class="popup-menu" :style="{ top: menuTop + 'px', right: menuRight + 'px' }">
				<view class="menu-item" @click="goToHistoryManagement">历史记录管理</view>
			</view>

			<!-- 新手引导 -->
			<view v-if="showGuide" class="guide-box">
				<text class="guide-title">💡 使用提示</text>
				<text class="guide-text">接口地址格式：http://xxx.xxx.x.x:7690/wol</text>
				<text class="guide-text">MAC 地址格式：XX-XX-XX-XX-XX-XX</text>
				<button class="guide-btn" @click="hideGuide">我知道了</button>
			</view>

			<!-- 表单区域 -->
			<view class="form-section">
				<text class="section-title" style="padding-bottom: 10px;">🔧 配置信息</text>

				<!-- 历史记录选择器 -->
				<picker mode="selector" range-key="macAddress" :range="historyListLabels" @change="onSelectHistory"
					class="select-box" style="margin-top: 20rpx;">
					<view class="select-content">
						<text v-if="selectedHistoryIndex >= 0">{{ selectedHistoryText }}</text>
						<text v-else style="color: #999;">请选择历史记录</text>
						<image src="/static/icons/down-arrow.png" mode="aspectFit" class="arrow-icon" />
					</view>
				</picker>

				<!-- WOL 接口地址 -->
				<view class="input-group">
					<text class="input-label">WOL 接口地址</text>
					<input type="text" v-model="wolUrl" placeholder="请输入WOL接口地址" class="input-field" />
				</view>

				<!-- MAC 地址 -->
				<view class="input-group">
					<text class="input-label">目标设备 MAC </text>
					<input type="text" v-model="macAddress" placeholder="例如：6C-7F-12-18-15-1F"
						class="input-field" />
				</view>

				<!-- 提交按钮 -->
				<button class="submit-btn" @click="sendWolRequest">唤醒设备</button>
			</view>

			<!-- 响应结果 -->
			<view v-if="responseMessage" class="response-section"
				:class="{ success: !isErrorResponse, error: isErrorResponse }">
				<text class="response-line">{{ responseLine1 }}</text>
				<text class="response-line">{{ responseLine2 }}</text>
			</view>

			<!-- 请求日志 -->
			<view class="log-section">
				<text class="section-title">📜 请求日志</text>
				<view v-if="logList.length === 0" class="log-empty">
					<text>暂无操作记录</text>
				</view>
				<view v-for="(log, index) in logList.slice(0, 10)" :key="index" class="log-item"
					:class="{ success: log.success, error: !log.success }" @click="fillFromLog(log)">
					<text class="log-time">{{ log.time }}</text>
					<text class="log-mac">{{ log.mac }}</text>
					<text class="log-status">{{ log.success ? '✅ 成功' : '❌ 失败' }}</text>
				</view>
			</view>
		</scroll-view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			wolUrl: '',
			macAddress: '',
			responseMessage: '',
			isErrorResponse: false,
			showGuide: true,
			historyList: [],
			historyListLabels: [],
			selectedHistoryIndex: -1,
			logList: [],
			isMenuVisible: false,
			menuTop: 0,
			menuRight: 0,
			statusBarHeight: 0
		};
	},
	computed: {
		selectedHistoryText() {
			return this.selectedHistoryIndex >= 0
				? this.historyList[this.selectedHistoryIndex]?.macAddress
				: '';
		},
		responseLine1() {
			if (!this.responseMessage) return '';
			return this.isErrorResponse ? '错误发生：\n' : '✅ 操作成功：\n';
		},
		responseLine2() {
			if (!this.responseMessage) return '';
			const parts = this.responseMessage.split('：');
			return parts[1] || parts[0];
		}
	},
	onLoad() {
		const systemInfo = uni.getSystemInfoSync();
		this.statusBarHeight = systemInfo.statusBarHeight;
	},
	mounted() {
		this.initDefaultValues();
		this.loadHistory();
		this.loadLogs();
		this.showGuide = !uni.getStorageSync('hasViewedGuide');
	},
	methods: {
		hideGuide() {
			uni.setStorageSync('hasViewedGuide', true);
			this.showGuide = false;
		},
		initDefaultValues() {
			this.wolUrl = 'http://192.168.2.124:7690/wol';
			this.macAddress = '6C-7F-12-18-15-1F';
		},
		onSelectHistory(e) {
			const index = e.detail.value;
			this.selectedHistoryIndex = index;
			if (index >= 0 && this.historyList[index]) {
				this.wolUrl = this.historyList[index].wolUrl || '';
				this.macAddress = this.historyList[index].macAddress || '';
			}
		},
		validateMac(mac) {
			const reg = /^([0-9A-Fa-f]{2}[:-]){5}([0-9A-Fa-f]{2})$/;
			return reg.test(mac);
		},
		saveToHistory() {
			let list = uni.getStorageSync('wolHistory') || [];
			const exists = list.some(item => item.macAddress === this.macAddress && item.wolUrl === this.wolUrl);
			if (!exists) {
				list.unshift({
					wolUrl: this.wolUrl,
					macAddress: this.macAddress
				});
				uni.setStorageSync('wolHistory', list.slice(0, 10));
			}
		},
		loadHistory() {
			this.historyList = uni.getStorageSync('wolHistory') || [];
			this.historyListLabels = this.historyList.map(h => h.macAddress);
		},
		addLog(success, message) {
			const time = new Date().toLocaleString();
			const logEntry = {
				time,
				url: this.wolUrl,
				mac: this.macAddress,
				success,
				message
			};

			let logs = uni.getStorageSync('wolLogs') || [];
			logs.unshift(logEntry);
			uni.setStorageSync('wolLogs', logs.slice(0, 30));
			this.loadLogs();
		},
		loadLogs() {
			this.logList = uni.getStorageSync('wolLogs') || [];
		},
		fillFromLog(log) {
			this.wolUrl = log.url;
			this.macAddress = log.mac;
		},
		sendWolRequest() {
			if (!this.wolUrl || !this.macAddress) {
				uni.showToast({ title: '请填写完整信息', icon: 'none' });
				return;
			}

			if (!this.validateMac(this.macAddress)) {
				uni.showToast({ title: 'MAC 地址格式错误', icon: 'none' });
				return;
			}

			this.saveToHistory();

			uni.showLoading({ title: '正在发送指令...' });

			uni.request({
				url: `${this.wolUrl}?mac=${this.macAddress}`,
				method: 'GET',
				success: (res) => {
					uni.hideLoading();
					if (res.data && res.data.error === 0) {
						this.responseMessage = `操作成功：${res.data.message}`;
						this.isErrorResponse = false;
						this.addLog(true, res.data.message);
					} else {
						this.responseMessage = `错误发生：${res.data?.message || '未知错误'}`;
						this.isErrorResponse = true;
						this.addLog(false, res.data?.message || '未知错误');
					}
				},
				fail: () => {
					uni.hideLoading();
					this.responseMessage = '请求失败，请检查网络连接和地址是否正确';
					this.isErrorResponse = true;
					this.addLog(false, '请求失败，请检查网络连接和地址是否正确');
				}
			});
		},
		toggleMenu() {
			const query = uni.createSelectorQuery();
			query.select('.menu-trigger').boundingClientRect(res => {
				if (res) {
					this.menuTop = res.top + res.height;
					this.menuRight = uni.getSystemInfoSync().windowWidth - res.right;
					this.isMenuVisible = !this.isMenuVisible;
				}
			}).exec();
		},
		goToHistoryManagement() {
			uni.navigateTo({
				url: '/pages/history-management/history-management'
			});
			this.isMenuVisible = false;
		}
	}
};
</script>

<style>
.container {
	min-height: 100vh;
	background-color: #f9f9f9;
	display: flex;
	flex-direction: column;
}

.status-bar-placeholder {
	width: 100%;
	background-color: transparent;
}

.content-scroll {
	flex: 1;
	padding-bottom: 40rpx;
}

.header {
	margin-bottom: 60rpx;
	padding-top: 20rpx;
	padding-left: 40rpx;
	padding-right: 40rpx;
}

.title {
	font-size: 36rpx;
	font-weight: bold;
	color: #333;
}

.subtitle {
	font-size: 28rpx;
	color: #666;
	display: block;
	margin-top: 10rpx;
}

.menu-trigger {
	position: absolute;
	top: var(--status-bar-height);
	right: 40rpx;
	width: 60rpx;
	height: 60rpx;
	z-index: 10;
}

.icon-more {
	width: 100%;
	height: 100%;
}

.popup-menu {
	position: absolute;
	z-index: 999;
	background-color: #fff;
	border-radius: 10rpx;
	box-shadow: 0 2rpx 10rpx rgba(0, 0, 0, 0.1);
	min-width: 200rpx;
}

.menu-item {
	padding: 20rpx 30rpx;
	font-size: 28rpx;
	color: #333;
}

.guide-box {
	background-color: #e8f4ff;
	color: #333;
	padding: 30rpx;
	margin: 0 40rpx 40rpx;
	border-radius: 16rpx;
}

.guide-title {
	font-size: 30rpx;
	font-weight: bold;
	display: block;
	margin-bottom: 20rpx;
}

.guide-text {
	font-size: 26rpx;
	color: #555;
	line-height: 40rpx;
	display: block;
	margin-bottom: 10rpx;
}

.guide-btn {
	margin-top: 30rpx;
	font-size: 28rpx;
	background-color: #4caaff;
	color: #fff;
}

.form-section {
	padding: 0 40rpx 40rpx;
}

.select-box {
	border: 1px solid #ccc;
	border-radius: 10rpx;
	padding: 20rpx;
	margin-bottom: 30rpx;
	background-color: #fff;
	display: flex;
	align-items: center;
}

.select-content {
	flex: 1;
	display: flex;
	justify-content: space-between;
	align-items: center;
}

.arrow-icon {
	width: 30rpx;
	height: 30rpx;
}

.input-group {
	margin-bottom: 30rpx;
	display: flex;
	align-items: center;
}

.input-label {
	font-size: 24rpx;
	color: #333;
	flex-basis: 180rpx;
	flex-shrink: 0;
}

.input-field {
	border: 1px solid #ccc;
	border-radius: 10rpx;
	padding: 20rpx;
	font-size: 24rpx;
	background-color: #fff;
	flex-grow: 1;
	margin-left: 20rpx;
}

.submit-btn {
	background-color: #4caaff;
	color: #fff;
	font-size: 30rpx;
	margin-top: 40rpx;
	width: 100%;
}

.response-section {
	padding: 30rpx;
	margin: 0 40rpx 40rpx;
	border-radius: 16rpx;
	font-size: 28rpx;
	line-height: 40rpx;
}

.response-section.success {
	background-color: #e6f7ff;
	color: #1890ff;
}

.response-section.error {
	background-color: #fff1f0;
	color: #ff4d4f;
}

.log-section {
	padding: 0 40rpx;
}

.log-empty {
	font-size: 28rpx;
	color: #999;
	display: block;
	text-align: center;
	margin: 40rpx 0;
}

.log-item {
	background-color: #fff;
	border: 1px solid #eee;
	padding: 20rpx;
	margin-bottom: 10rpx;
	margin-top: 20rpx;
	border-radius: 16rpx;
	display: flex;
	justify-content: space-between;
	align-items: center;
}

.log-item.success {
	border-left: 6rpx solid #52c41a;
}

.log-item.error {
	border-left: 6rpx solid #ff4d4f;
}

.log-time {
	font-size: 24rpx;
	color: #888;
	flex: 1;
}

.log-mac {
	font-size: 28rpx;
	color: #333;
	flex: 2;
	text-align: center;
}

.log-status {
	font-size: 28rpx;
	color: #333;
	flex: 1;
}
</style>
