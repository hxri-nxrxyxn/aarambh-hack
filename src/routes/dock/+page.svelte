<script>
	import { onMount, onDestroy } from 'svelte';
	import { Pose } from '@mediapipe/pose';
	import { Camera } from '@mediapipe/camera_utils';
	// Drawing utils might be needed for debug, but user's code uses window.drawConnectors if available.
	// We will implement custom drawing or just trust logic.

	let videoSource = null;
	let canvasRef = null;
	let logs = $state([]);
	let selectedLog = $state(null);
	let isMonitoring = $state(false);
	
	let statusText = $state("SAFE");
	let statusColor = $state("green");

	// Logic State
	let isBeeping = false;
	let activeEvent = null;
	let fallStartTime = null;
	let seizureStartTime = null;
	let lastNoseY = null;
	let rapidFallDetected = false;
	let lastAnchor = null;
	
	let history = {
		leftWrist: [], rightWrist: [],
		leftElbow: [], rightElbow: [],
		mouth: [], nose: []
	};
	const HISTORY_SIZE = 30;

	// Audio
	let audioCtx = null;
	let oscillator = null;

	// Config (from hidden inputs in original)
	const fallThresholdY = 0.7; // 70%
	const fallDurationMs = 2000;
	const seizureTortuosityThresh = 3.0;
	const anchorStabilityThresh = 0.02;
	const seizureDurationMs = 1500;

	let camera = null;
	let pose = null;

	onMount(async () => {
		const link = document.createElement('link');
		link.href = 'https://fonts.googleapis.com/css2?family=Fira+Code:wght@300;400;500&display=swap';
		link.rel = 'stylesheet';
		document.head.appendChild(link);
		
		initMediaPipe();
	});

	onDestroy(() => {
		stopMonitoring();
	});

	function initAudio() {
		if (typeof window !== 'undefined') {
			if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
			if (audioCtx.state === 'suspended') audioCtx.resume();
		}
	}

	function startBeep() {
		initAudio();
		if (isBeeping || !audioCtx) return;
		isBeeping = true;
		oscillator = audioCtx.createOscillator();
		const gainNode = audioCtx.createGain();
		oscillator.type = 'sine';
		oscillator.frequency.setValueAtTime(1000, audioCtx.currentTime);
		gainNode.gain.setValueAtTime(0.5, audioCtx.currentTime);
		oscillator.connect(gainNode);
		gainNode.connect(audioCtx.destination);
		oscillator.start();
	}

	function stopBeep() {
		if (oscillator) {
			try { oscillator.stop(); } catch(e){}
			oscillator = null;
		}
		isBeeping = false;
	}

	function stopMonitoring() {
		isMonitoring = false;
		stopBeep();
		if (camera) camera.stop();
		if (pose) pose.close();
	}

	async function initMediaPipe() {
		try {
			pose = new Pose({
				locateFile: (file) => {
					// We need to serve these files or use a CDN. 
					// Using jsdelivr for reliability in this context if local node_modules path is tricky in static build.
					return `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`;
				}
			});
			
			pose.setOptions({
				modelComplexity: 1,
				smoothLandmarks: true,
				enableSegmentation: false,
				minDetectionConfidence: 0.5,
				minTrackingConfidence: 0.5
			});
			
			pose.onResults(onResults);

			if (videoSource) {
				camera = new Camera(videoSource, {
					onFrame: async () => { await pose.send({image: videoSource}); },
					width: 640,
					height: 480
				});
				camera.start();
				isMonitoring = true;
			}
		} catch (e) {
			console.error("MediaPipe Init Error", e);
			addLog(Date.now(), "SYSTEM", "MediaPipe Init Failed");
		}
	}

	function onResults(results) {
		if (!canvasRef) return;
		const ctx = canvasRef.getContext('2d');
		ctx.save();
		ctx.clearRect(0, 0, canvasRef.width, canvasRef.height);
		
		// Draw video
		ctx.drawImage(results.image, 0, 0, canvasRef.width, canvasRef.height);

		if (results.poseLandmarks) {
			processLandmarks(results.poseLandmarks, ctx);
		}
		
		ctx.restore();
	}

	function processLandmarks(landmarks, ctx) {
		// Helpers
		const nose = landmarks[0];
		const mouthLeft = landmarks[9];
		const mouthRight = landmarks[10];
		const mouth = { x: (mouthLeft.x + mouthRight.x)/2, y: (mouthLeft.y + mouthRight.y)/2 };
		
		const leftShoulder = landmarks[11];
		const rightShoulder = landmarks[12];
		const leftElbow = landmarks[13];
		const rightElbow = landmarks[14];
		const leftWrist = landmarks[15];
		const rightWrist = landmarks[16];

		const anchor = { 
			x: (leftShoulder.x + rightShoulder.x) / 2,
			y: (leftShoulder.y + rightShoulder.y) / 2
		};

		// 1. Fall Detection
		let noseVelocity = 0;
		if (lastNoseY !== null) {
			noseVelocity = (nose.y - lastNoseY) * 30;
		}
		lastNoseY = nose.y;

		const RAPID_FALL_VELOCITY = 0.5;
		let isFall = false;

		if (nose.y > fallThresholdY) {
			if (!fallStartTime) fallStartTime = Date.now();
			else if (Date.now() - fallStartTime > fallDurationMs) isFall = true;

			if (rapidFallDetected && (Date.now() - fallStartTime > 500)) {
				isFall = true;
			}
		} else {
			fallStartTime = null;
			rapidFallDetected = false;
		}

		if (noseVelocity > RAPID_FALL_VELOCITY && nose.y < fallThresholdY) {
			rapidFallDetected = true;
		}

		// Draw Fall Line
		const thresholdY = fallThresholdY * canvasRef.height;
		ctx.beginPath();
		ctx.setLineDash([5, 5]);
		ctx.moveTo(0, thresholdY);
		ctx.lineTo(canvasRef.width, thresholdY);
		ctx.strokeStyle = 'rgba(255, 255, 0, 0.7)';
		ctx.stroke();
		ctx.setLineDash([]);

		// Draw Skeleton
		drawSkeleton(ctx, landmarks);

		// 2. Seizure Detection
		updateHistory(leftWrist, rightWrist, leftElbow, rightElbow, mouth, nose);

		const tLW = calculateTortuosity(history.leftWrist);
		const tRW = calculateTortuosity(history.rightWrist);
		const tLE = calculateTortuosity(history.leftElbow);
		const tRE = calculateTortuosity(history.rightElbow);
		const bodyTortuosity = (tLW + tRW + tLE + tRE) / 4;

		const tM = calculateTortuosity(history.mouth);
		const tN = calculateTortuosity(history.nose);
		const faceTortuosity = ((tM + tN) / 2) * 1.2;

		const bodyMotion = getAverageSpeed(history.leftWrist) + getAverageSpeed(history.rightWrist);
		const faceMotion = (getAverageSpeed(history.mouth) + getAverageSpeed(history.nose)) * 2.0;

		let anchorVel = 0;
		if (lastAnchor) {
			anchorVel = Math.sqrt(Math.pow(anchor.x - lastAnchor.x, 2) + Math.pow(anchor.y - lastAnchor.y, 2));
		}
		lastAnchor = anchor;

		const isBodyShaking = bodyTortuosity > seizureTortuosityThresh && bodyMotion > 0.5;
		const isBodyStable = anchorVel < anchorStabilityThresh;

		const faceStabilityLimit = anchorStabilityThresh * 4.0; 
		const isFaceShaking = faceTortuosity > seizureTortuosityThresh && faceMotion > 0.2;
		const isFaceStable = anchorVel < faceStabilityLimit;

		const isSeizureDetected = (isBodyShaking && isBodyStable) || (isFaceShaking && isFaceStable);

		let isSeizure = false;
		if (isSeizureDetected) {
			if (!seizureStartTime) seizureStartTime = Date.now();
			else if (Date.now() - seizureStartTime > seizureDurationMs) isSeizure = true;
		} else {
			seizureStartTime = null;
		}

		// Status Update
		if (isFall) setStatus('ALERT', 'FALL');
		else if (isSeizure) setStatus('ALERT', 'SEIZURE');
		else setStatus('SAFE', null);
	}

	function updateHistory(lw, rw, le, re, m, n) {
		history.leftWrist.push(lw);
		history.rightWrist.push(rw);
		history.leftElbow.push(le);
		history.rightElbow.push(re);
		history.mouth.push(m);
		history.nose.push(n);

		Object.values(history).forEach(arr => {
			if (arr.length > HISTORY_SIZE) arr.shift();
		});
	}

	function drawSkeleton(ctx, landmarks) {
		const connections = [
			[11, 13], [13, 15], // Left Arm
			[12, 14], [14, 16], // Right Arm
			[11, 12], // Shoulders
			[0, 11], [0, 12] // Neck/Head hint
		];

		ctx.lineWidth = 2;
		ctx.strokeStyle = '#00FF00';

		connections.forEach(([i, j]) => {
			const p1 = landmarks[i];
			const p2 = landmarks[j];
			if (p1 && p2 && p1.visibility > 0.5 && p2.visibility > 0.5) {
				ctx.beginPath();
				ctx.moveTo(p1.x * ctx.canvas.width, p1.y * ctx.canvas.height);
				ctx.lineTo(p2.x * ctx.canvas.width, p2.y * ctx.canvas.height);
				ctx.stroke();
			}
		});

		ctx.fillStyle = '#FF0000';
		[0, 11, 12, 13, 14, 15, 16].forEach(index => {
			const p = landmarks[index];
			if (p && p.visibility > 0.5) {
				ctx.beginPath();
				ctx.arc(p.x * ctx.canvas.width, p.y * ctx.canvas.height, 4, 0, 2 * Math.PI);
				ctx.fill();
			}
		});
	}

	function calculateTortuosity(hist) {
		if (hist.length < 2) return 0;
		let totalPath = 0;
		for (let i = 1; i < hist.length; i++) {
			totalPath += Math.sqrt(Math.pow(hist[i].x - hist[i-1].x, 2) + Math.pow(hist[i].y - hist[i-1].y, 2));
		}
		const start = hist[0];
		const end = hist[hist.length - 1];
		const netDisp = Math.sqrt(Math.pow(end.x - start.x, 2) + Math.pow(end.y - start.y, 2));
		return totalPath / (netDisp + 0.05);
	}

	function getAverageSpeed(hist) {
		if (hist.length < 2) return 0;
		let totalPath = 0;
		for (let i = 1; i < hist.length; i++) {
			totalPath += Math.sqrt(Math.pow(hist[i].x - hist[i-1].x, 2) + Math.pow(hist[i].y - hist[i-1].y, 2));
		}
		return totalPath;
	}

	function setStatus(state, type) {
		if (state === 'ALERT') {
			statusText = `${type} DETECTED!`;
			statusColor = 'red';
			startBeep();
			updateEventLog(type);
		} else {
			statusText = "SAFE";
			statusColor = "green";
			stopBeep();
			clearEvent();
		}
	}

	function updateEventLog(type) {
		const now = Date.now();
		if (!activeEvent) {
			activeEvent = { type: type, start: now, lastLog: now };
			addLog(now, "ALERT", `${type} STARTED`);
		} else if (activeEvent.type !== type) {
			addLog(now, "INFO", `${activeEvent.type} ENDED`);
			activeEvent = { type: type, start: now, lastLog: now };
			addLog(now, "ALERT", `${type} STARTED`);
		} else {
			if (now - activeEvent.lastLog > 10000) {
				activeEvent.lastLog = now;
			}
		}
	}

	function clearEvent() {
		if (activeEvent) {
			const now = Date.now();
			addLog(now, "INFO", `${activeEvent.type} ENDED`);
			activeEvent = null;
		}
	}

	function addLog(id, type, val) {
		const timestamp = new Date().toLocaleTimeString();
		const logItem = {
			id: id,
			time: timestamp,
			type: type,
			intensity: val,
			image: null
		};
		logs = [logItem, ...logs].slice(0, 50);
	}

	function selectLog(log) { selectedLog = log; }
	function closeLog() { selectedLog = null; }
</script>

<div class="container">
	<header>
		<h1>DOCK_MONITOR</h1>
		<div class="status">
			<span class="indicator" class:active={isMonitoring} style="background-color: {isMonitoring ? statusColor : '#E0E0E0'}"></span>
			{statusText}
		</div>
	</header>

	<main>
		<section class="camera-feed">
			<div class="video-wrapper">
				<!-- svelte-ignore a11y_media_has_caption -->
				<video bind:this={videoSource} class="hidden-video" playsinline muted autoplay></video>
				<canvas bind:this={canvasRef} width="640" height="480"></canvas>
				<div class="overlay">
					<div class="metric">
						<label>STATUS</label>
						<span style="color: {statusColor}; font-weight: bold;">{statusText}</span>
					</div>
				</div>
			</div>
		</section>

		<section class="logs-panel">
			<div class="logs-header">
				<h2>EVENT_LOG</h2>
			</div>
			<div class="logs-list">
				{#each logs as log (log.id)}
					<button class="log-item" onclick={() => selectLog(log)}>
						<span class="time">{log.time}</span>
						<span class="type" style="color: {log.type === 'ALERT' ? 'red' : 'inherit'}">{log.type}</span>
						<span class="value">{log.intensity}</span>
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
				<div class="modal-info">
					<p>TIME: {selectedLog.time}</p>
					<p>TYPE: {selectedLog.type}</p>
					<p>MSG: {selectedLog.intensity}</p>
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
		background: #FFFFFF;
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

	.modal-info p {
		margin: 5px 0;
		font-size: 0.8rem;
		color: #555;
	}
</style>