<script lang="ts" context="module">
	function captureImage(canvas: HTMLCanvasElement, player: HTMLVideoElement) {
		const ctx = canvas.getContext('2d');
		ctx.drawImage(player, 0, 0, canvas.width, canvas.height);
		return new Promise<Blob>((resolve) => {
			canvas.toBlob(resolve);
		});
	}

	function captureVideo(stream: MediaStream) {
		const mediaRecorder = new MediaRecorder(stream);
		const recordedChunks: Blob[] = [];

		mediaRecorder.addEventListener('dataavailable', function (e) {
			if (e.data.size > 0) {
				recordedChunks.push(e.data);
			}
		});

		mediaRecorder.start();

		return () =>
			new Promise<Blob>((resolve) => {
				mediaRecorder.addEventListener('stop', () => {
					resolve(new Blob(recordedChunks));
				});
				mediaRecorder.stop();
			});
	}
</script>

<script lang="ts">
	import MdClose from 'svelte-icons/md/MdClose.svelte';
	import MdRadioButtonChecked from 'svelte-icons/md/MdRadioButtonChecked.svelte';
	import MdRadioButtonUnchecked from 'svelte-icons/md/MdRadioButtonUnchecked.svelte';
	import MdSwitchCamera from 'svelte-icons/md/MdSwitchCamera.svelte';
	import { createEventDispatcher, onMount, tick } from 'svelte/internal';
	import Modal from './Modal.svelte';

	export let open = false;
	export let audio = false;
	export let video = false;
	export let width = 0;
	export let height = 0;
	export let mediaStream: MediaStream = undefined;
	export let player: HTMLVideoElement = undefined;
	export let canvas: HTMLCanvasElement = undefined;

	const dispatch = createEventDispatcher<{
		image: Blob;
		video: Blob;
	}>();

	let captureDown = false;
	let recording = false;
	let recordingTimeoutId: number;
	let getCapturedVideo: () => Promise<Blob>;

	function onClose() {
		open = false;
	}
	function onSwitch() {
		currentDevice = currentDevice >= devices.length - 1 ? 0 : currentDevice + 1;
		getMediaStream();
	}
	function onStart(event: MouseEvent | TouchEvent) {
		if (event.type === 'mousedown' && event['button'] !== 0) {
			return;
		}
		if (captureDown) {
			return;
		}
		captureDown = true;
		if (video) {
			recordingTimeoutId = setTimeout(() => {
				if (captureDown) {
					recording = true;
					getCapturedVideo = captureVideo(mediaStream);
				}
				recordingTimeoutId = undefined;
			}, 500) as any;
		}
	}
	function onEnd() {
		if (captureDown) {
			captureDown = false;
			if (recordingTimeoutId) {
				clearTimeout(recordingTimeoutId);
				recordingTimeoutId = undefined;
			}
			if (recording) {
				recording = false;
				onCaptureVideo();
			} else {
				onCaptureImage();
			}
		}
	}
	function onCaptureImage() {
		tick()
			.then(() => captureImage(canvas, player))
			.then((blob) => dispatch('image', blob));
	}

	function onCaptureVideo() {
		tick()
			.then(getCapturedVideo)
			.then((blob) => dispatch('video', blob));
	}

	onMount(() => {
		window.document.addEventListener('mouseup', onEnd);
		window.document.addEventListener('mouseleave', onEnd);
		window.document.addEventListener('touchend', onEnd);
		window.document.addEventListener('touchcancel', onEnd);

		return () => {
			window.document.removeEventListener('mouseup', onEnd);
			window.document.removeEventListener('mouseleave', onEnd);
			window.document.removeEventListener('touchend', onEnd);
			window.document.removeEventListener('touchcancel', onEnd);
			closeMediaStream();
		};
	});

	function closeMediaStream() {
		if (mediaStream) {
			mediaStream.getTracks().forEach((track) => track.stop());
			mediaStream = undefined;
			player.srcObject = undefined;
		}
	}
	function getDevices() {
		navigator.mediaDevices
			.enumerateDevices()
			.then((devices) => devices.filter((device) => device.kind === 'videoinput'))
			.then((d) => {
				devices = d;
				if (currentDevice >= devices.length) {
					currentDevice = 0;
				}
				getMediaStream();
			});
	}
	let openingMediaStream = false;
	function getMediaStream() {
		if (openingMediaStream) {
			return;
		}
		openingMediaStream = true;
		closeMediaStream();
		navigator.mediaDevices
			.getUserMedia({
				video: {
					deviceId: devices[currentDevice]?.deviceId
				},
				audio
			})
			.then((stream) => {
				let muted = false;
				let maxWidth = 0;
				let maxHeight = 0;
				stream.getVideoTracks().forEach((track) => {
					const settings = track.getSettings();
					maxWidth = Math.max(maxWidth, settings.width);
					maxHeight = Math.max(maxHeight, settings.height);
					if (track.muted) {
						muted = track.muted;
					}
				});
				if (muted) {
					onSwitch();
					getMediaStream();
				} else {
					videoWidth = maxWidth;
					videoHeight = maxHeight;
					mediaStream = stream;
					player.srcObject = stream;

					ratio =
						videoWidth > innerWidth || videoHeight > innerHeight
							? Math.min(innerWidth / videoWidth, innerHeight / videoHeight)
							: 1;
					width = videoWidth * ratio;
					height = videoHeight * ratio;
				}
			})
			.finally(() => {
				openingMediaStream = false;
			});
	}

	let innerWidth: number;
	let innerHeight: number;
	let videoWidth = 1;
	let videoHeight = 1;

	let devices: MediaDeviceInfo[] = [];
	let currentDevice = 0;
	$: if (open && !mediaStream && typeof navigator.mediaDevices !== 'undefined') {
		getDevices();
	}
	$: if (!open && mediaStream) {
		closeMediaStream();
	}

	$: ratio =
		videoWidth > innerWidth || videoHeight > innerHeight
			? Math.min(innerWidth / videoWidth, innerHeight / videoHeight)
			: 1;
	$: width = videoWidth * ratio;
	$: height = videoHeight * ratio;
</script>

<svelte:window bind:innerWidth bind:innerHeight />

<Modal bind:open>
	<canvas bind:this={canvas} {width} {height} />
	<video bind:this={player} autoplay {width} {height} />
	<button class="btn close" on:click={onClose}>
		<MdClose />
	</button>
	<button class="btn switch" on:click={onSwitch}>
		<MdSwitchCamera />
	</button>
	<button
		class="btn capture"
		class:active={captureDown}
		class:recording
		on:mousedown={onStart}
		on:touchstart={onStart}
	>
		<span class="checked"><MdRadioButtonChecked /></span>
		<span class="unchecked"><MdRadioButtonUnchecked /></span>
	</button>
</Modal>

<style>
	canvas {
		display: none;
	}
	video {
		display: block;
	}
	.btn {
		cursor: pointer;
		position: absolute;
		color: white;
		background-color: rgba(0, 0, 0, 0.3);
		border: none;
		border-radius: 100%;
		padding: 0;
		margin: 0;
	}
	.close {
		width: 2rem;
		height: 2rem;
		top: 0.5rem;
		right: 0.5rem;
	}
	.capture {
		width: 4rem;
		height: 4rem;
		bottom: 0.5rem;
		right: 50%;
		margin-right: -2rem;
	}
	.switch {
		width: 3rem;
		height: 3rem;
		right: 0.5rem;
		bottom: 0.5rem;
	}
	.capture .checked {
		display: none;
	}
	.capture:hover .checked {
		display: inherit;
	}
	.capture.active {
		color: #f10000;
	}
	.capture.recording {
		animation-name: color;
		animation-duration: 1s;
		animation-iteration-count: infinite;
	}
	.capture:hover .unchecked {
		display: none;
	}

	@keyframes color {
		0% {
			color: #f10000;
		}
		50% {
			color: #800;
		}
		100% {
			color: #f10000;
		}
	}
</style>
