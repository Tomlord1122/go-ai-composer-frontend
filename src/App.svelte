<script lang="ts">
	import FileUploader from './lib/components/FileUploader.svelte';
</script>

<div class="min-h-screen bg-[#EDEDED] py-12">
	<div class="container mx-auto lg:max-w-screen-md px-4">
		<FileUploader />
		
		<div class="mt-8 p-6 bg-white shadow-md rounded-lg max-w-md mx-auto text-center border border-gray-300">
			<h2 class="text-xl font-serif font-semibold mb-4 text-gray-800">下載檔案</h2>
			<p class="text-gray-600 mb-4">點擊下方按鈕下載PDF檔案：</p>
			<a 
				href="/717這件事21_20_28-1-8.pdf" 
				download="717這件事21_20_28-1-8.pdf"
				class="bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 transition-colors duration-200 inline-block"
			>
				下載測試用PDF檔案
			</a>
		</div>
	</div>
</div>
