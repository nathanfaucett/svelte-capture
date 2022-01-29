<script lang="ts">
	import { Capture } from 'svelte-capture';

	let capture = false;
	function onClick() {
		console.log();
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

<button on:click={onClick}>Capture</button>

{#if imageUrl}
	<img src={imageUrl} />
{/if}
{#if videoUrl}
	<video src={videoUrl} controls />
{/if}

<Capture bind:open={capture} audio video on:image={onImageCapture} on:video={onVideoCapture} />

<style>
	img,
	video {
		display: block;
		max-width: 100%;
		margin: 0 auto;
	}
</style>
