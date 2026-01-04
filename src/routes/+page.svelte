<script>
	import { onMount } from 'svelte';

	onMount(() => {
		const link = document.createElement('link');
		link.href = 'https://fonts.googleapis.com/css2?family=Fira+Code:wght@300;400;500&display=swap';
		link.rel = 'stylesheet';
		document.head.appendChild(link);
	});
</script>

<div class="container">
	<header>
		<h1>HELPOSZ_HUB</h1>
		<div class="status">SYSTEM_READY</div>
	</header>

	<main>
		<p class="description">SELECT_OPERATION_MODE</p>
		
		<div class="nav-container">
			<a href="/dock" class="btn btn-primary">
				<span class="label">LAUNCH_DOCK_MONITOR</span>
				<span class="arrow">→</span>
			</a>
			
			<a href="/otg" class="btn btn-secondary">
				<span class="label">OPEN_OTG_TOOLS</span>
				<span class="arrow">→</span>
			</a>
		</div>
	</main>
</div>

<style>
	:global(body) {
		margin: 0;
		background-color: #FFFFFF;
		color: #111111;
		font-family: 'Fira Code', monospace;
	}

	.container {
		display: flex;
		flex-direction: column;
		height: 100vh;
		max-width: 800px;
		margin: 0 auto;
		padding: 20px;
		box-sizing: border-box;
	}

	header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 40px;
		border-bottom: 1px solid #F0F0F0;
		padding-bottom: 10px;
	}

	h1 {
		font-size: 1.2rem;
		font-weight: 500;
		margin: 0;
		letter-spacing: -0.5px;
	}

	.status {
		font-size: 0.8rem;
		color: #888;
	}

	main {
		flex: 1;
		display: flex;
		flex-direction: column;
		justify-content: center;
	}

	.description {
		font-size: 0.9rem;
		color: #666;
		margin-bottom: 20px;
		text-align: center;
	}

	.nav-container {
		display: flex;
		flex-direction: column;
		gap: 15px;
		padding: 2rem;
	}

	.btn {
		display: flex;
		justify-content: space-between;
		align-items: center;
		width: 100%;
		padding: 16px 20px;
		font-family: inherit;
		font-size: 0.9rem;
		text-decoration: none;
		cursor: pointer;
		box-sizing: border-box;
		transition: all 0.2s ease;
		border: 1px solid #111;
	}

	.btn-primary {
		background-color: #111;
		color: #FFF;
	}

	.btn-primary:hover {
		background-color: #333;
	}

	.btn-secondary {
		background-color: #FFF;
		color: #111;
	}

	.btn-secondary:hover {
		background-color: #F5F5F5;
	}

	.arrow {
		font-weight: 300;
	}
</style>