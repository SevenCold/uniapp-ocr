<template>
	<view class="page">
		<u-card title="添加图片" title-size="40" border-radius="0">
			<view class="" slot="body" :prop="need_init" :change:prop="ocr.initWorker">
				<view class="u-body-item u-flex u-col-between u-p-t-0" :prop="base64" :change:prop="ocr.receiveImg">
					<easy-upload
					:dataList="imageList" @successDelImage="successDelImage" @successImageChoose="successImageChoose"></easy-upload>
				</view>
			</view>
			<view class="" slot="foot" v-if="rec_status !== 'no'">
				<u-line-progress v-if="rec_status === 'doing'" :percent="process" type="success" :height="100" :show-percent="true">
					<view>识别中: {{process + '%'}}</view>
				</u-line-progress>
				<view class="content" v-if="rec_status === 'finished'">
					<view v-for="(content in ocr_contents">{{content}}</view>
				</view>
			</view>
		</u-card>
	</view>
</template>
<script>
	import easyUpload from '@/components/easy-upload/easy-upload.vue'
	export default {
		data() {
			return {
				imageList: [],
				base64: '',
				process: 0,
				rec_status: 'no',    // 识别图像状态： no、doing、finished
				ocr_contents: '',
				need_init: false    // 加载worker
			}
		},
		mounted() {
			// 创建时，改变该变量，触发加载worker事件
			this.need_init = !this.need_init
		},
		components:{
			easyUpload
		},
		methods: {
			getProcess(worker) {
				if (worker.status === 'recognizing text') {
					// 加强显示效果，因为识别总是从0.6 -> 1
					if (worker.progress < 0.7) {
						this.process = parseInt(worker.progress * 100) + 25
					} else {
						this.process = parseInt(worker.progress * 100)
					}
				}
			},
			// 接收识别结果，并赋值
			setText (text) {
				let lines = text.split('\n')
				this.ocr_contents = lines
				// 清空base64的值，避免识别同一图片，触发不了更新
				this.base64 = ''
				// 识别结束
				this.rec_status = 'finished'
				this.process = 0
			},
			successDelImage() {
				// 删除图片时，状态为no，都不显示
				this.rec_status = 'no'
			},
			successImageChoose() {
				let localPath = this.imageList[0] 
				let _this = this
				// 状态改为doing
				this.rec_status = 'doing'
				uni.compressImage({
					src: localPath,
					quality: 80,
					success: ys => {
						plus.io.resolveLocalFileSystemURL(ys.tempFilePath, function(entry) {
							entry.file(function(file) {
								var fileReader = new plus.io.FileReader();
								fileReader.readAsDataURL(file); 
								fileReader.onloadend = function(evt) {
									_this.base64 = evt.target.result
								}
							});
						});
					}
				})
			},
		}
	}
</script>

<script module="ocr" lang="renderjs">
	import Tesseract from 'static/ocr/tesseract.min.js'
	let worker
	export default {
		mounted: function() {
		},
		methods: {
			initWorker: async function(newValue, oldValue, ownerInstance, instance) {
				if (worker == undefined) {
					const { createWorker } = Tesseract;
					const localPath = plus.io.convertLocalFileSystemURL('_downloads/')
					try{
						worker = createWorker({
							workerPath: localPath + '/worker.min.js',
							langPath: localPath,
							corePath: localPath +  'tesseract-core.wasm.js',
							logger: function(m) {
								ownerInstance.callMethod('getProcess', m)
							},
							//  打包apk时.gz打不进去，只能是解压后的数据
							gzip: false
						});
						await worker.load();
						await worker.loadLanguage('eng');
						await worker.initialize('eng', 1);
						await worker.setParameters({
							tessedit_pageseg_mode: 6,
						});
					}catch(e){
						ownerInstance.callMethod('getProcess', e)
					}
				}
			},
			receiveImg:async function(newValue, oldValue, ownerInstance, instance) {
				if (newValue === '' || worker == undefined) {
					return
				}
				try{
					const { data: { text } } = await worker.recognize(newValue);
					ownerInstance.callMethod('setText', text)
				}catch(e){
					ownerInstance.callMethod('getProcess', e)
				}
				
			}
		}
	}
</script>
<style>
	.page {
		padding: 40rpx;
	}
	.tip-content {
		text-align: center;
		margin-top: 10rpx;
		font-size: 40rpx;
		font-weight: 700;
	}
</style>
