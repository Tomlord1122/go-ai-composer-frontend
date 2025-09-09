<script lang="ts">
	import EnhancedFileUploader from './lib/components/EnhancedFileUploader.svelte';
	import ToastContainer from './lib/components/ToastContainer.svelte';
</script>

<EnhancedFileUploader />
<ToastContainer />

<!-- Test File Download Section -->
<div class="fixed bottom-4 left-4 z-40">
	<div class="max-w-xs rounded-lg border border-gray-300 bg-white p-4 shadow-lg">
		<h3 class="mb-2 font-serif text-sm font-semibold text-gray-800">測試檔案</h3>
		<p class="mb-3 text-xs text-gray-600">下載範例PDF檔案進行測試</p>
		<a
			href="/717這件事21_20_28-1-8.pdf"
			download="717這件事21_20_28-1-8.pdf"
			class="inline-block rounded-md bg-gray-600 px-3 py-1.5 text-xs font-medium text-white transition-colors duration-200 hover:bg-gray-700"
		>
			下載測試PDF
		</a>
	</div>
</div>
