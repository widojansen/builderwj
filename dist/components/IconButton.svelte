<script>
	import Icon from '@iconify/svelte'

	export let icon = 'carbon:overflow-menu-vertical'
	export let color = null
	let clicked = false
</script>

<button class="show-menu" class:active={clicked} on:click style:color>
	<Icon {icon} />
</button>

<style>
	button.show-menu {
		flex: 1;
		padding: 0.25rem;
		border-radius: 4px;
		transition: 0.1s;
		/* opacity: 0; */
		opacity: var(--IconButton-opacity, 1);
	}

		button.show-menu:hover,
		button.show-menu.active {
			/* background: #f4f4f6; */
			/* opacity: 1; */
		}</style>
