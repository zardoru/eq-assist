<script lang="ts">
	import { browser } from '$app/environment';

	let active = false;
	let buffSource: AudioBufferSourceNode;
	let gainNode: GainNode;
	let pannerNode: StereoPannerNode;
    let filterNode: BiquadFilterNode;
	let gain = 0.25;
	let pan = 0;

	if (browser) {
		window.AudioContext = window.AudioContext || window.webkitAudioContext;
		const audioContext = new AudioContext();

		/* https://noisehack.com/generate-noise-web-audio-api/ */
		function makePinkNoiseBuffer(audioContext: AudioContext): AudioBuffer {
			const buffSize = 2 * audioContext.sampleRate;
			const noise = audioContext.createBuffer(1, buffSize, audioContext.sampleRate);
			const noise_data = noise.getChannelData(0);

			let b0 = 0.5362;
			let b = [0, 0, 0, 0, 0, 0, 0];
			let filter = [
				[0.99886, 0.0555179],
				[0.99332, 0.0750759],
				[0.969, 0.153852],
				[0.8665, 0.3104856],
				[0.55, 0.5329522],
				[-0.7616, -0.016898],
				[0, 0.115926]
			];

			for (let i = 0; i < buffSize; i++) {
				var white = Math.random() * 2 - 1;
				for (let j = 0; j < b.length; j++) {
					b[j] = filter[j][0] * b[j] + white * filter[j][1];
				}

				noise_data[i] = b.reduce((acc, cur) => acc + cur, white * b0) * 0.11;
			}

			return noise;
		}

		let pink = makePinkNoiseBuffer(audioContext);
		buffSource = audioContext.createBufferSource();

		buffSource.buffer = pink;
		buffSource.loop = true;

        filterNode = audioContext.createBiquadFilter();
        filterNode.type = "bandpass";
        buffSource.connect(filterNode);

        pannerNode = audioContext.createStereoPanner();
		filterNode.connect(pannerNode);
        

		gainNode = audioContext.createGain();
		pannerNode.connect(gainNode);

		gainNode.connect(audioContext.destination);
	}

    let qval = 0;
    let frequency = 4000;

	let audioStarted = false;
	let oldGain = gain;
	const maxGain = 1;
	const minGain = 0;
	function toggleAudio() {
		if (!active) {
			if (!audioStarted) {
				audioStarted = true;
				buffSource.start(0);
			}

			gain = oldGain;
		} else {
			oldGain = gain;
			gain = 0;
		}
	}

	function buttonClass(_: any) {
		let base = 'transition-color  border-0 p-2 ';
		if (active) {
			return base + 'bg-red-500';
		} else {
			return base + 'bg-green-500';
		}
	}

	$: activateClass = buttonClass(active);
	$: if (gainNode) gainNode.gain.value = gain * (maxGain - minGain) + minGain;

	$: if (pannerNode) pannerNode.pan.value = pan;

    $: if (filterNode) { 
        filterNode.Q.value = qval;
        filterNode.frequency.value = frequency;
    }

    $: active = gain > 0 && audioStarted;
</script>

<main class="flex flex-col align-middle justify-between p-6 gap-8">
    <h1 class="drop-shadow-xl text-3xl leading-6 text-center">zardoru's unremarkable pink noise tool</h1>
	<div class="flex flex-col">
		<div class="flex flex-col grow">
			<div class="flex gap-2 align-middle mx-auto bg-slate-800 rounded shadow-md shadow-black">
				<div class="basis-1 self-center p-4">
					{#if active && audioStarted}
						<span class="text-red-500 animate-pulse font-bold leading-6">ACTIVE</span>
					{:else}
						<span class="text-slate-200">Inactive</span>
					{/if}
				</div>

				<button on:click={toggleAudio} class={activateClass}>
					{#if active} Deactivate {:else} Activate {/if}
				</button>
			</div>
			<div class="flex justify-center rounded grow-0 p-4 gap-4">
				<div class="flex flex-row shadow-md shadow-black">
					<label for="_gain" class=" bg-slate-800 rounded-l p-4">Gain </label>
					<div class="bg-slate-700 rounded-r self-center p-4">
						<input
							name="_gain"
							type="range"
							min="0"
							max="1"
							bind:value={gain}
							step="any"
							class="self-center"
							on:dblclick={() => (gain = 0.25)}
						/>
					</div>
				</div>

				<div class="flex flex-row shadow-md shadow-black">
					<label for="_pan" class=" bg-slate-800 rounded-l p-4">Pan</label>
					<div class="bg-slate-700 rounded-r self-center p-4">
						<input
							name="_pan"
							type="range"
							min="-1"
							max="1"
							bind:value={pan}
							step="any"
							class="self-center"
							on:dblclick={() => (pan = 0)}
						/>
					</div>
				</div>
			</div>
		</div>

		<div class="flex flex-col mx-20">
			<div class="flex flex-col shadow-md shadow-black bg-slate-800">
				<span class="p-4 self-center rounded-t w-full"> Band pass </span>
				<div class="grid grid-cols-[20%_1fr]">
					<label for="_q" class="p-2 grow text-right">Q value</label>
					<div class="flex bg-slate-700 w-full h-full">
						<input
							name="_q"
							type="range"
							min="0"
							max="10"
							bind:value={qval}
							step="any"
							class="my-2 w-[70%] mx-auto self-center"
							on:dblclick={() => (pan = 0)}
						/>
					</div>
                    <label for="_freq" class="p-2 grow text-right">Frequency</label>
					<div class="flex bg-slate-700 w-full h-full">
						<input
							name="_freq"
							type="range"
							min="10"
							max="18000"
							bind:value={frequency}
							step="any"
							class="my-2 w-[70%] mx-auto self-center"
							on:dblclick={() => (pan = 0)}
						/>
					</div>
				</div>
			</div>
		</div>
	</div>
</main>
