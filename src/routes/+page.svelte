<script lang="ts">
	import { FFmpeg } from '@ffmpeg/ffmpeg';
	import { filesize } from 'filesize';
	import { onMount } from 'svelte';
	import { FileDrop } from 'svelte-droplet';
	import { tweened } from 'svelte/motion';

	import { UploadIcon, VideoFileIcon } from '$lib/components/icons';
	import * as Accordion from '$lib/components/ui/accordion';
	import Button from '$lib/components/ui/button/button.svelte';
	import Label from '$lib/components/ui/label/label.svelte';
	import Progress from '$lib/components/ui/progress/progress.svelte';
	import Separator from '$lib/components/ui/separator/separator.svelte';

	import Input from '$lib/components/ui/input/input.svelte';
	import '../app.postcss';
	import LabelledSwitch from '$lib/components/ui/LabelledSwitch.svelte';
	import { quartOut } from 'svelte/easing';

	type FFmpegState = 'loading' | 'ready' | 'compressing' | 'done' | 'error';

	const acceptedMimes = ['image/*', 'video/*', 'audio/*'];

	let progress = $state(
		tweened(0, {
			easing: quartOut
		})
	);
	let uploadedFile = $state<File>();
	let mb_limit = $state(10);

	const presets = [10, 25, 50, 100, 500];

	let options = $state({
		mute: false,
		resize: true,
		aggressive: false
	});

	function handleFiles(files: File[]) {
		if (files.length > 0) uploadedFile = files[0];
	}

	let ffmpegState: FFmpegState = $state<FFmpegState>('loading');
	let ffmpeg: FFmpeg;
	let ffmpegMessage = $state('');

	const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm';

	function file_too_small() {
		return uploadedFile && uploadedFile.size < mb_limit * 1048576 && !options.aggressive;
	}

	async function loadFFmpeg() {
		ffmpeg = new FFmpeg();

		ffmpeg.on('log', ({ message }) => {
			ffmpegMessage = message === 'Aborted()' ? 'Done!' : message;
			console.log(ffmpegMessage);
		});
		ffmpeg.on('progress', (ev) => {
			console.log(ev.progress);
			$progress = ev.progress * 100;
			if ($progress >= 100) ffmpegMessage = 'Compression complete!';
			console.info('Compression complete!');
		});
		await ffmpeg.load({
			coreURL: `${baseURL}/ffmpeg-core.js`,
			wasmURL: `${baseURL}/ffmpeg-core.wasm`
		});
		ffmpegState = 'ready';
		console.log('Ready!');
	}

	async function readFile(file: File): Promise<Uint8Array> {
		return new Promise((resolve) => {
			const fileReader = new FileReader();
			fileReader.onload = () => {
				const { result } = fileReader;
				if (result instanceof ArrayBuffer) {
					resolve(new Uint8Array(result));
				}
			};

			fileReader.onerror = () => {
				ffmpegState = 'error';
				ffmpegMessage = 'Could not read file';
			};

			fileReader.readAsArrayBuffer(file);
		});
	}

	async function compress() {
		if (!ffmpeg || !uploadedFile) return;
		ffmpegState = 'compressing';
		console.log('Compressing video...');
		const fileData = await readFile(uploadedFile);
		await ffmpeg.writeFile(uploadedFile.name, fileData);
		// THIS IS THE IMPORTANT PART, THIS IS WHERE IT'S DEFINE HOW WE PROCESS THE VIDEO
		let params: string[] = [];
		if (options.aggressive) {
			params.push(
				'-vf',
				'scale=ceil(iw/4):ceil(ih/4)',
				'-b:v',
				'32k',
				'-b:a',
				'32k',
				'-c:a',
				'aac'
			);
		} else {
			params.push('-vcodec', 'libx264', '-crf', '20');
		}

		if (options.mute) {
			params.push('-an');
		}
		const status = await ffmpeg.exec(['-i', uploadedFile.name, ...params, 'output.mp4']);
		if (status != 0) {
			console.error(`Compression failed with status code ${status}`);
			ffmpegMessage = 'Compression failed! Check console for details.';
			ffmpegState = 'error';
			return;
		}
		const data = (await ffmpeg.readFile('output.mp4')) as Uint8Array;
		ffmpegState = 'done';
		console.log('Done!');

		const a = document.createElement('a');
		a.href = URL.createObjectURL(new Blob([data.buffer], { type: 'video/mp4' }));
		a.download = 'video.mp4';

		setTimeout(() => a.click(), 500);
	}

	onMount(() => loadFFmpeg());
</script>

{#snippet preset_button(size: number)}
	<Button
		class="bg-amber-200 shadow-lg hover:bg-amber-300 
        transition-all duration-200 hover:scale-[1.05]"
		onclick={() => (mb_limit = size)}
		disabled={(ffmpegState !== 'loading' && ffmpegState !== 'ready') ||
			options.aggressive ||
			(uploadedFile && uploadedFile.size < size * 1048576)}
		size="sm">{`${size}MB`}</Button
	>
{/snippet}

<div class="flex flex-col items-center justify-center p-8 w-full md:w-1/2 xl:w-1/3 mx-auto">
	<h1>ZipMyVid</h1>
	<p class="md:text-xl my-2">Compress your videos easily.</p>
	<FileDrop {handleFiles} let:droppable max={1} {acceptedMimes}>
		<div class="zone" class:droppable class:uploaded={uploadedFile}>
			{#if uploadedFile}
				<VideoFileIcon />
				<p>{uploadedFile.name}</p>
				<small>{filesize(uploadedFile.size, { standard: 'jedec' })}</small>
			{:else}
				<UploadIcon />
				<p class="text-sm">Upload/drop your video here</p>
			{/if}
		</div>
	</FileDrop>
	<Separator />
	<div class="flex flex-col items-start my-4 gap-2 w-full">
		<p>Options</p>
		<div class="flex items-center gap-2">
			<Label for="filesize-limit">File size limit (in MB)</Label>
			<Input
				id="filesize-limit"
				class="w-24"
				min="2"
				max="500"
				disabled={(ffmpegState !== 'loading' && ffmpegState !== 'ready') || options.aggressive}
				bind:value={mb_limit}
				type="number"
			/>
		</div>
		{#if file_too_small()}
			<small class="text-red-500">Size limit is larger than the input file's size.</small>
		{/if}
		<small>Presets</small>
		<div class="w-full gap-2 grid items-center justify-evenly grid-cols-2 md:grid-cols-3">
			{#each presets as p}
				{@render preset_button(p)}
			{/each}
		</div>
	</div>
	<Separator />
	<div class="flex flex-col gap-2 w-full">
		<Accordion.Root>
			<Accordion.Item value="options-item">
				<Accordion.Trigger>Advanced options</Accordion.Trigger>
				<Accordion.Content>
					<LabelledSwitch
						id="mute-output"
						label="Mute"
						description="Removes audio from the output video."
						bind:checked={options.mute}
						disabled={ffmpegState !== 'loading' && ffmpegState !== 'ready'}
					/>
					<LabelledSwitch
						id="enable-scaling"
						label="Resize"
						description="Allows the output video to be resized."
						bind:checked={options.resize}
						disabled={ffmpegState !== 'loading' && ffmpegState !== 'ready'}
					/>
					<LabelledSwitch
						id="aggressive-mode"
						label="Aggresive"
						description="Compresses the video to extreme levels, use with caution."
						bind:checked={options.aggressive}
						disabled={ffmpegState !== 'loading' && ffmpegState !== 'ready'}
					/>
				</Accordion.Content>
			</Accordion.Item>
		</Accordion.Root>
		<Button
			class="shadow-lg"
			onclick={async () => {
				mb_limit = Math.min(Math.max(mb_limit, 2), 500);
				await compress();
			}}
			disabled={!uploadedFile ||
				(ffmpegState !== 'loading' && ffmpegState !== 'ready') ||
				file_too_small()}
			size="lg">Compress it!</Button
		>
		{#if ffmpegState === 'compressing' || ffmpegState === 'done' || ffmpegState === 'error'}
			<div class="flex flex-col gap-2">
				<Progress value={$progress} />
				<p class="leading-none">Compressing ({Math.round($progress)}%)</p>
				<small class:text-red-500={ffmpegState === 'error'}>{ffmpegMessage}</small>
			</div>
		{/if}
	</div>
	<Separator class="my-2" />
	<footer class="flex flex-col items-center text-sm text-muted-foreground w-full">
		<p>
			Developed by <a class="underline text-[#9a70ff]" href="https://roaming97.com" target="_blank"
				>roaming97</a
			>
		</p>
		<!--
            <p class="text-xs">
                Did you find this tool useful? Consider buying me a <a
                    class="underline text-[#ff9431]"
                    href="https://croizzant.com/roaming97"
                    target="_blank">croissant</a
                > 🥐!
            </p>
        -->
	</footer>
</div>

<style lang="postcss">
	h1 {
		@apply p-4 text-5xl md:text-6xl lg:text-8xl font-black text-transparent 
        bg-gradient-to-r from-lime-200 to-yellow-400 bg-clip-text tracking-tighter;
	}
	.zone {
		@apply my-4 flex flex-col items-center justify-center rounded-lg 
        bg-yellow-950/20 border-2 border-dashed border-amber-400 text-muted-foreground
        md:w-96 h-64 shadow-lg transition-all duration-200 ease-in-out 
        hover:brightness-200 hover:shadow-xl;
	}
	.uploaded {
		@apply text-white;
	}
	.droppable {
		@apply brightness-200 shadow-xl -translate-y-1;
	}
</style>
