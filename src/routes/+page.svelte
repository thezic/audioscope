<script lang="ts">
	import MicrophoneSelector from '$lib/components/MicrophoneSelector.svelte';
	import { onMount } from 'svelte';

	let audioContext: AudioContext;
	let analyzer: AnalyserNode;

	onMount(() => {
		audioContext = new AudioContext();
		analyzer = audioContext.createAnalyser();
		analyzer.fftSize = 2048;
	});

	let isRunning = $state(false);
	let canvas = $state<HTMLCanvasElement>();
	let canvasCtx = $derived(canvas?.getContext('2d'));

	// let micSource = $state<MediaStreamAudioSourceNode>();
	let micSource: MediaStreamAudioSourceNode | undefined;
	let microphoneId = $state<string>();

	async function start() {
		const stream = await navigator.mediaDevices.getUserMedia({
			audio: microphoneId ? { deviceId: { exact: microphoneId } } : true,
			video: false
		});
		micSource = audioContext.createMediaStreamSource(stream);
		micSource.connect(analyzer);
		audioContext.resume();

		isRunning = true;

		console.log('start', 'stream', stream, 'micSource', micSource, 'analyzer', analyzer);
	}

	function stop() {
		console.log('stop');
		if (micSource) {
			console.log('stop tracks');
			micSource.disconnect();
			micSource.mediaStream.getTracks().forEach((track) => track.stop());
			micSource = undefined;
			audioContext.suspend();
		}

		isRunning = false;
	}

	let lastTimestamp = 0;
	function render(timestamp: number) {
		if (!isRunning) return;

		requestAnimationFrame(render);

		if (!canvasCtx || !canvas) return;

		if (timestamp - lastTimestamp < 100) return;
		lastTimestamp = timestamp;

		const bufferLength = analyzer.frequencyBinCount;
		const dataArray = new Uint8Array(bufferLength);
		analyzer.getByteFrequencyData(dataArray);

		console.log(dataArray);

		canvasCtx.clearRect(0, 0, canvas.width, canvas.height);

		const barWidth = (canvas.width / bufferLength) * 2.5;
		let x = 0;

		for (let i = 0; i < bufferLength; i++) {
			const barHeight = dataArray[i];

			canvasCtx.fillStyle = 'rgb(' + (barHeight + 100) + ',50,50)';
			canvasCtx.fillRect(x, canvas.height - barHeight / 2, barWidth, barHeight / 2);

			x += barWidth + 1;
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
	<canvas bind:this={canvas} width="800" height="200"></canvas>
	<div>
		<button class="rounded border px-4 py-2" onclick={() => void start()}>Start</button>
		<button class="rounded border px-4 py-2" onclick={() => stop()}>Stop</button>
		<MicrophoneSelector onselect={onSelectMicrophone} />
	</div>
</div>
