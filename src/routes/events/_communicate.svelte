<script type="ts">
	import { page } from '$app/stores';
	import type { EventWithPeople } from '$lib/event';

	export let event: EventWithPeople;
	let linkCopied = false;
	let linkInput: HTMLInputElement | null = null;

	function copyLink() {
		if (linkInput) linkInput.select();
		document.execCommand('copy');
		linkCopied = true;
	}
</script>

<h5>Communiquer auprès de vos invités</h5>
<p>Vous pouvez envoyer le lien suivant à vos invités pour leur permettre de s'inscrire :</p>
<div class="input-group input-group-sm mb-3">
	<input
		bind:this={linkInput}
		type="text"
		class="form-control"
		value="http://{$page.host}#{event.public_code || '...'}"
	/>
	<button class="input-group-text" title="Copier" on:click={copyLink}>
		{#if !linkCopied}📋
		{:else}Copié !
		{/if}
	</button>
</div>
<p>
	<small>
		Attention, ne confondez pas! Envoyez bien le lien indiqué ci-dessus, et non l'adresse de la page
		actuelle à vos invités. La page actuelle est réservée aux administrateurs de l'évènement.
	</small>
</p>
