<script lang="ts">
	import CodeFound from '../_CodeFound.svelte';
	import { Alert } from 'sveltestrap';
	import { BarcodeFormat, DecodeHintType, NotFoundException } from '@zxing/library';
	import { BrowserMultiFormatReader } from '@zxing/browser';
	const pdfjs_promise = import('pdfjs-dist');

	export let codeFound: string | undefined = undefined;
	let error = '';
	let processing = false;
	let fileElement: HTMLInputElement | undefined = undefined;
	let canvasElement: HTMLCanvasElement | undefined = undefined;
	let imageUrl: string | null = null;

	const codeReader = new BrowserMultiFormatReader(
		new Map([
			[DecodeHintType.POSSIBLE_FORMATS, [BarcodeFormat.DATA_MATRIX, BarcodeFormat.QR_CODE]],
			[DecodeHintType.TRY_HARDER, [true]]
		])
	);

	async function renderPdf(canvasElement: HTMLCanvasElement, buffer: ArrayBuffer) {
		const canvasContext = canvasElement.getContext('2d');
		if (!canvasContext) throw new Error('Impossible de créer un canvas pour lire le PDF.');
		const pdfjs = await pdfjs_promise;
		pdfjs.GlobalWorkerOptions.workerSrc = await import('pdfjs-dist/build/pdf.worker.entry');
		const doc = await pdfjs.getDocument(new Uint8Array(buffer)).promise;
		const page = await doc.getPage(1);
		const viewport = page.getViewport({ scale: 2.5 });
		canvasElement.width = viewport.width;
		canvasElement.height = viewport.height;
		await page.render({ canvasContext, viewport }).promise;
	}

	async function renderImage(canvasElement: HTMLCanvasElement, buffer: ArrayBuffer) {
		const blob = new Blob([buffer], { type: 'image' });
		imageUrl = URL.createObjectURL(blob);
		const img = new Image();
		img.src = imageUrl;
		await new Promise((accept, reject) => {
			img.onload = accept;
			img.onerror = (_) => reject(new Error("Le fichier n'est pas une image valide."));
		});
		canvasElement.width = img.width;
		canvasElement.height = img.height;
		canvasElement.getContext('2d')?.drawImage(img, 0, 0);
	}

	async function checkFile() {
		try {
			processing = true;
			codeFound = undefined;
			error = '';
			const file = fileElement?.files?.item(0);
			if (!file || !canvasElement) throw new Error('Aucun fichier');
			canvasElement.height = 0;
			const buffer = await file.arrayBuffer();
			const render = file.name.endsWith('.pdf') ? renderPdf : renderImage;
			await render(canvasElement, buffer);
			const result = await codeReader.decodeFromCanvas(canvasElement);
			codeFound = result.getText();
		} catch (e) {
			console.log(e);
			if (e instanceof NotFoundException)
				error =
					'Aucun code QR ou 2D-DOC détecté dans le document. ' +
					'Essayez de recadrer le document pour que le code apparaisse clairement et dans une taille raisonnable.' +
					'Vous pouvez par exemple prendre une capture d’écran qui contient uniquement le QR code';
			else error = `${e?.message || e}`;
			codeFound = undefined;
		} finally {
			processing = false;
		}
	}
</script>
<svelte:head>
	<title>Sanipasse - Importer un passe sanitaire depuis un fichier</title>
</svelte:head>

<div class="card">
	<canvas class="card-img-top" bind:this={canvasElement} width="0" height="0">Pas de canvas</canvas>
	{#if processing}
		<div class="card-body text-center">
			<span class="spinner-border text-info" />
			<p>Chargement du document...</p>
		</div>
	{/if}
	<div class="card-body">
		<h5 class="card-title">Importer un certificat de test ou de vaccination</h5>
		<p class="card-text">
			Après votre test ou votre vaccination, un certificat numérique vous a été communiqué. Vous
			pouvez l'importer dans Sanipasse ci-dessous.
		</p>
	</div>
	<form class="card-body" on:submit={checkFile}>
		<label for="formFile" class="form-label">Certificat</label>
		<input
			class="form-control"
			type="file"
			bind:this={fileElement}
			on:change={checkFile}
			accept="image/*, .pdf"
		/>
	</form>
	{#if error}
		<div class="card-body">
			<Alert color="danger">
				<h5>Erreur</h5>
				<code>{error}</code>
			</Alert>
		</div>
	{/if}
</div>

<CodeFound bind:codeFound />

<style>
	canvas {
		max-height: 30vh;
		object-fit: contain;
		background-color: var(--bs-light);
	}
</style>
