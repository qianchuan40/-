<template>
	<!-- 整体容器：左侧导航 + 右侧内容 -->
	<view class="layout-container">
		<!-- 左侧机房导航栏 -->
		<view class="sidebar">
			<view class="sidebar-title">IDC智能监控系统</view>
			<view v-for="(room, index) in roomList" :key="index" class="sidebar-item"
				:class="{ active: currentPage === room.page }" @click="goToPage(room.page)">
				{{ room.name }}
			</view>
		</view>

		<!-- 右侧内容区 -->
		<view class="main-content">
			<!-- 未选中机房时：空白提示 -->
			<view v-if="currentPage === 'home'" class="blank-page">
				请选择左侧机房查看详情
			</view>

			<!-- 选中机房时：显示机房内容 -->
			<view v-else class="room-content">
				<!-- 机房标题 + 返回按钮 + 当前时间 -->
				<view class="room-header">
					<text class="room-title">{{ currentRoomConfig.title }}</text>
					<view class="header-right">
						<button class="back-btn" @click="goToPage('home')">返回主界面</button>
						<text class="current-time">{{ currentTime }}</text>
					</view>
				</view>

				<!-- 多数据报警提示 -->
				<view class="alarms-container">
					<view v-for="(dataType, typeIndex) in dataTypes" :key="typeIndex" class="data-alarm"
						:class="getAlarmClass(currentPage, dataType)">
						{{ getAlarmText(currentPage, dataType) }}
					</view>
				</view>

				<!-- 设备控制 + 状态展示 -->
				<view class="content-container">
					<!-- 控制按钮 -->
					<view class="button-container">
						<view v-for="(device, deviceIndex) in safeGetRoomDevices(currentPage)" :key="deviceIndex">
							<button v-for="(cmd, cmdIndex) in commands" :key="cmdIndex"
								@click="sendCommand(cmd, device)">
								控制{{ device }}的{{ cmd === 'on' ? 'LED开' : 'LED关' }}
							</button>
						</view>
					</view>

					<!-- 设备状态 -->
					<view class="status-container">
						<view v-for="(isOnline, name) in safeGetRoomData(currentPage).deviceOnline" :key="name">
							<view class="status-icon"
								:style="isOnline ? 'background-color: #4CAF50;' : 'background-color: #f44336;'"></view>
							<view class="status-text">{{ name }}: {{ isOnline ? '设备在线' : '设备离线' }}</view>
						</view>
					</view>

					<!-- 硬件数据展示 -->
					<view class="title">硬件数据展示</view>
					<view class="data-table" v-if="safeGetRoomData(currentPage).dataHeaders.length > 0">
						<view class="table-row">
							<view class="data-name">数据名称</view>
							<view v-for="device in safeGetRoomDevices(currentPage)" :key="device" class="data-value">
								{{ device }}数值
							</view>
						</view>
						<view class="table-row" v-for="(header, index) in safeGetRoomData(currentPage).dataHeaders"
							:key="index">
							<view class="data-name">{{ formatHeaderName(header) }}</view>
							<view v-for="device in safeGetRoomDevices(currentPage)" :key="device" class="data-value">
								{{ formatValue(safeGetRoomData(currentPage).deviceData[device]?.[header], header) }}
							</view>
						</view>
					</view>

					<!-- 折线图 -->
					<view v-for="(header, index) in safeGetChartHeaders(currentPage)" :key="index">
						<view class="chart-title">{{ formatHeaderName(header) }}折线图</view>
						<view :id="`${currentPage}-${header}-chart`" class="chart-container"></view>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	import * as echarts from 'echarts';

	export default {
		data() {
			return {
				// 左侧机房列表配置（可扩展）
				roomList: [{
						name: '机房1',
						page: 'page1'
					},
					{
						name: '机房2',
						page: 'page2'
					},
					{
						name: '机房3',
						page: 'page3'
					}
				],
				currentPage: 'home', // 默认未选中机房
				roomConfig: {
					page1: {
						devices: ['device1'],
						title: '机房1'
					},
					page2: {
						devices: ['device2'],
						title: '机房2'
					},
					page3: {
						devices: ['device3'],
						title: '机房3'
					}
				},
				currentTime: '', // 新增：存储当前时间
				timeTimer: null, // 新增：时间更新定时器（用于页面卸载时清除）
				// 多数据类型阈值配置
				thresholds: {
					temperature: {
						min: 25,
						max: 40,
						label: '温度',
						unit: '℃',
						normalColor: '#e8f5e9',
						normalText: '#2e7d32',
						alarmColor: '#ffebee',
						alarmText: '#b71c1c'
					},
					humidity: {
						min: 40,
						max: 60,
						label: '湿度',
						unit: '%',
						normalColor: '#e3f2fd',
						normalText: '#0d47a1',
						alarmColor: '#fff3e0',
						alarmText: '#b71c1c'
					},
					lightIntensity: {
						min: 2000,
						max: 8000,
						label: '光照强度',
						unit: 'lx',
						normalColor: '#e8f5e9',
						normalText: '#2e7d32',
						alarmColor: '#ffebee',
						alarmText: '#b71c1c'
					},
					voltage: {
						min: 19,
						max: 23,
						label: '电压',
						unit: 'V',
						normalColor: '#e8f5e9',
						normalText: '#2e7d32',
						alarmColor: '#ffebee',
						alarmText: '#b71c1c'
					},
					current: {
						min: 0,
						max: 10,
						label: '电流',
						unit: 'A',
						normalColor: '#e8f5e9',
						normalText: '#2e7d32',
						alarmColor: '#ffebee',
						alarmText: '#b71c1c'
					}
				},
				commands: ['on', 'off'],
				webSocketUrl: 'ws://192.168.1.104:3000',
				roomData: {
					page1: {
						deviceOnline: {},
						dataHeaders: [],
						deviceData: {},
						historyData: {},
						timeStamps: []
					},
					page2: {
						deviceOnline: {},
						dataHeaders: [],
						deviceData: {},
						historyData: {},
						timeStamps: []
					},
					page3: {
						deviceOnline: {},
						dataHeaders: [],
						deviceData: {},
						historyData: {},
						timeStamps: []
					}
				},
				charts: {},
				socket: null,
				reconnectTimer: null,
				reconnectInterval: 5000,
				maxDataPoints: 30,
				dataTypes: ['temperature', 'humidity', 'lightIntensity', 'voltage', 'current'] // 提取通用数据类型
			};
		},
		computed: {
			// 当前选中机房的配置（方便右侧内容区使用）
			currentRoomConfig() {
				return this.roomConfig[this.currentPage] || {
					title: ''
				};
			}
		},
		onLoad() {
			// 初始化所有机房数据结构
			Object.keys(this.roomConfig).forEach(page => {
				const devices = this.roomConfig[page].devices;
				devices.forEach(device => {
					this.$set(this.roomData[page].deviceOnline, device, false);
					this.$set(this.roomData[page].deviceData, device, {});
					this.$set(this.roomData[page].historyData, device, {});
				});
			});
			this.connectWebSocket();
			// 新增：页面加载时初始化时间并启动定时器
			this.updateCurrentTime();
			this.timeTimer = setInterval(() => this.updateCurrentTime(), 1000);
		},
		onReady() {
			this.initResizeObserver();
		},
		onUnload() {
			// 清理资源
			Object.values(this.charts).forEach(chart => chart.dispose());
			this.charts = {};

			if (this.socket) this.socket.close();
			this.socket = null;

			if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
			// 新增：页面卸载时清除时间定时器（避免内存泄漏）
			if (this.timeTimer) clearInterval(this.timeTimer);
		},
		methods: {
			// 左侧导航点击事件
			goToPage(page) {
				// 销毁当前页面图表
				Object.keys(this.charts).forEach(chartId => {
					if (chartId.startsWith(this.currentPage)) {
						this.charts[chartId].dispose();
						delete this.charts[chartId];
					}
				});

				this.currentPage = page;

				// 初始化新页面图表（仅选中机房时执行）
				if (this.currentPage !== 'home') {
					this.$nextTick(() => {
						setTimeout(() => this.initRoomCharts(this.currentPage), 500);
					});
				}
			},

			// 新增：更新当前时间（格式：2024-05-20 14:30:00）
			updateCurrentTime() {
				const date = new Date();
				// 补零：确保个位数时间显示为两位数（如 5 → 05）
				const year = date.getFullYear();
				const month = this.formatZero(date.getMonth() + 1); // 月份从 0 开始，需 +1
				const day = this.formatZero(date.getDate());
				const hour = this.formatZero(date.getHours());
				const minute = this.formatZero(date.getMinutes());
				const second = this.formatZero(date.getSeconds());
				// 拼接时间格式并赋值
				this.currentTime = `${year}-${month}-${day} ${hour}:${minute}:${second}`;
			},

			// 新增：补零工具函数（用于时间格式统一）
			formatZero(num) {
				return num < 10 ? `0${num}` : num;
			},

			// 安全获取数据（保持原逻辑）
			safeGetRoomDevices(page) {
				return this.roomConfig[page]?.devices || [];
			},
			safeGetRoomData(page) {
				return this.roomData[page] || {
					deviceOnline: {},
					dataHeaders: [],
					historyData: {},
					timeStamps: []
				};
			},
			safeGetChartHeaders(page) {
				const roomData = this.safeGetRoomData(page);
				const excludeHeaders = ['name', 'state', 'led state'];
				return (roomData.dataHeaders || []).filter(header => !excludeHeaders.includes(header));
			},

			// 图表初始化（保持原逻辑，注意参数为 page）
			initRoomCharts(page) {
				const chartHeaders = this.safeGetChartHeaders(page);
				if (chartHeaders.length === 0) return;

				chartHeaders.forEach(header => {
					const chartId = `${page}-${header}-chart`;
					let attempt = 0;
					const maxAttempts = 10;

					const checkDom = setInterval(() => {
						attempt++;
						const chartDom = document.getElementById(chartId);

						if (chartDom) {
							clearInterval(checkDom);
							chartDom.style.width = '100%';
							chartDom.style.height = '300px';

							if (this.charts[chartId]) this.charts[chartId].dispose();
							this.charts[chartId] = echarts.init(chartDom);
							this.setRoomChartOption(page, header);
						} else if (attempt >= maxAttempts) {
							clearInterval(checkDom);
							console.error(`图表容器${chartId}加载失败`);
						}
					}, 100);
				});
			},
			setRoomChartOption(page, header) {
				const chartId = `${page}-${header}-chart`;
				if (!this.charts[chartId]) return;

				const roomData = this.safeGetRoomData(page);
				const devices = this.safeGetRoomDevices(page);
				const series = devices.map(device => ({
					name: device,
					type: 'line',
					data: roomData.historyData[header]?.[device] || [],
					smooth: true,
					showSymbol: false
				}));

				this.charts[chartId].setOption({
					tooltip: {
						trigger: 'axis'
					},
					legend: {
						data: devices,
						top: 0
					},
					xAxis: {
						type: 'category',
						data: roomData.timeStamps || [],
						boundaryGap: false
					},
					yAxis: {
						type: 'value',
						name: this.thresholds[header]?.unit || ''
					},
					series: series
				});
			},

			updateRoomHistoryData(page, header, device, value) {
				const roomData = this.safeGetRoomData(page);
				const numericValue = parseFloat(value.toString().replace(/[^\d.]/g, ''));
				if (isNaN(numericValue)) return;

				// 初始化历史数据结构
				if (!roomData.historyData[header]) this.$set(roomData.historyData, header, {});
				if (!roomData.historyData[header][device]) this.$set(roomData.historyData[header], device, []);

				// 添加新数据
				roomData.historyData[header][device].push(numericValue);

				// 同步时间戳
				const timeStamp = new Date().toLocaleTimeString();
				if (roomData.timeStamps.length < roomData.historyData[header][device].length) {
					roomData.timeStamps.push(timeStamp);
				}

				// 限制数据数量
				if (roomData.timeStamps.length > this.maxDataPoints) {
					roomData.timeStamps.shift();
					const devices = this.safeGetRoomDevices(page);
					devices.forEach(dev => {
						if (roomData.historyData[header][dev]) roomData.historyData[header][dev].shift();
					});
				}

				// 更新图表
				this.updateRoomCharts(page, header);
			},

			updateRoomCharts(page, header) {
				const chartId = `${page}-${header}-chart`;
				if (!this.charts[chartId]) return;

				const roomData = this.safeGetRoomData(page);
				const devices = this.safeGetRoomDevices(page);
				this.charts[chartId].setOption({
					xAxis: {
						data: roomData.timeStamps
					},
					series: devices.map(device => ({
						name: device,
						data: roomData.historyData[header]?.[device] || []
					}))
				});
			},

			// 数据格式化
			formatHeaderName(header) {
				const headerMap = {
					'temperature': '温度',
					'humidity': '湿度',
					'lightIntensity': '光照强度',
					'voltage': '电压',
					'current': '电流'
				};
				return headerMap[header] || header;
			},

			formatValue(value, header) {
				if (value === '-' || value === undefined || value === null) return value;

				const thresholds = this.thresholds[header];
				if (!thresholds) return value;

				const numericValue = parseFloat(value);
				if (isNaN(numericValue)) return value;

				return `${numericValue.toFixed(header === 'lightIntensity'? 2 : 1)}${thresholds.unit}`;
			},

			// 报警相关方法
			getAlarmText(page, dataType) {
				const thresholds = this.thresholds[dataType];
				if (!thresholds) return '';

				const roomData = this.safeGetRoomData(page);
				const devices = this.safeGetRoomDevices(page);
				const abnormalDevices = [];

				devices.forEach(device => {
					const value = roomData.deviceData[device]?.[dataType];
					if (value === undefined || value === '-') return;

					const numericValue = parseFloat(value);
					if (numericValue < thresholds.min) {
						abnormalDevices.push(`${device}${thresholds.label}过低(${numericValue}${thresholds.unit})`);
					} else if (numericValue > thresholds.max) {
						abnormalDevices.push(`${device}${thresholds.label}过高(${numericValue}${thresholds.unit})`);
					}
				});

				return abnormalDevices.length > 0 ?
					`⚠️ ${thresholds.label}异常: ${abnormalDevices.join('，')}` :
					`${thresholds.label}正常`;
			},

			getAlarmClass(page, dataType) {
				const thresholds = this.thresholds[dataType];
				if (!thresholds) return '';

				const roomData = this.safeGetRoomData(page);
				const devices = this.safeGetRoomDevices(page);

				for (const device of devices) {
					const value = roomData.deviceData[device]?.[dataType];
					if (value === undefined || value === '-') continue;

					const numericValue = parseFloat(value);
					if (numericValue < thresholds.min || numericValue > thresholds.max) {
						return 'alarm-pulse alarm-shake';
					}
				}

				return 'normal';
			},

			// WebSocket相关方法
			connectWebSocket() {
				if (this.socket) this.socket.close();
				this.socket = uni.connectSocket({
					url: this.webSocketUrl,
					success: () => {
						console.log('WebSocket连接成功');
						if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
					},
					fail: (err) => {
						console.error('WebSocket连接失败:', err);
						this.handleDisconnect();
					}
				});

				this.socket.onMessage((res) => this.handleMessage(res));
				this.socket.onClose(() => this.handleDisconnect());
				this.socket.onError((err) => this.handleDisconnect());
			},

			handleDisconnect() {
				Object.keys(this.roomData).forEach(page => {
					const devices = this.safeGetRoomDevices(page);
					devices.forEach(device => {
						this.$set(this.safeGetRoomData(page).deviceOnline, device, false);
					});
				});
				this.startReconnect();
			},

			startReconnect() {
				if (!this.reconnectTimer) {
					this.reconnectTimer = setTimeout(() => {
						console.log('尝试重新连接WebSocket...');
						this.connectWebSocket();
					}, this.reconnectInterval);
				}
			},

			handleMessage(res) {
				try {
					const data = JSON.parse(res.data);
					const dataArray = Array.isArray(data) ? data : [data];
					dataArray.forEach(item => this.processDataItem(item));
				} catch (e) {
					console.error('解析数据失败:', e);
				}
			},

			findDeviceRoom(device) {
				for (const page in this.roomConfig) {
					if (this.roomConfig[page].devices.includes(device)) {
						return page;
					}
				}
				return null;
			},

			processDataItem(dataItem) {
				const realData = dataItem.data || dataItem;
				if (!realData || !realData.name) return;

				const device = realData.name;
				const page = this.findDeviceRoom(device);
				if (!page) {
					console.warn(`未知设备${device}，跳过处理`);
					return;
				}

				const roomData = this.safeGetRoomData(page);
				const currentDevices = this.safeGetRoomDevices(page);

				// 更新设备在线状态
				this.$set(roomData.deviceOnline, device, true);

				// 首次接收数据时初始化表头
				if (roomData.dataHeaders.length === 0) {
					const headers = Object.keys(realData).filter(key => key !== 'name');
					this.$set(roomData, 'dataHeaders', headers);

					// 初始化历史数据结构
					headers.forEach(header => {
						this.$set(roomData.historyData, header, {});
						currentDevices.forEach(dev => {
							this.$set(roomData.historyData[header], dev, []);
						});
					});

					// 延迟初始化图表
					this.$nextTick(() => {
						setTimeout(() => {
							this.initRoomCharts(page);
						}, 500);
					});
				}

				// 更新实时数据和历史数据
				currentDevices.forEach(dev => {
					if (dev === device) {
						(roomData.dataHeaders || []).forEach(header => {
							if (realData[header] !== undefined) {
								this.$set(roomData.deviceData[dev], header, realData[header]);
								this.updateRoomHistoryData(page, header, dev, realData[header]);
							}
						});
					}
				});
			},

			// 发送控制命令
			sendCommand(command, deviceName) {
				if (!this.socket) {
					console.error('WebSocket未连接，无法发送指令');
					return;
				}

				const message = JSON.stringify({
					name: deviceName,
					code: command
				});

				this.socket.send({
					data: message,
					success() {
						console.log(`已向${deviceName}发送${command}指令`);
					},
					fail(err) {
						console.error(`发送指令失败:`, err);
					}
				});
			},

			// 初始化窗口大小监听
			initResizeObserver() {
				if (typeof ResizeObserver === 'undefined') return;

				const resizeObserver = new ResizeObserver(entries => {
					Object.values(this.charts).forEach(chart => {
						chart.resize();
					});
				});

				// 监听所有图表容器
				['page1', 'page2', 'page3'].forEach(page => {
					this.safeGetChartHeaders(page).forEach(header => {
						const chartDom = document.getElementById(`${page}-${header}-chart`);
						if (chartDom) {
							resizeObserver.observe(chartDom);
						}
					});
				});
			},
		}
	}
</script>

<style scoped>
	/* 整体布局：左侧固定 + 右侧自适应 */
	.layout-container {
		display: flex;
		height: 100vh;
		background-color: #f5f5f5;
	}

	/* 左侧导航栏 */
	.sidebar {
		width: 220px;
		background-color: #2c3e50;
		color: white;
		display: flex;
		flex-direction: column;
		align-items: center;
		padding-top: 20px;
	}

	.sidebar-title {
		font-size: 20px;
		font-weight: bold;
		margin-bottom: 30px;
	}

	.sidebar-item {
		width: 100%;
		text-align: center;
		padding: 15px 0;
		cursor: pointer;
		transition: background-color 0.3s;
	}

	.sidebar-item.active {
		background-color: #34495e;
	}

	/* 右侧内容区 */
	.main-content {
		flex: 1;
		padding: 20px;
		box-sizing: border-box;
	}

	/* 未选中机房时的空白提示 */
	.blank-page {
		display: flex;
		justify-content: center;
		align-items: center;
		height: calc(100vh - 40px);
		/* 减去 padding */
		font-size: 18px;
		color: #999;
	}

	/* 选中机房时的内容区 */
	.room-content {
		background-color: white;
		border-radius: 8px;
		padding: 20px;
		height: calc(100vh - 40px);
		/* 减去 padding */
		overflow-y: auto;
	}

	/* 机房标题 + 返回按钮 + 当前时间（新增布局调整） */
	.room-header {
		display: flex;
		justify-content: space-between;
		align-items: flex-start;
		/* 顶部对齐，避免时间换行影响布局 */
		margin-bottom: 20px;
	}

	/* 新增：右侧按钮+时间的容器，垂直排列 */
	.header-right {
		display: flex;
		flex-direction: column;
		align-items: flex-end;
		/* 右对齐，与标题左右呼应 */
		gap: 8px;
		/* 按钮和时间的间距 */
	}

	.room-title {
		font-size: 20px;
		font-weight: bold;
	}

	.back-btn {
		padding: 8px 16px;
		background-color: #34495e;
		color: white;
		border-radius: 4px;
	}

	/* 新增：当前时间样式 */
	.current-time {
		font-size: 14px;
		color: #666;
		/* 灰色字体，避免与功能按钮冲突 */
		white-space: nowrap;
		/* 防止时间换行 */
	}

	.container {
		padding: 20px;
		font-family: Arial, sans-serif;
		box-sizing: border-box;
	}

	.title {
		font-size: 24px;
		font-weight: bold;
		margin: 20px 0;
		text-align: center;
	}

	.data-table {
		border: 1px solid #000;
		border-collapse: collapse;
		width: 100%;
		max-width: 1200px;
		margin: 0 auto;
		margin-bottom: 30px;
	}

	.table-row {
		display: flex;
		border-bottom: 1px solid #000;
	}

	.data-name {
		flex: 1;
		padding: 12px 8px;
		border-right: 1px solid #000;
		text-align: center;
		background-color: #f0f0f0;
		font-weight: bold;
	}

	.data-value {
		flex: 1;
		padding: 12px 8px;
		text-align: center;
		border-right: 1px solid #000;
	}

	.data-value:last-child {
		border-right: none;
	}

	.chart-container {
		width: 100% !important;
		height: 300px !Important;
		max-width: 1200px;
		margin: 0 auto;
		border: 1px solid #eee;
		margin-bottom: 30px;
		position: relative;
	}

	.chart-title {
		font-size: 18px;
		font-weight: bold;
		margin: 20px 0 10px;
		text-align: center;
	}

	.button-container {
		margin: 20px 0;
		text-align: center;
		display: flex;
		flex-wrap: wrap;
		justify-content: center;
		gap: 10px;
	}

	.button-container button {
		padding: 10px 15px;
		font-size: 16px;
		background-color: #4285f4;
		color: white;
		border: none;
		border-radius: 4px;
		cursor: pointer;
	}

	.button-container button:active {
		background-color: #3367d6;
	}

	.status-container {
		display: flex;
		align-items: center;
		justify-content: center;
		flex-wrap: wrap;
		gap: 20px;
		margin: 20px 0;
	}

	.status-icon {
		width: 20px;
		height: 20px;
		border-radius: 50%;
		display: inline-block;
		margin-right: 8px;
	}

	.status-text {
		font-size: 18px;
		font-weight: bold;
		display: flex;
		align-items: center;
	}

	/* 报警样式 */
	.alarms-container {
		display: flex;
		flex-direction: column;
		gap: 10px;
		margin: 15px 0;
	}

	.data-alarm {
		padding: 10px;
		text-align: center;
		border-radius: 8px;
		font-weight: bold;
		font-size: 18px;
	}

	.data-alarm.normal {
		background-color: #e8f5e9;
		color: #2e7d32;
	}

	.data-alarm.alarm-pulse {
		animation: pulse 1.5s ease-in-out infinite;
	}

	.data-alarm.alarm-shake {
		animation: shake 1.5s cubic-bezier(.36, .07, .19, .97) 2s infinite;
	}

	@keyframes pulse {

		0%,
		100% {
			opacity: 1;
		}

		50% {
			opacity: 0.7;
		}
	}

	@keyframes shake {

		10%,
		90% {
			transform: translateX(-1px);
		}

		20%,
		80% {
			transform: translateX(2px);
		}

		30%,
		50%,
		70% {
			transform: translateX(-4px);
		}

		40%,
		60% {
			transform: translateX(4px);
		}
	}

	/* 温度报警特殊样式 */
	.temperature-alarm.normal {
		background-color: #e8f5e9;
		color: #2e7d32;
	}

	.temperature-alarm.alarm-pulse {
		background-color: #ffebee;
		color: #b71c1c;
	}

	/* 湿度报警特殊样式 */
	.humidity-alarm.normal {
		background-color: #e3f2fd;
		color: #0d47a1;
	}

	.humidity-alarm.alarm-pulse {
		background-color: #fff3e0;
		color: #e65100;
	}

	/* 电压报警特殊样式 */
	.lightIntensity-alarm.normal {
		background-color: #e8f5e9;
		color: #2e7d32;
	}

	.lightIntensity-alarm.alarm-pulse {
		background-color: #ffebee;
		color: #b71c1c;
	}

	/* 响应式调整（新增时间适配） */
	@media (max-width: 768px) {
		.table-row {
			flex-wrap: wrap;
		}

		.data-name {
			flex-basis: 100%;
			border-right: none;
			border-bottom: 1px solid #000;
		}

		.data-value {
			flex-basis: 49%;
		}

		.button-container button {
			font-size: 14px;
			padding: 8px 12px;
		}

		.chart-title {
			font-size: 16px;
		}
	}
</style>