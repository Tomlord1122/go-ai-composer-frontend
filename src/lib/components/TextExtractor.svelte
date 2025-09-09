<script lang="ts">
	import { extractTextFromImage } from '../services/geminiService.js';

	interface Props {
		images: File[];
		modelConfig: {
			model: string;
		};
		customPrompt?: string;
		evaluatorPrompt?: string;
		selectedPageIndex?: number;
		onextractioncomplete: (text: string) => void;
	}

	let {
		images,
		modelConfig,
		customPrompt,
		evaluatorPrompt,
		selectedPageIndex = 0,
		onextractioncomplete
	}: Props = $props();

	let extractedText = $state('');
	let isProcessing = $state(false);
	let error = $state('');

	async function handleExtraction() {
		if (!images?.length || selectedPageIndex === undefined) {
			error = '請選擇圖片';
			return;
		}

		isProcessing = true;
		error = '';
		
		try {
			console.log('Selected image:', {
				index: selectedPageIndex,
				totalImages: images.length,
				selectedImage: images[selectedPageIndex]
			});

			const selectedImage = images[selectedPageIndex];
			if (!selectedImage) {
				throw new Error('No image found at selected index');
			}

			const result = await extractTextFromImage(selectedImage, modelConfig, customPrompt);
			
			if (result.error) throw new Error(result.error);
			extractedText = result.text;
			onextractioncomplete(result.text);
		} catch (err: any) {
			error = `文字擷取失敗：${err?.message || '未知錯誤'}`;
		} finally {
			isProcessing = false;
		}
	}
</script>

<div class="mb-6">
	<div class="flex justify-between items-center mb-4">
		<h2 class="text-xl font-serif font-semibold text-gray-800 m-0">批改結果</h2>
		<button 
			onclick={handleExtraction}
			disabled={isProcessing || !images?.length}
			class="px-4 py-2 text-white border-none rounded-md cursor-pointer text-sm transition-colors duration-200
				{isProcessing || !images?.length 
					? 'bg-gray-400 cursor-not-allowed' 
					: 'bg-blue-600 hover:bg-blue-700'}"
		>
			{isProcessing ? '正在批改中...' : '開始批改'}
		</button>
	</div>

	{#if error}
		<div class="text-red-600 mb-4 text-sm bg-red-50 border border-red-200 rounded-md p-3">
			{error}
		</div>
	{/if}

	<textarea
		bind:value={extractedText}
		rows="15"
		placeholder="批改結果將顯示在此處..."
		class="w-full p-3 border border-gray-300 rounded-md font-sans text-sm leading-relaxed resize-vertical box-border 
			focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-colors"
		style="font-family: 'Microsoft JhengHei', sans-serif;"
	></textarea>
</div>
