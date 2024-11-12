<script lang="ts">
	import MicrophoneSelector from '$lib/components/MicrophoneSelector.svelte';
	import { onMount } from 'svelte';

	let audioContext = $state<AudioContext>();
	let analyzer = $state<AnalyserNode>();
	const fftSize = 2048;

	onMount(() => {
		audioContext = new AudioContext();
		analyzer = audioContext.createAnalyser();
		analyzer.fftSize = fftSize;
	});

	let isRunning = $state(false);
	let canvas = $state<HTMLCanvasElement>();
	let canvasCtx = $derived(canvas?.getContext('2d'));

	let micSource = $state<MediaStreamAudioSourceNode>();
	let microphoneId = $state<string>();

	async function start() {
		if (!audioContext || !analyzer) return;
		const stream = await navigator.mediaDevices.getUserMedia({
			audio: microphoneId ? { deviceId: { exact: microphoneId } } : true,
			video: false
		});
		micSource = audioContext.createMediaStreamSource(stream);
		micSource.connect(analyzer);
		audioContext.resume();

		isRunning = true;
	}

	function stop() {
		if (!audioContext || !analyzer) return;
		if (micSource) {
			micSource.disconnect();
			micSource.mediaStream.getTracks().forEach((track) => track.stop());
			micSource = undefined;
			audioContext?.suspend();
		}

		isRunning = false;
	}

	let lastTimestamp = 0;
	let max = 0;
	function render(timestamp: number) {
		if (!audioContext || !analyzer) return;
		if (!isRunning) return;
		if (!canvasCtx || !canvas) return;

		requestAnimationFrame(render);

		if (canvas.height != analyzer.frequencyBinCount) {
			canvas.height = analyzer.frequencyBinCount;
		}

		if (timestamp - lastTimestamp < 10) return;
		lastTimestamp = timestamp;

		const bufferLength = analyzer.frequencyBinCount;
		const dataArray = new Uint8Array(bufferLength);
		analyzer.getByteFrequencyData(dataArray);

		canvasCtx.drawImage(canvas, 1, 0);

		for (let y = 0; y < bufferLength; y++) {
			const intensity = dataArray[y];
			canvasCtx.fillStyle = `rgb(${intensity} ${intensity} ${intensity})`;
			canvasCtx.fillRect(0, y, 1, 1);
		}
	}

	$effect(() => {
		if (isRunning) {
			requestAnimationFrame(render);
		}
	});

	function onSelectMicrophone(deviceId: string) {
		microphoneId = deviceId;
		if (isRunning) {
			stop();
			start();
		}
	}
</script>

<div>
	<canvas bind:this={canvas} width="800" height={fftSize / 2}></canvas>
	<div>
		<button class="rounded border px-4 py-2" onclick={() => void start()}>Start</button>
		<button class="rounded border px-4 py-2" onclick={() => stop()}>Stop</button>
		<MicrophoneSelector onselect={onSelectMicrophone} />
	</div>
</div>
