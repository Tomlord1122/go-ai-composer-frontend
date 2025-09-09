<script lang="ts">
	import { fly } from 'svelte/transition';

	interface Props {
		message: string;
		type: 'success' | 'error' | 'info';
		visible: boolean;
		onclose?: () => void;
	}

	let { message, type, visible = false, onclose }: Props = $props();

	// Auto-dismiss after 3 seconds for success/info, 5 seconds for error
	let timeoutId: ReturnType<typeof setTimeout>;

	$effect(() => {
		if (visible) {
			const duration = type === 'error' ? 5000 : 3000;
			timeoutId = setTimeout(() => {
				onclose?.();
			}, duration);
		}

		return () => {
			if (timeoutId) {
				clearTimeout(timeoutId);
			}
		};
	});

	function handleClose() {
		if (timeoutId) {
			clearTimeout(timeoutId);
		}
		onclose?.();
	}

	// Get styling based on type
	const getTypeStyles = (type: string) => {
		switch (type) {
			case 'success':
				return 'bg-green-50 border-green-200 text-green-800';
			case 'error':
				return 'bg-red-50 border-red-200 text-red-800';
			case 'info':
			default:
				return 'bg-blue-50 border-blue-200 text-blue-800';
		}
	};

	const getIconPath = (type: string) => {
		switch (type) {
			case 'success':
				return 'M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z';
			case 'error':
				return 'M10 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z';
			case 'info':
			default:
				return 'M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z';
		}
	};
</script>

{#if visible}
	<div class="fixed top-4 right-4 z-50 w-full max-w-sm" transition:fly={{ x: 300, duration: 200 }}>
		<div class="rounded-md border p-4 shadow-md {getTypeStyles(type)} transition-all duration-200">
			<div class="flex items-start">
				<div class="flex-shrink-0">
					<svg
						class="h-5 w-5"
						fill="none"
						stroke="currentColor"
						viewBox="0 0 24 24"
						aria-hidden="true"
					>
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d={getIconPath(type)}
						/>
					</svg>
				</div>
				<div class="ml-3 flex-1">
					<p class="text-sm font-medium">
						{message}
					</p>
				</div>
				<div class="ml-4 flex-shrink-0">
					<button
						type="button"
						onclick={handleClose}
						class="inline-flex rounded-md p-1.5 transition-colors duration-150 hover:bg-gray-100 focus:ring-2 focus:ring-gray-300 focus:outline-none"
						aria-label="關閉通知"
					>
						<svg class="h-4 w-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								stroke-width="2"
								d="M6 18L18 6M6 6l12 12"
							/>
						</svg>
					</button>
				</div>
			</div>
		</div>
	</div>
{/if}
