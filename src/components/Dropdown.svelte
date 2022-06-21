<script>
	import { createEventDispatcher, onDestroy, onMount, tick } from 'svelte';
	import VirtualList from 'svelte-tiny-virtual-list';
	import { isOutOfViewport } from './../lib/utils.js';
	import { slide } from 'svelte/transition';
	import { quintOut } from 'svelte/easing';
	export let lazyDropdown;

	export let creatable;
	export let maxReached = false;
	export let dropdownIndex = 0;
	export let renderer;
	export let disableHighlight;
	export let items = [];
	export let alreadyCreated;
	export let virtualList;
	export let vlItemSize;
	export let vlHeight;
	/** internal props */
	export let inputValue; // value only, not store
	export let multiple;
	export let listIndex;
	export let hasDropdownOpened;
	export let listMessage;
	export let disabledField;
	export let createLabel;
	export let metaKey;
	export let itemComponent;

	export function scrollIntoView(params) {
		if (virtualList) return;
		const focusedEl = container.querySelector(`[data-pos="${dropdownIndex}"]`);
		if (!focusedEl) return;
		const focusedRect = focusedEl.getBoundingClientRect();
		const menuRect = scrollContainer.getBoundingClientRect();
		const overScroll = focusedEl.offsetHeight / 3;
		const centerOffset = params && params.center ? scrollContainer.offsetHeight / 2 : 0;
		switch (true) {
			case focusedEl.offsetTop < scrollContainer.scrollTop:
				scrollContainer.scrollTop = focusedEl.offsetTop - overScroll + centerOffset;
				break;
			case focusedEl.offsetTop + focusedRect.height > scrollContainer.scrollTop + menuRect.height:
				scrollContainer.scrollTop =
					focusedEl.offsetTop +
					focusedRect.height -
					scrollContainer.offsetHeight +
					overScroll +
					centerOffset;
				break;
		}
	}

	export function getDimensions() {
		if (virtualList) {
			return [scrollContainer.offsetHeight, vl_itemSize];
		}
		return [scrollContainer.offsetHeight, container.firstElementChild.offsetHeight];
	}

	const dispatch = createEventDispatcher();

	let container;
	let scrollContainer;
	let isMounted = false;
	let hasEmptyList = false;
	let renderDropdown = !lazyDropdown;
	$: currentListLength = items.length;

	let vl_height = vlHeight;
	let vl_itemSize = vlItemSize;
	$: vl_listHeight = Math.min(
		vl_height,
		Array.isArray(vl_itemSize)
			? vl_itemSize.reduce((res, num) => {
					res += num;
					return res;
			  }, 0)
			: items.length * vl_itemSize
	);
	let vl_autoMode = vlHeight === null && vlItemSize === null;
	let refVirtualList;

	$: {
		hasEmptyList = items.length < 1 && (creatable ? !inputValue : true);
		// required when changing item list 'on-the-fly' for VL
		if (virtualList && vl_autoMode && isMounted && renderDropdown) {
			if (hasEmptyList) dropdownIndex = null;
			vl_itemSize = 0;
			tick().then(virtualListDimensionsResolver).then(positionDropdown);
		}
	}

	function positionDropdown(val) {
		if (!scrollContainer && !renderDropdown) return;
		const outVp = isOutOfViewport(scrollContainer);
		if (outVp.bottom && !outVp.top) {
			scrollContainer.parentElement.style.bottom =
				scrollContainer.parentElement.parentElement.clientHeight + 1 + 'px';
			// FUTURE: debounce ....
		} else if (!val || outVp.top) {
			scrollContainer.parentElement.style.bottom = '';
		}
	}

	function virtualListDimensionsResolver() {
		if (!refVirtualList) return;
		const pixelGetter = (el, prop) => {
			const styles = window.getComputedStyle(el);
			let {
				groups: { value, unit }
			} = styles[prop].match(/(?<value>\d+)(?<unit>[a-zA-Z]+)/);
			value = parseFloat(value);
			if (unit !== 'px') {
				const el =
					unit === 'rem' ? document.documentElement : scrollContainer.parentElement.parentElement;
				const multipler = parseFloat(window.getComputedStyle(el).fontSize.match(/\d+/).shift());
				value = multipler * value;
			}
			return value;
		};
		vl_height =
			pixelGetter(scrollContainer, 'maxHeight') -
			pixelGetter(scrollContainer, 'paddingTop') -
			pixelGetter(scrollContainer, 'paddingBottom');
		// get item size (hacky style)
		scrollContainer.parentElement.style = 'opacity: 0; display: block';
		const firstItem = refVirtualList.$$.ctx[1].firstElementChild.firstElementChild;
		if (firstItem) {
			firstItem.style = '';
			const firstSize = firstItem.getBoundingClientRect().height;
			const secondItem =
				refVirtualList.$$.ctx[1].firstElementChild.firstElementChild.nextElementSibling;
			let secondSize;
			if (secondItem) {
				secondItem.style = '';
				secondSize = secondItem.getBoundingClientRect().height;
			}
			if (firstSize !== secondSize) {
				const groupHeaderSize = items[0].$isGroupHeader ? firstSize : secondSize;
				const regularItemSize = items[0].$isGroupHeader ? secondSize : firstSize;
				vl_itemSize = items.map((opt) => (opt.$isGroupHeader ? groupHeaderSize : regularItemSize));
			} else {
				vl_itemSize = firstSize;
			}
		}
		scrollContainer.parentElement.style = '';
	}

	let dropdownStateSubscription = () => {};
	let onScrollHandler = null;
	/** ************************************ lifecycle */
	onMount(() => {
		/** ************************************ flawless UX related tweak */
		dropdownStateSubscription = hasDropdownOpened.subscribe((val) => {
			if (!renderDropdown && val) renderDropdown = true;
			tick().then(() => {
				positionDropdown(val);
				val && scrollIntoView({ center: true });
			});
			if (!onScrollHandler) onScrollHandler = () => positionDropdown(val);
			// bind/unbind scroll listener
			document[val ? 'addEventListener' : 'removeEventListener']('scroll', onScrollHandler, {
				passive: true
			});
		});
		isMounted = true;
	});
	onDestroy(() => dropdownStateSubscription());
</script>

{#if isMounted && renderDropdown}
	<div
		class="sv-dropdown "
		class:is-virtual={virtualList}
		aria-expanded={$hasDropdownOpened}
		on:mousedown|preventDefault
	>
		<div
			class="sv-dropdown-scroll"
			class:is-empty={!items.length}
			bind:this={scrollContainer}
			tabindex="-1"
		>
			<div class="sv-dropdown-content" bind:this={container} class:max-reached={maxReached}>
				{#if items.length && !maxReached}
					{#if virtualList}
						<VirtualList
							bind:this={refVirtualList}
							width="100%"
							height={vl_listHeight}
							itemCount={items.length}
							itemSize={vl_itemSize}
							scrollToAlignment="auto"
							scrollToIndex={!multiple && dropdownIndex ? parseInt(dropdownIndex) : null}
						>
							<div
								slot="item"
								let:index
								let:style
								{style}
								class="sv-dd-item"
								class:sv-dd-item-active={index == dropdownIndex}
								class:sv-group-item={items[index].$isGroupItem}
								class:sv-group-header={items[index].$isGroupHeader}
							>
								<svelte:component
									this={itemComponent}
									formatter={renderer}
									index={listIndex.map[index]}
									isDisabled={items[index][disabledField]}
									item={items[index]}
									{inputValue}
									{disableHighlight}
									on:hover
									on:select
								/>
							</div>
						</VirtualList>
					{:else}
						<div in:slide={{ delay: 120, duration: 1000, easing: quintOut }}>
							{#each items as opt, i}
								<div
									data-pos={listIndex.map[i]}
									class="sv-dd-item"
									class:sv-dd-item-active={listIndex.map[i] == dropdownIndex}
									class:sv-group-item={opt.$isGroupItem}
									class:sv-group-header={opt.$isGroupHeader}
								>
									<svelte:component
										this={itemComponent}
										formatter={renderer}
										index={listIndex.map[i]}
										isDisabled={opt[disabledField]}
										item={opt}
										{inputValue}
										{disableHighlight}
										on:hover
										on:select
									/>
								</div>
							{/each}
						</div>
					{/if}
				{/if}
				{#if hasEmptyList || maxReached}
					<div class="empty-list-row">{listMessage}</div>
				{/if}
			</div>
		</div>
		<!-- scroll container end -->
		{#if inputValue && creatable && !maxReached}
			<div class="creatable-row-wrap">
				<div
					class="creatable-row"
					on:click={dispatch('select', inputValue)}
					on:mouseenter={dispatch('hover', listIndex.last)}
					class:active={currentListLength == dropdownIndex}
					class:is-disabled={alreadyCreated.includes(inputValue)}
				>
					{@html createLabel(inputValue)}
					{#if currentListLength != dropdownIndex}
						<span class="shortcut"><kbd>{metaKey}</kbd>+<kbd>Enter</kbd></span>
					{/if}
				</div>
			</div>
		{/if}
	</div>
{/if}

<style>
	.sv-dropdown {
		@apply bg-base-300 w-full overflow-y-auto overflow-x-hidden hidden z-[2] absolute box-border rounded-md shadow-xl text-base-content;
	}

	/*.sv-dropdown.is-virtual .sv-dropdown-scroll {
  @apply overflow-y-hidden;
}*/
	.sv-dropdown-scroll {
		@apply max-h-60 overflow-x-hidden overflow-y-auto transition-all duration-1000 px-1 py-2;
	}

	.sv-dropdown-scroll::-webkit-scrollbar {
    @apply bg-transparent w-1;
	}

	/* background of the scrollbar except button or resizer */
	.sv-dropdown-scroll::-webkit-scrollbar-track {
    @apply bg-base-content/20;
	}
	.sv-dropdown-scroll::-webkit-scrollbar-track:hover {
    @apply bg-base-content/30;
	}

	/* scrollbar itself */
	.sv-dropdown-scroll::-webkit-scrollbar-thumb {
    @apply bg-neutral rounded-2xl w-1;
	}
	.sv-dropdown-scroll::-webkit-scrollbar-thumb:hover {
    @apply bg-primary;
	}

	/* set button(top and bottom of the scrollbar) */
	.sv-dropdown-scroll::-webkit-scrollbar-button {
		display: none;
	}

	.sv-dropdown-scroll.is-empty {
		@apply p-0;
	}
	.sv-dropdown[aria-expanded='true'] {
		display: block;
	}
	.sv-dropdown-content.max-reached {
		opacity: 0.75;
		cursor: not-allowed;
	}

	.sv-dropdown-scroll:not(.is-empty) + .creatable-row-wrap {
		@apply border-t-2 bg-neutral-focus;
	}

	.creatable-row-wrap {
		@apply p-1;
	}
	.creatable-row {
		@apply box-border flex justify-center items-center rounded-sm p-1;
	}
	.creatable-row:hover,
	.creatable-row:active,
	.creatable-row.active {
		@apply bg-success text-success-content;
	}
	.creatable-row.active.is-disabled {
		@apply opacity-50 bg-warning;
	}
	.creatable-row.is-disabled {
		@apply opacity-50 cursor-not-allowed;
	}

	.shortcut {
		@apply flex items-center content-center;
	}
	.shortcut > kbd {
		@apply border-2 border-gray-600 rounded-sm px-0 py-2 bg-base-300 text-base-content h-6;
	}

	.empty-list-row {
		min-width: 0px;
		box-sizing: border-box;
		border-radius: 2px;
		text-overflow: ellipsis;
		white-space: nowrap;
		box-sizing: border-box;
		border-radius: 2px;
		overflow: hidden;
		padding: 7px 7px 7px 10px;
		text-align: left;
	}
</style>
