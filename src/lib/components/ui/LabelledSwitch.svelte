<script lang="ts">
	import Switch from '$lib/components/ui/switch/switch.svelte';
	import Label from '$lib/components/ui/label/label.svelte';

	export let id = 'switch-id';
	export let label = 'Switch label';
	export let description = '';
	export let checked = false;
	export let disabled = false;
</script>

<div class="flex items-center gap-2 my-3">
	<Switch {id} bind:checked bind:disabled />
	<div class="flex flex-col items-start gap-1">
		<Label for={id}>{label}</Label>
		{#if description}
			<small>{description}</small>
		{/if}
	</div>
</div>
