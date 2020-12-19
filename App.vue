<script>
	export default {
		onLaunch: function() {
			// 将私有目录中的识别文件 移动到公共目录
			// 因为tesseract.js 运行时需要加载外部js文件，访问不了私有目录中文件
			this.resolveOcrFile('worker.min.js')
			this.resolveOcrFile('tesseract-core.wasm.js')
			this.resolveOcrFile('eng.traineddata')
		},
		onShow: function() {
		},
		onHide: function() {
		},
		methods: {
			resolveOcrFile(fileName) {
				plus.io.resolveLocalFileSystemURL( '_downloads/' + fileName, function() {
					// 公共目录已有文件，跳过
				}, function () {
					// 没有文件， 从私有目录移动过来
					plus.io.resolveLocalFileSystemURL( '_www/static/ocr/' + fileName, function( entry ) {
						entry.copyTo( {"fullPath": plus.io.convertLocalFileSystemURL('_downloads/')}, fileName);
					});
				});
			}
		}
	}
</script>

<style lang="scss">
@import "uview-ui/index.scss";
</style>
