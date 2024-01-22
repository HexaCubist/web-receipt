<script lang="ts">
	import Dropzone from 'svelte-file-dropzone';
	import { Printer, WebUSB, Align, Style, Image as EscImage, type ImageData } from 'escpos-buffer';
	import { fade, slide } from 'svelte/transition';
	import wrap from 'word-wrap';
	import Editor, { type lineDiff } from '$lib/Editor.svelte';
	import type Quill from 'quill';
	let loaded = false;
	let textToPrint = '';
	let lines: string[] = [];
	let canvas: HTMLCanvasElement;
	$: ctx = canvas?.getContext('2d');

	let showFiles = false;

	function handleFilesSelect(e) {
		const { acceptedFiles, _fileRejections } = e.detail;
		if (!ctx) return;
		const img = new Image();
		img.onload = () => {
			canvas.width = img.width;
			canvas.height = img.height;
			ctx?.drawImage(img, 0, 0, canvas.width, canvas.height);
			console.log('Loaded image');
			showFiles = true;
		};
		img.src = URL.createObjectURL(acceptedFiles[0]);
	}

	let printer: Printer | undefined = undefined;
	const connect = async () => {
		const device = await navigator.usb.requestDevice({
			filters: [
				// {
				// 	vendorId: 0x0456
				// }
			]
		});
		const connection = new WebUSB(device);
		printer = await Printer.CONNECT('POS-80', connection);
		await printer.setColumns(32);
		loaded = true;
	};
	let processing = false;
	const print = async () => {
		lines.push(textToPrint);
		if (!printer) {
			console.log(textToPrint);
			return;
		}
		try {
			await printer.withStyle(
				{
					// italic: true,
					bold: true,
					align: Align.Center
				},
				async () => {
					const wrappedLines = wrap(textToPrint.replaceAll(/[^\x00-\x7F]+/g, ' '), {
						width: 30,
						trim: true
					}).split('\n');
					for (const line of wrappedLines) {
						await printer!.writeln(line);
					}
				}
			);
			if (showFiles) {
				if (!ctx) return;
				let image_data_url = canvas.toDataURL('image/jpeg');
				let image_data = ctx.getImageData(0, 0, canvas.width, canvas.height);
				const image = new EscImage({
					data: image_data.data as any,
					width: image_data.width,
					height: image_data.height
				});
				await printer.draw(image);
			}
			await printer.feed(3);
			await printer.cutter();
			showFiles = false;
		} catch (error) {
			console.log("Wow, looks like the printer didn't work!", error);
			return;
		}
	};

	async function processDeltas(detail: lineDiff) {
		if (!printer) {
			console.log('Failed to print', detail);
			return;
		}
		console.log('Printing', detail);
		processing = true;
		for (const delta of detail) {
			// Each delta is a line, so print children with formatting then newline
			let alignment = Align.Left;
			switch (delta.attributes?.align) {
				case 'right':
					alignment = Align.Right;
					break;
				case 'center':
					alignment = Align.Center;
					break;
				default:
					alignment = Align.Left;
					break;
			}
			await printer.setAlignment(alignment);
			for (const op of delta.line.ops) {
				if (op.insert) {
					if (typeof op.insert === 'string') {
						await printer.withStyle(
							{
								italic: op.attributes?.italic as boolean | undefined,
								bold: op.attributes?.bold as boolean | undefined,
								underline: op.attributes?.underline as boolean | undefined
							},
							async () => {
								await printer?.write((op.insert as string).replaceAll(/[^\x00-\x7F]+/g, ' '));
							}
						);
					} else if (op.insert.image) {
						// Create canvas
						const canvas = document.createElement('canvas');
						const ctx = canvas.getContext('2d');
						if (!ctx) continue;

						// Create image
						await new Promise<void>((resolve) => {
							const img = new Image();
							img.onload = () => {
								canvas.width = img.width;
								canvas.height = img.height;
								ctx?.drawImage(img, 0, 0, canvas.width, canvas.height);
								resolve();
							};
							img.src = (op.insert as { image: string }).image;
						});
						// Get image data
						let image_data_url = canvas.toDataURL('image/jpeg');
						let image_data = ctx.getImageData(0, 0, canvas.width, canvas.height);

						// Print
						const image = new EscImage({
							data: image_data.data as any,
							width: image_data.width,
							height: image_data.height
						});
						await printer.draw(image);
					}
				}
			}
			await printer.writeln('');
		}
		await printer.feed(3);
		await printer.cutter();
		processing = false;
	}
</script>

<svelte:window
	on:keydown={(e) => {
		if ((e.key === 'b' || e.key === 'PageDown') && loaded) {
			print();
		}
	}}
/>

<main class="receipt receipt-after">
	<h1>Receipt Printer</h1>
	<p class="balance">
		Welcome to a web-based receipt printer! Press the button below to connect, and then enter what
		you'd like printed
	</p>
	<div class="pt-3 flex flex-wrap gap-3 justify-center">
		<button
			on:click={() => {
				connect();
			}}>Connect to printer</button
		>
		<button
			on:click={() => {
				loaded = true;
			}}>Skip</button
		>
	</div>
	{#if processing}
		<div
			class="absolute inset-0 bg-white bg-opacity-80 backdrop-blur flex justify-center align-middle items-center"
			transition:fade
		>
			<div class="text-6xl text-center animate-bounce">❤️‍🔥</div>
		</div>
	{/if}
</main>

{#if loaded}
	<div class="receipt receipt-after receipt-before" in:slide>
		<Editor
			on:print={({ detail }) => {
				processDeltas(detail);
			}}
		></Editor>
	</div>
{/if}

{#each lines.reverse().slice(0, Math.min(lines.length, 3)) as line (line)}
	<div class="receipt receipt-after receipt-before" in:slide>
		<q class="text-center block font-mono text-lg">{line}</q>
	</div>
{/each}

<style lang="postcss">
	:global(body) {
		@apply bg-cover bg-fixed;
		background-image: linear-gradient(135deg, #ccffff 0%, #ffffcc 50%, #ffccff 100%);
	}
	h1 {
		@apply mb-4 text-center text-4xl font-bold;
	}
	.balance {
		/* Balance break */
		@apply text-center;
		text-wrap: balance;
	}
	.receipt {
		@apply m-4 mx-auto w-full max-w-prose rounded bg-white px-7 py-6 drop-shadow-md;
		--zag-size: 0.8rem;
		&.receipt-after {
			@apply rounded-b-none;
			&:after {
				@apply absolute left-0 top-full block w-full;
				@apply bg-left-bottom bg-repeat-x;
				height: var(--zag-size);
				background: linear-gradient(-45deg, transparent var(--zag-size), #fff var(--zag-size)),
					linear-gradient(45deg, transparent var(--zag-size), #fff 0);
				background-size: var(--zag-size) var(--zag-size);
				content: ' ';
			}
		}
		&.receipt-before {
			@apply mt-10 rounded-t-none;
			&:before {
				@apply absolute bottom-full left-0 block w-full;
				@apply bg-left-bottom bg-repeat-x;
				height: var(--zag-size);
				background: linear-gradient(-45deg, #fff calc(var(--zag-size) / 2), transparent 0),
					linear-gradient(45deg, #fff calc(var(--zag-size) / 2), transparent 0);
				background-size: var(--zag-size) var(--zag-size);
				content: ' ';
			}
		}
	}
	main {
		@apply mt-16;
	}
	button,
	.button {
		@apply rounded bg-black bg-opacity-100 px-4 py-2 font-bold text-white transition;
		&:hover {
			@apply bg-opacity-70;
		}
	}
</style>
