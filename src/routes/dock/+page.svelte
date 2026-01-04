<script>
	import { onMount, onDestroy } from 'svelte';

	let videoSource = null;
	let canvasRef = null;
	let stream = null;
	let logs = $state([]);
	let selectedLog = $state(null);
	let isMonitoring = $state(false);
	let lastFrameData = null;
	let motionMagnitude = $state(0);
	
	// Sensor Data State
	let sensorData = $state({
		accel: { x: 0, y: 0, z: 0 },
		gyro: { alpha: 0, beta: 0, gamma: 0 },
		motion: 0
	});

	const MOTION_THRESHOLD = 25; // Minimum avg pixel difference to trigger
	const SAMPLE_STEP = 4; // Sample every 4th pixel for performance
	const PROCESS_WIDTH = 160;
	const PROCESS_HEIGHT = 120;
	const WEBHOOK_URL = 'https://fahim-n8n.laddu.cc';

	onMount(() => {
		const link = document.createElement('link');
		link.href = 'https://fonts.googleapis.com/css2?family=Fira+Code:wght@300;400;500&display=swap';
		link.rel = 'stylesheet';
		document.head.appendChild(link);
		
		startCamera();
		startSensors();
	});

	onDestroy(() => {
		stopCamera();
		stopSensors();
	});

	async function startCamera() {
		if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
			try {
				// Try facingMode: 'environment' (back camera) or 'user'
				stream = await navigator.mediaDevices.getUserMedia({ 
					video: { 
						width: 640, 
						height: 480, 
						facingMode: 'user' // Try front camera first
					},
					audio: false
				});
				videoSource.srcObject = stream;
				videoSource.play();
				isMonitoring = true;
				processFrame();
			} catch (err) {
				console.error("Primary camera error:", err);
				// Fallback: Try any camera without constraint
				try {
					stream = await navigator.mediaDevices.getUserMedia({ video: true });
					videoSource.srcObject = stream;
					videoSource.play();
					isMonitoring = true;
					processFrame();
				} catch (e2) {
					console.error("Fallback camera error:", e2);
					addLog(Date.now(), "System", "Camera Access Failed");
				}
			}
		} else {
			console.error("getUserMedia not supported");
			addLog(Date.now(), "System", "Camera API Unavailable");
		}
	}

	function stopCamera() {
		isMonitoring = false;
		if (stream) {
			stream.getTracks().forEach(track => track.stop());
		}
	}

	function startSensors() {
		if (typeof window !== 'undefined' && window.DeviceMotionEvent) {
			window.addEventListener('devicemotion', handleMotion, true);
			window.addEventListener('deviceorientation', handleOrientation, true);
		}
	}

	function stopSensors() {
		if (typeof window !== 'undefined' && window.DeviceMotionEvent) {
			window.removeEventListener('devicemotion', handleMotion, true);
			window.removeEventListener('deviceorientation', handleOrientation, true);
		}
	}

	function handleMotion(event) {
		const { x, y, z } = event.accelerationIncludingGravity || { x:0, y:0, z:0 };
		sensorData.accel = { 
			x: (x || 0).toFixed(2), 
			y: (y || 0).toFixed(2), 
			z: (z || 0).toFixed(2) 
		};
		
		// Simple magnitude calculation
		const mag = Math.sqrt(x*x + y*y + z*z);
		sensorData.motion = mag.toFixed(2);
	}

	function handleOrientation(event) {
		sensorData.gyro = {
			alpha: (event.alpha || 0).toFixed(1),
			beta: (event.beta || 0).toFixed(1),
			gamma: (event.gamma || 0).toFixed(1)
		};
	}

	function addLog(id, type, val) {
		const timestamp = new Date().toLocaleTimeString();
		const logItem = {
			id: id,
			time: timestamp,
			type: type || "INFO",
			intensity: val || "N/A",
			image: null
		};
		logs = [logItem, ...logs].slice(0, 50);
	}

	function processFrame() {
		if (!isMonitoring || !canvasRef || !videoSource) return;

		const ctx = canvasRef.getContext('2d', { willReadFrequently: true });
		
		// Draw downscaled frame
		ctx.drawImage(videoSource, 0, 0, PROCESS_WIDTH, PROCESS_HEIGHT);
		const currentFrame = ctx.getImageData(0, 0, PROCESS_WIDTH, PROCESS_HEIGHT);
		
		if (lastFrameData) {
			calculateMotion(currentFrame.data, lastFrameData.data);
		}

		lastFrameData = new ImageData(
			new Uint8ClampedArray(currentFrame.data),
			currentFrame.width,
			currentFrame.height
		);

		requestAnimationFrame(processFrame);
	}

	function calculateMotion(current, previous) {
		let totalDiff = 0;
		let pixelCount = 0;

		for (let i = 0; i < current.length; i += SAMPLE_STEP * 4) {
			const rDiff = Math.abs(current[i] - previous[i]);
			const gDiff = Math.abs(current[i+1] - previous[i+1]);
			const bDiff = Math.abs(current[i+2] - previous[i+2]);
			
			totalDiff += (rDiff + gDiff + bDiff) / 3;
			pixelCount++;
		}

		const avgDiff = totalDiff / pixelCount;
		motionMagnitude = avgDiff;

		if (avgDiff > MOTION_THRESHOLD) {
			handleMotionDetected(avgDiff);
		}
	}

	let lastAlertTime = 0;
	const ALERT_COOLDOWN = 2000;

	async function handleMotionDetected(intensity) {
		const now = Date.now();
		if (now - lastAlertTime < ALERT_COOLDOWN) return;
		lastAlertTime = now;

		const timestamp = new Date().toLocaleTimeString();
		const snapshot = canvasRef.toDataURL('image/jpeg', 0.7);

		const logItem = {
			id: now,
			time: timestamp,
			type: "MOTION DETECTED",
			intensity: intensity.toFixed(2),
			image: snapshot
		};
		logs = [logItem, ...logs].slice(0, 50);

		try {
			fetch(WEBHOOK_URL, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({
					event_type: "motion_alert",
					intensity: intensity,
					frame: snapshot,
					sensors: sensorData // Include sensor data in payload
				})
			}).catch(err => console.error("Webhook failed", err));
		} catch (e) {
			console.error("Alert error", e);
		}
	}

	function selectLog(log) {
		selectedLog = log;
	}

	function closeLog() {
		selectedLog = null;
	}
</script>

<div class="container">
	<header>
		<h1>DOCK_MONITOR</h1>
		<div class="status">
			<span class="indicator" class:active={isMonitoring}></span>
			{isMonitoring ? 'ACTIVE' : 'OFFLINE'}
		</div>
	</header>

	<main>
		<!-- Camera Section -->
		<section class="camera-feed">
			<div class="video-wrapper">
				<!-- svelte-ignore a11y_media_has_caption -->
				<video bind:this={videoSource} playsinline muted autoplay></video>
				<canvas bind:this={canvasRef} width={PROCESS_WIDTH} height={PROCESS_HEIGHT} class="hidden-canvas"></canvas>
				<div class="overlay">
					<div class="metric">
						<label>VISUAL_MAG</label>
						<span>{motionMagnitude.toFixed(1)}</span>
					</div>
				</div>
			</div>
		</section>

		<!-- Sensor Data Table -->
		<section class="sensor-panel">
			<div class="panel-header">SENSOR_TELEMETRY</div>
			<div class="table-wrapper">
				<table>
					<thead>
						<tr>
							<th>SENSOR</th>
							<th>X / ALPHA</th>
							<th>Y / BETA</th>
							<th>Z / GAMMA</th>
						</tr>
					</thead>
					<tbody>
						<tr>
							<td>ACCEL</td>
							<td>{sensorData.accel.x}</td>
							<td>{sensorData.accel.y}</td>
							<td>{sensorData.accel.z}</td>
						</tr>
						<tr>
							<td>GYRO</td>
							<td>{sensorData.gyro.alpha}</td>
							<td>{sensorData.gyro.beta}</td>
							<td>{sensorData.gyro.gamma}</td>
						</tr>
						<tr>
							<td>TOTAL</td>
							<td colspan="3">MAGNITUDE: {sensorData.motion}</td>
						</tr>
					</tbody>
				</table>
			</div>
		</section>

		<!-- Logs Section -->
		<section class="logs-panel">
			<div class="logs-header">
				<h2>EVENT_LOG</h2>
			</div>
			<div class="logs-list">
				{#each logs as log (log.id)}
					<button class="log-item" onclick={() => selectLog(log)}>
						<span class="time">{log.time}</span>
						<span class="type">{log.type}</span>
						<span class="value">INT: {log.intensity}</span>
					</button>
				{/each}
				{#if logs.length === 0}
					<div class="empty-state">NO EVENTS RECORDED</div>
				{/if}
			</div>
		</section>

		<div class="nav-container">
			<a href="/" class="btn btn-secondary">
				<span class="arrow">←</span>
				<span class="label">RETURN_TO_HUB</span>
			</a>
		</div>
	</main>

	{#if selectedLog}
		<div class="modal-backdrop" onclick={closeLog} role="button" tabindex="0" onkeydown={(e) => e.key === 'Escape' && closeLog()}>
			<div class="modal-content" onclick={(e) => e.stopPropagation()} role="presentation">
				<div class="modal-header">
					<h3>EVENT_{selectedLog.id}</h3>
					<button onclick={closeLog}>[CLOSE]</button>
				</div>
				{#if selectedLog.image}
					<img src={selectedLog.image} alt="Capture" />
				{/if}
				<div class="modal-info">
					<p>TIME: {selectedLog.time}</p>
					<p>TYPE: {selectedLog.type}</p>
					<p>INTENSITY: {selectedLog.intensity}</p>
				</div>
			</div>
		</div>
	{/if}
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
		background-color: #333;
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
		background-color: #F9F9F9;
		border: 1px solid #F0F0F0;
		padding: 10px;
	}

	.video-wrapper {
		width: 100%;
		aspect-ratio: 4/3;
		background: #000;
		position: relative;
		overflow: hidden;
	}

	video {
		width: 100%;
		height: 100%;
		object-fit: cover;
		opacity: 0.9;
	}

	.hidden-canvas {
		display: none;
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

	/* Sensor Panel */
	.sensor-panel {
		border: 1px solid #F0F0F0;
		padding: 0;
	}

	.panel-header {
		background: #F9F9F9;
		padding: 8px 12px;
		font-size: 0.85rem;
		border-bottom: 1px solid #F0F0F0;
		color: #444;
	}

	.table-wrapper {
		padding: 10px;
	}

	table {
		width: 100%;
		border-collapse: collapse;
		font-size: 0.8rem;
	}

	th {
		text-align: left;
		padding: 6px;
		color: #888;
		font-weight: 400;
		border-bottom: 1px solid #EEE;
	}

	td {
		padding: 6px;
		color: #333;
	}

	/* Logs */
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
		padding: 10px 12px;
		border: none;
		border-bottom: 1px solid #F5F5F5;
		background: none;
		width: 100%;
		cursor: pointer;
		font-family: inherit;
		font-size: 0.8rem;
		color: #333;
		text-align: left;
		transition: background-color 0.1s;
	}

	.log-item:hover {
		background-color: #F9F9F9;
	}

	.time {
		color: #888;
		width: 80px;
	}

	.type {
		flex: 1;
		font-weight: 500;
	}

	.value {
		color: #666;
	}

	.empty-state {
		padding: 20px;
		text-align: center;
		color: #CCC;
		font-size: 0.8rem;
	}

	.nav-container {
		display: flex;
		flex-direction: column;
		gap: 15px;
		padding: 2rem;
		background: #FAFAFA;
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

	.modal-backdrop {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background: rgba(255, 255, 255, 0.8);
		display: flex;
		justify-content: center;
		align-items: center;
		z-index: 100;
	}

	.modal-content {
		background: #FFF;
		border: 1px solid #DDD;
		padding: 20px;
		width: 90%;
		max-width: 500px;
		box-shadow: 0 10px 30px rgba(0,0,0,0.05);
	}

	.modal-header {
		display: flex;
		justify-content: space-between;
		margin-bottom: 15px;
	}

	.modal-header h3 {
		margin: 0;
		font-size: 0.9rem;
	}

	.modal-header button {
		background: none;
		border: none;
		cursor: pointer;
		font-family: inherit;
		font-size: 0.9rem;
		text-decoration: underline;
	}

	.modal-content img {
		width: 100%;
		border: 1px solid #EEE;
		margin-bottom: 15px;
	}

	.modal-info p {
		margin: 5px 0;
		font-size: 0.8rem;
		color: #555;
	}
</style>