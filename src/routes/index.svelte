<script lang="ts">
	import { Capture } from 'svelte-capture';

	let video = true;
	function onVideoChange(e: Event) {
		video = (e.target as HTMLInputElement).checked;
	}
	let audio = false;
	function onAudioChange(e: Event) {
		audio = (e.target as HTMLInputElement).checked;
	}
	let capture = false;
	function onClick() {
		capture = true;
	}

	let imageUrl: string;
	function onImageCapture(event: CustomEvent<Blob>) {
		imageUrl = URL.createObjectURL(event.detail);
		capture = false;
	}
	let videoUrl: string;
	function onVideoCapture(event: CustomEvent<Blob>) {
		videoUrl = URL.createObjectURL(event.detail);
		capture = false;
	}
</script>

<div class="controls">
	<label for="video">Use Video</label>
	<input id="video" type="checkbox" checked={video} on:change={onVideoChange} />
	<label for="audio">Use Audio</label>
	<input id="audio" type="checkbox" checked={audio} on:change={onAudioChange} />
	<button on:click={onClick}>Capture</button>
</div>

{#if imageUrl}
	<img src={imageUrl} />
{/if}
{#if videoUrl}
	<video src={videoUrl} controls />
{/if}

<Capture bind:open={capture} {video} {audio} on:image={onImageCapture} on:video={onVideoCapture} />

<style>
	img,
	video {
		display: block;
		max-width: 100%;
		margin: 0 auto;
	}

	.controls {
		text-align: center;
		border-bottom: 1px solid #888;
	}
</style>
