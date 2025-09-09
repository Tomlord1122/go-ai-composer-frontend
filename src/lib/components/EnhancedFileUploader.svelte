<script lang="ts">
	import PDFPreview from './PDFPreview.svelte';
	import TextExtractor from './TextExtractor.svelte';
	import ProgressSteps from './ProgressSteps.svelte';
	import JSZip from 'jszip';
	import { textExtractorPrompt } from '../utils/prompts.js';
	import { toastStore } from '../stores/toastStore.js';
	import { fade, fly } from 'svelte/transition';

	// Workflow steps definition
	const workflowSteps = [
		{ id: 'upload', title: '上傳檔案', description: '選擇PDF檔案或圖片' },
		{ id: 'setup', title: '設定參數', description: '配置行列數參數' },
		{ id: 'convert', title: '轉換處理', description: '處理圖片轉向' },
		{ id: 'analyze', title: 'AI 批改', description: '分析並評分' }
	];

	// State management using Svelte 5 runes
	let currentStep = $state(0);
	let files = $state<File[]>([]);
	let error = $state('');
	let processedImages = $state<File[]>([]);

	// Processing inputs
	let rows = $state(0);
	let pagesPerArticle = $state(2);
	let useUniformColumns = $state(false);
	let processTrigger = $state(0);
	let pageColumns = $state<number[]>([]);

	// Model selection
	const modelOptions = [
		{ value: 'gemini-1.5-flash', label: 'Gemini 1.5 Flash' },
		{ value: 'gemini-1.5-flash-8b', label: 'Gemini 1.5 Flash 8B' },
		{ value: 'gemini-1.5-pro', label: 'Gemini 1.5 Pro' },
		{ value: 'gemini-2.0-flash-exp', label: 'Gemini 2.0 Flash Exp' },
		{ value: 'gemini-2.0-pro-exp-02-05', label: 'Gemini 2.0 Pro Exp' }
	];
	let selectedModel = $state('gemini-2.0-flash-exp');

	// Page selection state
	let selectedPageIndex = $state<number>(0);
	let isProcessing = $state(false);

	let customPrompt = $state(textExtractorPrompt);
	let cols = $state(0);
	let extractedText = $state('');

	// Use derived state for pageColumns to avoid infinite loops
	$effect(() => {
		if (useUniformColumns) {
			const newColumns = Array(pagesPerArticle).fill(cols);
			// Only update if actually different
			if (JSON.stringify(newColumns) !== JSON.stringify(pageColumns)) {
				pageColumns = newColumns;
			}
		} else {
			// Only update array length if needed
			if (pageColumns.length !== pagesPerArticle) {
				const currentCols = [...pageColumns];
				pageColumns = Array(pagesPerArticle)
					.fill(0)
					.map((_, i) => currentCols[i] || 0);
			}
		}
	});

	function handleFileChange(event: Event) {
		const input = event.target as HTMLInputElement;
		console.log('File change event triggered:', input.files);
		
		if (input.files && input.files.length > 0) {
			const file = input.files[0];
			console.log('Selected file:', file.name, file.type, file.size);
			
			const validTypes = ['image/jpeg', 'image/png', 'application/pdf'];

			if (!validTypes.includes(file.type)) {
				error = '請上傳 JPG、PNG 或 PDF 檔案';
				toastStore.add('檔案格式不正確，請上傳 JPG、PNG 或 PDF 檔案', 'error');
				return;
			}

			files = Array.from(input.files);
			error = '';
			processedImages = [];
			console.log('Files updated:', files.length, 'Current step:', currentStep);
			
			toastStore.add('檔案上傳成功！請設定參數後開始轉換', 'success');

			// Auto advance to setup step
			if (currentStep === 0) {
				console.log('Advancing to step 1');
				currentStep = 1;
			}
		} else {
			console.log('No files selected or input.files is null');
		}
	}

	function handleProcessedImages(images: File[]) {
		console.log('handleProcessedImages called with:', images.length, 'images');
		console.log('Current isProcessing state:', isProcessing);
		console.log('Current step:', currentStep);
		
		processedImages = images;
		isProcessing = false;
		toastStore.add('圖片轉換完成！請查看結果並選擇下一步操作', 'success');

		console.log('Updated isProcessing to false, processedImages length:', processedImages.length);

		// Don't auto advance - let user manually choose next step
	}

	function handleSelectedPages(pageIndex: number) {
		console.log('Selected page:', pageIndex);
		selectedPageIndex = Math.floor(pageIndex / pagesPerArticle);
	}

	async function handleDownloadAll() {
		try {
			const zip = new JSZip();

			processedImages.forEach((file, index) => {
				zip.file(`page-${index + 1}.png`, file);
			});

			const content = await zip.generateAsync({
				type: 'blob',
				compression: 'DEFLATE',
				compressionOptions: {
					level: 6
				}
			});

			const link = document.createElement('a');
			link.href = URL.createObjectURL(content);
			link.download = files[0].name.replace(/\.[^.]+$/, '') + '.zip';
			document.body.appendChild(link);
			link.click();
			document.body.removeChild(link);
			URL.revokeObjectURL(link.href);

			toastStore.add('檔案下載完成！', 'success');
		} catch (err) {
			console.error('Error creating zip:', err);
			error = '建立壓縮檔時發生錯誤';
			toastStore.add('下載失敗，請重試', 'error');
		}
	}

	function handleExtractionComplete(text: string) {
		extractedText = text;
		toastStore.add('AI 批改完成！', 'success');
	}

	function handleSetupSubmit() {
		if (rows <= 0) {
			error = '請輸入有效的列數（大於 0）';
			toastStore.add('請輸入有效的列數', 'error');
			return;
		}

		if (!pageColumns.every((col) => col > 0)) {
			error = '請確保每頁的行數都大於 0';
			toastStore.add('請確保每頁的行數都大於 0', 'error');
			return;
		}

		try {
			isProcessing = true;
			processTrigger++;
			currentStep = 2;
			error = '';
			toastStore.add('開始轉換處理...', 'info');
		} catch (err) {
			console.error('Error processing images:', err);
			error = '處理圖片時發生錯誤，請重試';
			toastStore.add('處理失敗，請重試', 'error');
			isProcessing = false;
		}
	}

	function goToStep(step: number) {
		console.log('goToStep called with:', step, 'current step:', currentStep);
		console.log('Conditions check:', {
			stepLessOrEqual: step <= currentStep,
			step1Available: step === 1 && files.length > 0,
			step2Available: step === 2 && processedImages.length > 0,
			step3Available: step === 3 && processedImages.length > 0
		});
		
		if (
			step <= currentStep ||
			(step === 1 && files.length > 0) ||
			(step === 2 && processedImages.length > 0) ||
			(step === 3 && processedImages.length > 0)
		) {
			console.log('Advancing to step:', step);
			currentStep = step;
		} else {
			console.log('Cannot advance to step', step, 'conditions not met');
		}
	}

	function selectEssay(index: number) {
		console.log('Selected essay index:', index);
		selectedPageIndex = index;
		toastStore.add(`已選擇作文 ${index + 1}`, 'info');
	}

	function resetWorkflow() {
		currentStep = 0;
		files = [];
		processedImages = [];
		extractedText = '';
		error = '';
		isProcessing = false;
		processTrigger = 0;
		selectedPageIndex = 0;
	}
</script>

<div class="min-h-screen bg-[#EDEDED] py-8">
	<div class="container mx-auto px-4 lg:max-w-screen-md">
		<!-- Header -->
		<div >
			<h1 class="mb-2 font-serif text-3xl font-bold text-gray-800">改作文gem</h1>
			<p class="prose prose-sm text-gray-600">
				AI 驅動的中文作文批改系統，將直書作文轉換為橫書並提供專業評分
			</p>
		</div>

		<!-- Progress Steps -->
		<div class="mb-8">
			<ProgressSteps steps={workflowSteps} {currentStep} />
		</div>

		<!-- Main Content Card -->
		<div class="mb-6 rounded-lg border border-gray-300 bg-white p-6 shadow-md">
			<!-- Error Display -->
			{#if error}
				<div
					class="mb-6 rounded-md border border-red-200 bg-red-50 p-3 text-sm text-red-600"
					transition:fade
				>
					{error}
				</div>
			{/if}

			<!-- Step Content -->
			{#if currentStep === 0}
				<!-- Step 1: File Upload -->
				<div 
					in:fly={{ y: 30, duration: 400, delay: 300 }}
					out:fly={{ y: 30, duration: 300}}
				>
					<h2 class="mb-4 font-serif text-xl font-semibold text-gray-800">步驟 1: 上傳檔案</h2>
					<div class="prose prose-sm mb-6 text-gray-700">
						<p>請上傳一個 PDF 檔案或圖片（JPG、PNG 格式）。系統將自動處理並轉換為可分析的格式。</p>
					</div>

					<div class="mb-6">
						<label for="file-upload" class="mb-3 block font-medium text-gray-700">選擇檔案</label>
						<input
							id="file-upload"
							type="file"
							accept=".jpg,.jpeg,.png,.pdf"
							onchange={handleFileChange}
							class="w-full rounded-md border border-gray-300 bg-gray-50 p-3 transition-colors hover:bg-white focus:border-blue-500 focus:ring-2 focus:ring-blue-500"
							multiple
						/>
					</div>
				</div>
			{:else if currentStep === 1}
				<!-- Step 2: Parameter Setup -->
				<div 
					in:fly={{ y: 30, duration: 400, delay: 300 }}
					out:fly={{ y: 30, duration: 300}}
				>
					<h2 class="mb-4 font-serif text-xl font-semibold text-gray-800">步驟 2: 設定參數</h2>
					<div class="prose prose-sm mb-6 text-gray-700">
						<p>請設定作文的網格參數，這將幫助系統正確識別和轉換文字方向。</p>
					</div>

					<!-- File Status -->
					{#if files.length > 0}
						<div class="mb-6 rounded-md border border-blue-200 bg-blue-50 p-4">
							<div class="flex items-center">
								<svg class="h-5 w-5 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
								</svg>
								<div class="ml-3">
									<p class="text-sm font-medium text-blue-800">已上傳檔案: {files[0].name}</p>
									<p class="text-xs text-blue-600">設定完參數後點擊「開始轉換」進行處理</p>
								</div>
							</div>
						</div>
					{/if}

					<div class="space-y-6">
						<!-- Basic Parameters -->
						<div class="grid grid-cols-1 gap-4 md:grid-cols-2">
							<div>
								<label for="pages-per-article" class="mb-2 block font-medium text-gray-700"
									>每篇文章頁數</label
								>
								<input
									id="pages-per-article"
									type="number"
									min="1"
									bind:value={pagesPerArticle}
									class="w-full rounded-md border border-gray-300 p-3 transition-colors focus:border-blue-500 focus:ring-2 focus:ring-blue-500"
								/>
							</div>
							<div>
								<label for="rows-input" class="mb-2 block font-medium text-gray-700">列數</label>
								<input
									id="rows-input"
									type="number"
									min="0"
									bind:value={rows}
									class="w-full rounded-md border border-gray-300 p-3 transition-colors focus:border-blue-500 focus:ring-2 focus:ring-blue-500"
								/>
							</div>
						</div>

						<!-- Uniform Columns Toggle -->
						<div class="flex items-center space-x-3 rounded-md bg-gray-50 p-4">
							<input
								type="checkbox"
								bind:checked={useUniformColumns}
								id="uniformColumns"
								class="h-4 w-4 rounded border-gray-300 text-blue-600 focus:ring-blue-500"
							/>
							<label for="uniformColumns" class="font-medium text-gray-700">
								所有頁面使用相同行數
							</label>
						</div>

						<!-- Column Configuration -->
						{#if useUniformColumns}
							<div>
								<label for="uniform-cols" class="mb-2 block font-medium text-gray-700"
									>每頁行數</label
								>
								<input
									id="uniform-cols"
									type="number"
									min="0"
									bind:value={cols}
									class="w-full rounded-md border border-gray-300 p-3 transition-colors focus:border-blue-500 focus:ring-2 focus:ring-blue-500"
								/>
							</div>
						{:else}
							<div class="grid grid-cols-2 gap-4 sm:grid-cols-3 md:grid-cols-4">
								{#each Array(pagesPerArticle) as _, index}
									<div>
										<label
											for="page-cols-{index}"
											class="mb-2 block text-sm font-medium text-gray-700"
										>
											第 {index + 1} 頁行數
										</label>
										<input
											id="page-cols-{index}"
											type="number"
											min="0"
											bind:value={pageColumns[index]}
											class="w-full rounded-md border border-gray-300 p-2 transition-colors focus:border-blue-500 focus:ring-2 focus:ring-blue-500"
										/>
									</div>
								{/each}
							</div>
						{/if}

						<!-- Action Buttons -->
						<div class="flex gap-3 pt-4">
							<button
								onclick={() => goToStep(0)}
								class="rounded-md bg-gray-200 px-4 py-2 text-gray-600 transition-colors duration-200 hover:bg-gray-300"
							>
								上一步
							</button>
							<button
								onclick={handleSetupSubmit}
								disabled={isProcessing}
								class="flex-1 rounded-md bg-blue-600 px-6 py-3 font-medium text-white transition-colors duration-200 hover:bg-blue-700 disabled:cursor-not-allowed disabled:bg-gray-400"
							>
								{isProcessing ? '處理中...' : '開始轉換'}
							</button>
						</div>
					</div>
				</div>
			{:else if currentStep === 2}
				<!-- Step 3: Processing -->
				<div 
					in:fly={{ y: 30, duration: 400, delay: 300 }}
					out:fly={{ y: 30, duration: 300}}
				>
					<h2 class="mb-4 font-serif text-xl font-semibold text-gray-800">步驟 3: 圖片轉換處理</h2>
					<div class="prose prose-sm mb-6 text-gray-700">
						<p>系統正在處理您的檔案，將直書文字轉換為橫書格式。請耐心等待...</p>
					</div>

					<!-- Processing Status -->
					{#if isProcessing}
						<div class="mb-6 rounded-md border border-yellow-200 bg-yellow-50 p-4">
							<div class="flex items-center">
								<svg class="h-5 w-5 animate-spin text-yellow-600" fill="none" viewBox="0 0 24 24">
									<circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
									<path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
								</svg>
								<div class="ml-3">
									<p class="text-sm font-medium text-yellow-800">正在處理中...</p>
									<p class="text-xs text-yellow-600">PDF轉換和圖片旋轉處理需要一些時間，請稍候</p>
								</div>
							</div>
						</div>
					{:else if processedImages.length > 0}
						<div class="mb-6 rounded-md border border-green-200 bg-green-50 p-4">
							<div class="flex items-center">
								<svg class="h-5 w-5 text-green-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z" />
								</svg>
								<div class="ml-3">
									<p class="text-sm font-medium text-green-800">處理完成！</p>
									<p class="text-xs text-green-600">已成功轉換 {processedImages.length} 個檔案，可以進行AI分析</p>
								</div>
							</div>
						</div>
					{/if}

					{#if files && files[0]?.type.startsWith('application/') && files?.length == 1}
						<PDFPreview
							file={files[0]}
							{rows}
							{pagesPerArticle}
							{pageColumns}
							{processTrigger}
							selectedIndex={selectedPageIndex}
							disabled={isProcessing}
							onprocessedimages={handleProcessedImages}
							onselectedpages={handleSelectedPages}
						/>
					{/if}

					<!-- Processing Results Preview -->
					{#if !isProcessing && processedImages.length > 0}
						<div class="mt-6 rounded-md border border-green-200 bg-green-50 p-6">
							<div class="text-center">
								<svg class="mx-auto h-12 w-12 text-green-600 mb-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z" />
								</svg>
								<h3 class="text-lg font-serif font-medium text-green-800 mb-2">轉換完成！</h3>
								<p class="text-sm text-green-600 mb-6">
									已成功轉換 {processedImages.length} 個檔案，直書作文已轉為橫書格式
								</p>
								
								<div class="flex gap-3 justify-center">
									<button
										onclick={handleDownloadAll}
										class="rounded-md bg-gray-600 px-4 py-2 text-white transition-colors duration-200 hover:bg-gray-700"
									>
										下載圖片
									</button>
									<button
										onclick={() => goToStep(3)}
										class="rounded-md bg-blue-600 px-6 py-2 text-white transition-colors duration-200 hover:bg-blue-700 font-medium"
									>
										繼續進行 AI 批改
									</button>
								</div>
							</div>
						</div>
					{/if}

					<div class="mt-6 flex gap-3">
						<button
							onclick={() => goToStep(1)}
							class="rounded-md bg-gray-200 px-4 py-2 text-gray-600 transition-colors duration-200 hover:bg-gray-300"
						>
							返回設定
						</button>
					</div>
				</div>
			{:else if currentStep === 3}
				<!-- Step 4: AI Analysis -->
				<div 
					in:fly={{ y: 30, duration: 400, delay: 300 }}
					out:fly={{ y: 30, duration: 300}}
				>
					<h2 class="mb-4 font-serif text-xl font-semibold text-gray-800">步驟 4: AI 智能批改</h2>
					<div class="prose prose-sm mb-6 text-gray-700">
						<p>選擇要分析的頁面，系統將使用 AI 進行文字識別和專業批改。</p>
					</div>

					<!-- Essay Selection -->
					{#if processedImages.length > 0}
						<div class="mb-6">
							<h4 class="mb-4 font-medium text-gray-700">選擇要分析的作文</h4>
							<div class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-4">
								{#each processedImages as image, index}
									<button 
										class="relative rounded-md border-2 transition-all duration-200 hover:shadow-md text-left
											{selectedPageIndex === index 
												? 'border-blue-500 bg-blue-50' 
												: 'border-gray-300 bg-white hover:border-gray-400'}"
										onclick={() => selectEssay(index)}
										aria-label="選擇作文 {index + 1}"
									>
										<!-- Image Preview -->
										<div class="aspect-[3/4] overflow-hidden rounded-t-md bg-gray-100">
											<img 
												src={URL.createObjectURL(image)}
												alt="作文預覽 {index + 1}"
												class="h-full w-full object-cover"
											/>
										</div>
										
										<!-- Image Info -->
										<div class="p-3">
											<p class="text-xs font-medium text-gray-700">
												作文 {index + 1}
											</p>
											<p class="text-xs text-gray-500">
												{Math.round(image.size / 1024)} KB
											</p>
										</div>

										<!-- Selection Indicator -->
										{#if selectedPageIndex === index}
											<div class="absolute top-2 right-2 rounded-full bg-blue-600 p-1">
												<svg class="h-3 w-3 text-white" fill="currentColor" viewBox="0 0 20 20">
													<path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd" />
												</svg>
											</div>
										{/if}
									</button>
								{/each}
							</div>
						</div>
					{/if}

					<!-- Model Selection -->
					<div class="mb-6">
						<label for="model-select" class="mb-2 block font-medium text-gray-700"
							>AI 模型選擇</label
						>
						<select
							id="model-select"
							bind:value={selectedModel}
							class="w-full rounded-md border border-gray-300 p-3 transition-colors focus:border-blue-500 focus:ring-2 focus:ring-blue-500"
						>
							{#each modelOptions as option}
								<option value={option.value}>
									{option.label}
								</option>
							{/each}
						</select>
					</div>

					<!-- Prompt Configuration -->
					<div class="mb-6">
						<label for="custom-prompt" class="mb-2 block font-medium text-gray-700"
							>批改提示詞</label
						>
						<textarea
							id="custom-prompt"
							bind:value={customPrompt}
							rows="4"
							class="resize-vertical w-full rounded-md border border-gray-300 p-3 transition-colors focus:border-blue-500 focus:ring-2 focus:ring-blue-500"
							placeholder="自定義 AI 批改的指導原則..."
						></textarea>
					</div>

					<!-- AI Processing Component -->
					{#if processedImages.length}
						<TextExtractor
							images={processedImages}
							{selectedPageIndex}
							{customPrompt}
							modelConfig={{ model: selectedModel }}
							onextractioncomplete={handleExtractionComplete}
						/>
					{/if}

					<div class="mt-6 flex gap-3">
						<button
							onclick={() => goToStep(2)}
							class="rounded-md bg-gray-200 px-4 py-2 text-gray-600 transition-colors duration-200 hover:bg-gray-300"
						>
							返回轉換
						</button>
						<button
							onclick={resetWorkflow}
							class="rounded-md bg-gray-200 px-4 py-2 text-gray-600 transition-colors duration-200 hover:bg-gray-300"
						>
							重新開始
						</button>
					</div>
				</div>
			{/if}
		</div>

		<!-- Quick Actions -->
		{#if currentStep > 0}
			<div class="rounded-lg border border-gray-300 bg-white p-4 shadow-sm" transition:fade>
				<h3 class="mb-3 text-sm font-medium text-gray-700">快速操作</h3>
				<div class="flex flex-wrap gap-2">
					{#each workflowSteps as step, index}
						<button
							onclick={() => goToStep(index)}
							disabled={index > currentStep &&
								!(index === 1 && files.length > 0) &&
								!(index === 2 && processedImages.length > 0)}
							class="rounded-full border px-3 py-1 text-xs transition-colors duration-200
								{index === currentStep
								? 'border-blue-300 bg-blue-100 text-blue-700'
								: index < currentStep
									? 'border-green-300 bg-green-100 text-green-700 hover:bg-green-200'
									: 'cursor-not-allowed border-gray-300 bg-gray-100 text-gray-500'}"
						>
							{step.title}
						</button>
					{/each}
				</div>
			</div>
		{/if}
	</div>
</div>
