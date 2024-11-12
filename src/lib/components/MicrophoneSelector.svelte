<script lang="ts">
	import { onMount } from 'svelte';

	const { onselect }: { onselect: (deviceId: string) => void } = $props();

	async function getMicrophones() {
		const devices = await navigator.mediaDevices.enumerateDevices();
		const microphones = devices.filter((device) => device.kind === 'audioinput');
		return microphones.map((microphone) => ({
			id: microphone.deviceId,
			label: microphone.label
		}));
	}

	let devices = $state<{ label: string; id: string }[]>([]);

	onMount(() => {
		getMicrophones().then((microphones) => {
			devices = microphones;
		});
	});
</script>

<select
	class="rounded border px-4 py-2 pr-8"
	oninput={(e) => onselect((e.target as HTMLSelectElement).value)}
>
	{#each devices as device (device.id)}
		<option value={device.id}>{device.label}</option>
	{/each}
</select>
