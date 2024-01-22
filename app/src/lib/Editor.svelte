<script context="module" lang="ts">
	import type { default as Delta, AttributeMap } from 'quill-delta';
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

	const dispatch = createEventDispatcher<{
		print: lineDiff;
	}>();

	$: if (browser)
		onMount(async () => {
			const Quill = (await import('quill')).default;
			quill = new Quill(edEl, {
				theme: 'snow',
				placeholder: 'Compose an epic...',

				modules: {
					toolbar: [['bold', 'italic', 'underline'], [{ align: [] }], ['image']]
				}
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
