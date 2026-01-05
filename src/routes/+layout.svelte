<script>
	import favicon from '$lib/assets/favicon.svg';
	import { onMount } from 'svelte';
	import { App } from '@capacitor/app';
	import { StatusBar } from '@capacitor/status-bar';

	let { children } = $props();

	onMount(() => {
		StatusBar.hide().catch(() => {});

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