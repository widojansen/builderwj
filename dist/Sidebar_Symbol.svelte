<script>
	import { createEventDispatcher, getContext } from 'svelte'
	import _ from 'lodash-es'
	import Icon from '@iconify/svelte'
	import modal from './stores/app/modal'
	import { showingIDE, userRole } from './stores/app/misc'
	import MenuPopup from './components/MenuPopup.svelte'
	import IconButton from './components/IconButton.svelte'
	import { processCode, processCSS } from './utils'
	import { code as siteCode } from './stores/data/site'
	import { code as pageCode } from './stores/app/activePage'
	import { locale } from './stores/app/misc'
	import { click_to_copy } from './utilities'
	import IFrame from './views/modal/ComponentLibrary/IFrame.svelte'
	const dispatch = createEventDispatcher()

	export let symbol
	export let controls_enabled = true
	export let header_hidden = false

	function edit_symbol_content(symbol) {
		$showingIDE = false
		modal.show(
			'SYMBOL_EDITOR',
			{
				symbol,
				header: {
					title: `Edit ${symbol.name || 'Block'}`,
					icon: 'fas fa-check',
					button: {
						label: `Save Block`,
						icon: 'fas fa-check',
						onclick: () => {
							modal.hide()
						}
					}
				}
			},
			{
				showSwitch: true,
				disabledBgClose: true
			}
		)
	}

	function edit_symbol_code(symbol) {
		$showingIDE = true
		modal.show(
			'SYMBOL_EDITOR',
			{
				symbol,
				header: {
					title: `Edit ${symbol.title || 'Block'}`,
					icon: 'fas fa-check',
					button: {
						label: `Save Block`,
						icon: 'fas fa-check',
						onclick: () => {
							modal.hide()
						}
					}
				}
			},
			{
				showSwitch: true,
				disabledBgClose: true
			}
		)
	}

	let name_el

	// move cursor to end of name
	$: if (name_el) {
		const range = document.createRange()
		const sel = window.getSelection()
		range.setStart(name_el, 1)
		range.collapse(true)

		sel?.removeAllRanges()
		sel?.addRange(range)
	}

	let renaming = false
	async function toggle_name_input() {
		renaming = !renaming
		// workaround for inability to see cursor when div empty
		if (symbol.name === '') {
			symbol.name = 'Block'
		}
	}

	let height = 0

	let componentCode
	let cachedSymbol = {}
	let component_error
	$: compile_component_code(symbol, $locale)
	async function compile_component_code(symbol, language) {
		if (
			_.isEqual(cachedSymbol.code, symbol.code) &&
			_.isEqual(cachedSymbol.content, symbol.content)
		) {
			return
		}

		const parent_css = await processCSS($siteCode.css + $pageCode.css)
		let res = await processCode({
			component: {
				...symbol.code,
				head: $siteCode.html.head + $pageCode.html.head,
				css: parent_css + symbol.code.css,
				html: `
          ${symbol.code.html}
          ${$pageCode.html.below}`,
				data: symbol.content[language]
			},
			buildStatic: true,
			hydrated: true
		})
		if (res.error) {
			component_error = res.error
		} else {
			component_error = null
			res.css = res.css + parent_css
			componentCode = res
			cachedSymbol = _.cloneDeep({ code: symbol.code, content: symbol.content })
		}
	}
</script>

<div class="sidebar-symbol">
	<header style:opacity={header_hidden ? 0 : 1}>
		{#if renaming}
			<!-- svelte-ignore a11y-autofocus -->
			<!-- svelte-ignore a11y-no-static-element-interactions -->
			<div
				bind:this={name_el}
				contenteditable
				autofocus
				class="name"
				on:blur={toggle_name_input}
				on:keydown={(e) => {
					if (e.code === 'Enter') {
						e.preventDefault()
						e.target.blur()
						dispatch('rename', e.target.textContent)
						renaming = false
					}
				}}
			>
				{symbol.name}
			</div>
		{:else}
			<div class="name">
				<h3>{symbol.name}</h3>
			</div>
		{/if}
		{#if controls_enabled}
			<div class="symbol-options">
				<IconButton
					icon="material-symbols:edit-square-outline-rounded"
					on:click={() => edit_symbol_content(symbol)}
				/>
				{#if $userRole === 'DEV'}
					<IconButton icon="material-symbols:code" on:click={() => edit_symbol_code(symbol)} />
				{/if}
				<MenuPopup
					icon="carbon:overflow-menu-vertical"
					options={[
						getContext('DEBUGGING')
							? {
									label: `${symbol.id.slice(0, 5)}...`,
									icon: 'ph:copy-duotone',
									on_click: (e) => click_to_copy(e.target, symbol.id)
							  }
							: {},
						{
							label: 'Duplicate',
							icon: 'bxs:duplicate',
							on_click: () => dispatch('duplicate')
						},
						{
							label: 'Rename',
							icon: 'ic:baseline-edit',
							on_click: toggle_name_input
						},
						{
							label: 'Download',
							icon: 'ic:baseline-download',
							on_click: () => dispatch('download')
						},
						{
							label: 'Delete',
							icon: 'ic:outline-delete',
							on_click: () => dispatch('delete')
						}
					]}
				/>
			</div>
		{/if}
	</header>
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<div class="symbol" on:mousedown on:mouseup>
		{#if component_error}
			<div class="error">
				<Icon icon="bxs:error" />
			</div>
		{:else}
			{#key componentCode}
				<IFrame bind:height {componentCode} />
			{/key}
		{/if}
	</div>
</div>

<style>
	.sidebar-symbol {
		--IconButton-opacity: 0;
	}

		.sidebar-symbol:hover:not(.dragging) {
			--IconButton-opacity: 1;
		}

		.sidebar-symbol header {
			display: flex;
			align-items: center;
			justify-content: space-between;
			padding: 6px 0;
			color: #e7e7e7;
			transition: opacity 0.2s;
		}

		.sidebar-symbol header .name {
				font-size: 13px;
				line-height: 16px;
			}

		.sidebar-symbol header .symbol-options {
				display: flex;
				align-items: center;
				color: #e7e7e7;
			}

		.sidebar-symbol header .symbol-options :global(svg) {
					height: 1rem;
					width: 1rem;
				}

		.sidebar-symbol .symbol {
			width: 100%;
			border-radius: 0.25rem;
			overflow: hidden;
			position: relative;
			cursor: grab;
			min-height: 2rem;
			transition: box-shadow 0.2s;
			border: 1px solid var(--color-gray-8);
		}
	.error {
		display: flex;
		justify-content: center;
		height: 100%;
		position: absolute;
		inset: 0;
		align-items: center;
		background: #ff0000;
	}
	[contenteditable] {
		outline: 0 !important;
	}</style>
