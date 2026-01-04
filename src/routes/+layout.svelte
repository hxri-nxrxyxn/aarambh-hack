<script>
	import favicon from '$lib/assets/favicon.svg';
	import { onMount } from 'svelte';
	import { App } from '@capacitor/app';

	let { children } = $props();

	onMount(() => {
		// Hardware back button handling
		const backListener = App.addListener('backButton', ({ canGoBack }) => {
			if (canGoBack) {
				window.history.back();
			} else {
				App.exitApp();
			}
		});

		return () => {
			backListener.then(l => l.remove());
		};
	});
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

{@render children()}