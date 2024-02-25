<script context="module" lang="ts">
	import { default as Delta, type AttributeMap } from 'quill-delta';
	export { Delta, AttributeMap };

	export type lineDiff = {
		line: Delta;
		attributes: AttributeMap;
		index: number;
	}[];
</script>

<script lang="ts">
	import { browser } from '$app/environment';
	import type Quill from 'quill';
	import 'quill/dist/quill.snow.css';
	import { createEventDispatcher, onMount } from 'svelte';
	let edEl: HTMLDivElement;
	let quill: Quill;

	// https://stackoverflow.com/a/57036704
	// Store accumulated changes
	var change = new Delta();

	// Save periodically
	setInterval(function () {
		if (change.length() > 0) {
			console.log('Saving changes', change);
			// Save the entire updated text to localStorage
			const data = JSON.stringify(quill.getContents());
			localStorage.setItem('storedText', data);
			change = new Delta();
		}
	}, 5 * 1000);

	const dispatch = createEventDispatcher<{
		print: lineDiff;
	}>();

	$: if (browser)
		onMount(async () => {
			const Quill = (await import('quill')).default;
			// @ts-ignore
			window.Quill = Quill;
			// @ts-ignore
			const ImageResize = (await import('quill-image-resize-module')).default;
			Quill.register('modules/imageResize', ImageResize);
			quill = new Quill(edEl, {
				theme: 'snow',
				placeholder: 'Compose an epic...',

				modules: {
					toolbar: [
						['bold', 'italic', 'underline'],
						[{ align: ['', 'center', 'right'] }],
						['image']
					],
					imageResize: {
						modules: ['Resize']
					}
				}
			});
			// Load the stored text
			const storedText = localStorage.getItem('storedText');
			if (storedText) {
				quill.setContents(JSON.parse(storedText));
			}
			quill.on('text-change', function (delta) {
				change = change.compose(delta);
			});
		});
	const print = () => {
		let lines: lineDiff = [];
		const delta = quill.getContents();
		delta.eachLine((line, attributes, index) => {
			// Parse each line, and add it to the lines array
			lines.push({ line, attributes, index });
		});
		dispatch('print', lines);
	};
</script>

<div class="font-mono">
	<div id="editor" bind:this={edEl}></div>
</div>
<button
	on:click={() => {
		print();
	}}>Print</button
>

<style lang="postcss">
	:global(.ql-editor) {
		@apply font-mono;
	}
	button {
		@apply mt-2 rounded bg-black px-4 py-2 text-white shadow;
	}
</style>
