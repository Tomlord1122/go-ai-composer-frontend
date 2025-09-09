<script lang="ts">
	import { toastStore } from '../stores/toastStore.js';
	import Toast from './Toast.svelte';

	let toasts = $state(toastStore);

	function handleToastClose(id: string) {
		toastStore.remove(id);
	}
</script>

<div class="fixed top-4 right-4 z-50 space-y-2">
	{#each $toasts as toast (toast.id)}
		<Toast
			message={toast.message}
			type={toast.type}
			visible={toast.visible}
			onclose={() => handleToastClose(toast.id)}
		/>
	{/each}
</div>
