<script>
	import { onMount, onDestroy } from 'svelte';
	import { Toast } from '@capacitor/toast';

	// Constants
	const WEBHOOK_URL = "http://fahim-n8n.laddu.cc/webhook/seizure-monitor";
	const SESSION_ID = crypto.randomUUID();

	let videoSource = null;
	let canvasRef = null;
	let isMonitoring = $state(false);
	let frameStreamInterval = null;
	
	// Logs for the UI (keeping it simple)
	let logs = $state([]);
	
	onMount(async () => {
		const link = document.createElement('link');
		link.href = 'https://fonts.googleapis.com/css2?family=Fira+Code:wght@300;400;500&display=swap';
		link.rel = 'stylesheet';
		document.head.appendChild(link);
		
		startCamera();
	});

	onDestroy(() => {
		stopMonitoring();
	});

	function stopMonitoring() {
		isMonitoring = false;
		stopFrameStream();
		if (videoSource && videoSource.srcObject) {
			videoSource.srcObject.getTracks().forEach(t => t.stop());
		}
	}

	async function startCamera() {
		try {
			if (videoSource) {
				const stream = await navigator.mediaDevices.getUserMedia({
					video: {
						width: { ideal: 1920 },
						height: { ideal: 1080 },
						facingMode: 'environment'
					},
					audio: false
				});
				videoSource.srcObject = stream;
				await videoSource.play();

				isMonitoring = true;
				startFrameStream();
				drawLoop();
			}
		} catch (e) {
			console.error("Camera Init Error", e);
			addLog("SYSTEM", "CAMERA_FAIL");
		}
	}

	function drawLoop() {
		if (!isMonitoring || !canvasRef || !videoSource) return;
		
		const ctx = canvasRef.getContext('2d');
		if (videoSource.videoWidth) {
			canvasRef.width = videoSource.videoWidth;
			canvasRef.height = videoSource.videoHeight;
			ctx.drawImage(videoSource, 0, 0, canvasRef.width, canvasRef.height);
		}
		
		requestAnimationFrame(drawLoop);
	}

	function startFrameStream() {
		if (frameStreamInterval) return;
		// 1500ms interval
		frameStreamInterval = setInterval(sendFrame, 1500);
	}

	function stopFrameStream() {
		if (frameStreamInterval) {
			clearInterval(frameStreamInterval);
			frameStreamInterval = null;
		}
	}

	async function sendFrame() {
		if (!canvasRef) return;
		
		try {
			// Convert canvas to base64
			const frame_data = canvasRef.toDataURL('image/jpeg', 0.6); // Slightly lower quality for network
			const timestamp = new Date().toISOString();

			const payload = {
				"sessionId": SESSION_ID,
				"image": frame_data,
				"timestamp": timestamp
			};

			// Show Native Toast
			await Toast.show({
				text: 'STREAM_PACKET_SENT',
				duration: 'short',
				position: 'bottom'
			});
			
			// Update UI Log
			addLog("CLOUD", "PACKET_SENT");

			// Fire and forget fetch
			fetch(WEBHOOK_URL, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(payload)
			}).catch(err => {
				console.error("Frame send error:", err);
				// addLog("CLOUD", "SEND_FAIL"); // Optional: don't spam logs with errors
			});

		} catch (e) {
			console.error("Error generating frame:", e);
		}
	}

	function addLog(type, msg) {
		const timestamp = new Date().toLocaleTimeString();
		logs = [{id: Date.now(), time: timestamp, type, msg}, ...logs].slice(0, 20);
	}
</script>

<div class="container">
	<header>
		<h1>PREMIUM_MONITOR</h1>
		<div class="status">
			<span class="indicator" class:active={isMonitoring}></span>
			{isMonitoring ? "STREAMING" : "OFFLINE"}
		</div>
	</header>

	<main>
		<section class="camera-feed">
			<div class="video-wrapper">
				<!-- svelte-ignore a11y_media_has_caption -->
				<video bind:this={videoSource} class="hidden-video" playsinline muted autoplay></video>
				<canvas bind:this={canvasRef}></canvas>
				<div class="overlay">
					<div class="metric">
						<label>CLOUD_UPLINK</label>
						<span style="color: blue; font-weight: bold;">ACTIVE</span>
					</div>
				</div>
			</div>
		</section>

		<section class="logs-panel">
			<div class="logs-header">
				<h2>TRANSMISSION_LOG</h2>
			</div>
			<div class="logs-list">
				{#each logs as log (log.id)}
					<div class="log-item">
						<span class="time">{log.time}</span>
						<span class="type">{log.type}</span>
						<span class="value">{log.msg}</span>
					</div>
				{/each}
			</div>
		</section>

		<div class="nav-container">
			<a href="/" class="btn btn-secondary">
				<span class="arrow">←</span>
				<span class="label">RETURN_TO_HUB</span>
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
		margin-bottom: 20px;
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
		display: flex;
		align-items: center;
		gap: 8px;
		color: #666;
	}

	.indicator {
		width: 8px;
		height: 8px;
		background-color: #E0E0E0;
		border-radius: 50%;
	}
	
	.indicator.active {
		background-color: #00FF00; /* Bright Green */
		box-shadow: 0 0 5px #00FF00;
	}

	main {
		display: flex;
		flex-direction: column;
		gap: 20px;
		flex: 1;
		overflow: hidden;
	}

	.camera-feed {
		position: relative;
		background-color: transparent;
		border: none;
		padding: 0;
	}

	.video-wrapper {
		width: 100%;
		aspect-ratio: 16/9;
		background: #000;
		position: relative;
		overflow: hidden;
	}

	.hidden-video {
		display: none;
	}

	canvas {
		width: 100%;
		height: 100%;
		object-fit: cover;
	}

	.overlay {
		position: absolute;
		bottom: 10px;
		left: 10px;
		background: rgba(255, 255, 255, 0.9);
		padding: 4px 8px;
		border: 1px solid #E0E0E0;
	}

	.metric {
		font-size: 0.75rem;
		display: flex;
		gap: 8px;
	}

	.nav-container {
		margin-top: auto;
		display: flex;
		flex-direction: column;
		gap: 15px;
		padding: 2rem;
		background: #FFFFFF;
		border-top: 1px solid #F0F0F0;
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

	.logs-panel {
		flex: 1;
		display: flex;
		flex-direction: column;
		border: 1px solid #F0F0F0;
		overflow: hidden;
	}

	.logs-header {
		background-color: #F9F9F9;
		padding: 8px 12px;
		border-bottom: 1px solid #F0F0F0;
	}

	.logs-header h2 {
		margin: 0;
		font-size: 0.85rem;
		font-weight: 400;
		color: #444;
	}

	.logs-list {
		flex: 1;
		overflow-y: auto;
		display: flex;
		flex-direction: column;
	}

	.log-item {
		display: flex;
		justify-content: space-between;
		padding: 8px 12px;
		border-bottom: 1px solid #F5F5F5;
		font-size: 0.75rem;
		color: #333;
	}

	.time { color: #888; width: 70px; }
	.type { font-weight: 500; width: 60px; }
	.value { color: #555; flex: 1; text-align: right; }
</style>