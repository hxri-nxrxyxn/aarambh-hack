<script>
	import { onMount, onDestroy } from 'svelte';
	import { KeepAwake } from '@capacitor-community/keep-awake';
	import { VolumeControl, VolumeType } from '@odion-cloud/capacitor-volume-control';
	import { Geolocation } from '@capacitor/geolocation';

	const WEBHOOK_URL = "http://fahim-n8n.laddu.cc/webhook/telegram-publish";

	let sensorData = $state({
		accel: { x: 0, y: 0, z: 0 },
		gyro: { alpha: 0, beta: 0, gamma: 0 },
		motion: 0
	});

	let isAlarmActive = $state(false);
	
	// Audio State
	let audioContext = null;
	let oscillator = null;
	let pulseInterval = null;
	let beepTimeout = null;
	let emergencyAudio = null;
	let playCount = 0;

	const FALL_THRESHOLD = 30; // ~3g impact

	onMount(async () => {
		const link = document.createElement('link');
		link.href = 'https://fonts.googleapis.com/css2?family=Fira+Code:wght@300;400;500&display=swap';
		link.rel = 'stylesheet';
		document.head.appendChild(link);

		startSensors();
		try {
			await KeepAwake.keepAwake();
		} catch (e) {
			console.error("KeepAwake failed", e);
		}
	});

	onDestroy(async () => {
		stopSensors();
		stopAlarm();
		try {
			await KeepAwake.allowSleep();
		} catch (e) {}
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

		if (mag > FALL_THRESHOLD) {
			triggerAlarm();
		}
	}

	function handleOrientation(event) {
		sensorData.gyro = {
			alpha: (event.alpha || 0).toFixed(1),
			beta: (event.beta || 0).toFixed(1),
			gamma: (event.gamma || 0).toFixed(1)
		};
	}

	async function triggerAlarm() {
		if (isAlarmActive) return;
		isAlarmActive = true;
		
		// 1. Set MAX Volume
		try {
			VolumeControl.setVolumeLevel({ type: VolumeType.Music, value: 1.0 });
			VolumeControl.setVolumeLevel({ type: VolumeType.System, value: 1.0 });
		} catch (e) {
			console.error("Failed to set max volume", e);
		}
		
		// 2. Play Alarm Sequence
		playHighPitchSound();

		// 3. Send Location Webhook
		try {
			// Check permissions first? Capacitor handles this usually.
			const coordinates = await Geolocation.getCurrentPosition({
				enableHighAccuracy: true,
				timeout: 10000
			});

			const { latitude, longitude } = coordinates.coords;
			const mapLink = `https://maps.google.com/?q=${latitude},${longitude}`;

			// Payload
			const payload = {
				type: "FALL_DETECTED",
				timestamp: new Date().toISOString(),
				location: mapLink,
				lat: latitude,
				lng: longitude
			};

			// Send
			await fetch(WEBHOOK_URL, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(payload)
			});
			console.log("Location sent:", mapLink);

		} catch (err) {
			console.error("Geolocation/Webhook Error:", err);
		}
	}

	function playHighPitchSound() {
		try {
			const AudioContext = window.AudioContext || window.webkitAudioContext;
			if (!AudioContext) return;

			audioContext = new AudioContext();
			oscillator = audioContext.createOscillator();
			const gainNode = audioContext.createGain();

			oscillator.type = 'square';
			oscillator.frequency.setValueAtTime(3000, audioContext.currentTime);
			
			oscillator.frequency.exponentialRampToValueAtTime(4000, audioContext.currentTime + 0.1);
			oscillator.frequency.exponentialRampToValueAtTime(3000, audioContext.currentTime + 0.2);

			gainNode.gain.setValueAtTime(1, audioContext.currentTime); 

			oscillator.connect(gainNode);
			gainNode.connect(audioContext.destination);

			oscillator.start();
			
			pulseInterval = setInterval(() => {
				if (oscillator && audioContext) {
					const now = audioContext.currentTime;
					oscillator.frequency.setValueAtTime(3000, now);
					oscillator.frequency.linearRampToValueAtTime(4000, now + 0.1);
				}
			}, 200);

			beepTimeout = setTimeout(() => {
				stopOscillator();
				playEmergencyAudio();
			}, 8000);

		} catch (e) {
			console.error("Audio error", e);
		}
	}

	function stopOscillator() {
		if (pulseInterval) {
			clearInterval(pulseInterval);
			pulseInterval = null;
		}
		if (oscillator) {
			try {
				oscillator.stop();
				oscillator.disconnect();
			} catch (e) {}
			oscillator = null;
		}
		if (audioContext) {
			try {
				audioContext.close();
			} catch (e) {}
			audioContext = null;
		}
	}

	function playEmergencyAudio() {
		if (emergencyAudio) return;

		playCount = 0;
		emergencyAudio = new Audio('/emergency_guide.wav');
		
		emergencyAudio.addEventListener('ended', () => {
			playCount++;
			if (playCount < 3) {
				emergencyAudio.currentTime = 0;
				emergencyAudio.play();
			} else {
				emergencyAudio = null;
			}
		});

		emergencyAudio.play().catch(e => console.error("Play failed", e));
	}

	function stopAlarm() {
		isAlarmActive = false;
		
		if (beepTimeout) {
			clearTimeout(beepTimeout);
			beepTimeout = null;
		}
		stopOscillator();

		if (emergencyAudio) {
			emergencyAudio.pause();
			emergencyAudio = null;
		}
		playCount = 0;
	}
</script>

<div class="container" class:alert-mode={isAlarmActive}>
	<header>
		<h1>OTG_TOOLS</h1>
		<div class="status">{isAlarmActive ? 'CRITICAL_ALERT' : 'SENSORS_ACTIVE'}</div>
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
			{#if isAlarmActive}
				<p class="alert-text">FALL_DETECTED</p>
				<button class="btn btn-danger" onclick={stopAlarm}>STOP_ALARM</button>
			{:else}
				<p>MONITORING_DEVICE_MOVEMENT</p>
				<p class="sub-text">DATA_STREAM_ACTIVE</p>
			{/if}
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
		transition: background-color 0.3s;
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

	.container.alert-mode {
		background-color: #FFE0E0; /* Light red background */
	}

	.container.alert-mode :global(body) {
		background-color: #FFE0E0;
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

	.alert-mode .status {
		color: #D32F2F;
		font-weight: bold;
		animation: blink 1s infinite;
	}

	@keyframes blink {
		50% { opacity: 0; }
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
		background: #FFF;
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
		flex: 1;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
	}

	.sub-text {
		color: #888;
		font-size: 0.8rem;
		margin-top: 5px;
	}

	.alert-text {
		color: #D32F2F;
		font-size: 1.2rem;
		font-weight: bold;
		margin-bottom: 20px;
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

	.btn-danger {
		background-color: #D32F2F;
		color: #FFF;
		border: 1px solid #B71C1C;
		justify-content: center;
		font-weight: bold;
	}

	.btn-danger:hover {
		background-color: #B71C1C;
	}

	.arrow {
		font-weight: 300;
	}
</style>