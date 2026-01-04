<script>
	import { onMount, onDestroy } from 'svelte';

	let sensorData = $state({
		accel: { x: 0, y: 0, z: 0 },
		gyro: { alpha: 0, beta: 0, gamma: 0 },
		motion: 0
	});

	onMount(() => {
		const link = document.createElement('link');
		link.href = 'https://fonts.googleapis.com/css2?family=Fira+Code:wght@300;400;500&display=swap';
		link.rel = 'stylesheet';
		document.head.appendChild(link);

		startSensors();
	});

	onDestroy(() => {
		stopSensors();
	});

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
</script>

<div class="container">
	<header>
		<h1>OTG_TOOLS</h1>
		<div class="status">SENSORS_ACTIVE</div>
	</header>

	<main>
		<section class="sensor-panel">
			<div class="panel-header">DEVICE_TELEMETRY</div>
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
						<tr class="highlight">
							<td>TOTAL</td>
							<td colspan="3">MAGNITUDE: {sensorData.motion}</td>
						</tr>
					</tbody>
				</table>
			</div>
		</section>

		<div class="info-block">
			<p>MONITORING_DEVICE_MOVEMENT</p>
			<p class="sub-text">DATA_STREAM_ACTIVE</p>
		</div>

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
		color: #888;
	}

	main {
		flex: 1;
		display: flex;
		flex-direction: column;
		gap: 20px;
	}

	/* Sensor Panel */
	.sensor-panel {
		border: 1px solid #F0F0F0;
		padding: 0;
		margin-top: 10px;
	}

	.panel-header {
		background: #F9F9F9;
		padding: 10px 12px;
		font-size: 0.85rem;
		border-bottom: 1px solid #F0F0F0;
		color: #444;
		font-weight: 500;
	}

	.table-wrapper {
		padding: 10px;
	}

	table {
		width: 100%;
		border-collapse: collapse;
		font-size: 0.85rem;
	}

	th {
		text-align: left;
		padding: 8px;
		color: #888;
		font-weight: 400;
		border-bottom: 1px solid #EEE;
		font-size: 0.75rem;
	}

	td {
		padding: 8px;
		color: #333;
		border-bottom: 1px solid #F9F9F9;
	}

	tr:last-child td {
		border-bottom: none;
	}

	.highlight td {
		font-weight: 500;
		background-color: #FAFAFA;
	}

	.info-block {
		text-align: center;
		padding: 20px;
		color: #444;
	}

	.sub-text {
		color: #888;
		font-size: 0.8rem;
		margin-top: 5px;
	}

	.nav-container {
		margin-top: auto;
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
</style>
