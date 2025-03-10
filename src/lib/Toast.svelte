<script lang="ts">
	import { onMount, afterUpdate } from 'svelte';
	import type { ToastClassnames, ToastProps } from './types.js';
	import { toastState, useEffect } from './state.js';
	import { cn } from './internal/helpers.js';
	import type { Expand } from './internal/types.js';

	type $$Props = Expand<ToastProps>;

	// Default lifetime of a toasts (in ms)
	const TOAST_LIFETIME = 4000;

	// Default gap between toasts
	const GAP = 14;

	const SWIPE_TRESHOLD = 20;

	const TIME_BEFORE_UNMOUNT = 200;

	const defaultClasses: ToastClassnames = {
		toast: '',
		title: '',
		description: '',
		loader: '',
		closeButton: '',
		cancelButton: '',
		actionButton: '',
		action: '',
		warning: '',
		error: '',
		success: '',
		default: '',
		info: '',
		loading: ''
	};

	const { toasts, heights, removeHeight, addHeight, dismiss } = toastState;

	export let toast: $$Props['toast'];
	export let index: $$Props['index'];
	export let expanded: $$Props['expanded'];
	export let invert: $$Props['invert'];
	export let position: $$Props['position'];
	export let visibleToasts: $$Props['visibleToasts'];
	export let expandByDefault: $$Props['expandByDefault'];
	export let closeButton: $$Props['closeButton'];
	export let interacting: $$Props['interacting'];
	export let cancelButtonStyle: $$Props['cancelButtonStyle'] = '';
	export let actionButtonStyle: $$Props['actionButtonStyle'] = '';
	export let duration: $$Props['duration'] = 4000;
	export let descriptionClass: $$Props['descriptionClass'] = '';
	export let classes: $$Props['classes'] = {};
	export let unstyled: $$Props['unstyled'] = false;

	let mounted = false;
	let removed = false;
	let swiping = false;
	let swipeOut = false;
	let offsetBeforeRemove = 0;
	let initialHeight = 0;
	let toastRef: HTMLLIElement;

	$: classes = { ...defaultClasses, ...classes };

	$: isFront = index === 0;
	$: isVisible = index + 1 <= visibleToasts;
	$: toastType = toast.type ?? 'default';
	$: toastClass = toast.class || '';
	$: toastDescriptionClass = toast.descriptionClass || '';

	// Height index is used to calculate the offset as it gets updated before the toast array, which means we can calculate the new layout faster.
	$: heightIndex =
		$heights.findIndex((height) => height.toastId === toast.id) || 0;

	let offset = 0;
	let closeTimerStartTimeRef = 0;

	let lastCloseTimerStartTimeRef = 0;
	let pointerStartRef: { x: number; y: number } | null = null;

	$: coords = position.split('-');
	$: toastsHeightBefore = $heights.reduce((prev, curr, reducerIndex) => {
		// Calculate offset up untill current  toast
		if (reducerIndex >= heightIndex) return prev;

		return prev + curr.height;
	}, 0);
	$: invert = toast.invert || invert;
	$: disabled = toastType === 'loading';

	$: {
		offset = heightIndex * GAP + toastsHeightBefore;
	}

	function deleteToast() {
		removed = true;
		// Save the offset for the exit swipe animation
		offsetBeforeRemove = offset;

		removeHeight(toast.id);

		setTimeout(() => {
			dismiss(toast.id);
		}, TIME_BEFORE_UNMOUNT);
	}

	let timeoutId: ReturnType<typeof setTimeout>;

	let remainingTime = toast.duration || duration || TOAST_LIFETIME;

	$: if (toast.updated) {
		// if the toast has been updated after the initial render,
		// we want to reset the timer and set the remaining time to the
		// new duration
		clearTimeout(timeoutId);
		remainingTime = toast.duration || duration || TOAST_LIFETIME;
		startTimer();
	}

	// If toast's duration changes, it will be out of sync with the
	// remainingAtTimeout, so we know we need to restart the timer
	// with the new duration

	// Pause the tmer on each hover
	function pauseTimer() {
		if (lastCloseTimerStartTimeRef < closeTimerStartTimeRef) {
			// Get the elapsed time since the timer started
			const elapsedTime = new Date().getTime() - closeTimerStartTimeRef;
			remainingTime = remainingTime - elapsedTime;
		}

		lastCloseTimerStartTimeRef = new Date().getTime();
	}

	function startTimer() {
		closeTimerStartTimeRef = new Date().getTime();
		// Let the toast know it has started
		timeoutId = setTimeout(() => {
			toast.onAutoClose?.(toast);
			deleteToast();
		}, remainingTime);
	}

	$: isPromiseLoadingOrInfiniteDuration =
		(toast.promise && toastType === 'loading') ||
		toast.duration === Number.POSITIVE_INFINITY;

	/**
	 * Hacky useEffect impl.
	 * https://github.com/sveltejs/svelte/issues/5283#issuecomment-678759305
	 */
	let effect;
	$: effect = useEffect(() => {
		if (!isPromiseLoadingOrInfiniteDuration) {
			if (expanded || interacting) {
				pauseTimer();
			} else {
				startTimer();
			}
		}

		return () => clearTimeout(timeoutId);
	});
	$: $effect;

	onMount(() => {
		mounted = true;

		return () => removeHeight(toast.id);
	});

	afterUpdate(() => {
		const height = toastRef.getBoundingClientRect().height;

		// Add toast height tot heights array after the toast is mounted
		initialHeight = height;

		addHeight({ toastId: toast.id, height });
	})

	$: if (toast.delete) {
		deleteToast();
	}

	function onPointerDown(event: PointerEvent) {
		if (disabled) {
			return;
		}

		offsetBeforeRemove = offset;
		const target = event.target as HTMLElement;
		// Ensure we maintain correct pointer capture even when going outside of the toast (e.g. when swiping)
		target.setPointerCapture(event.pointerId);
		if (target.tagName === 'BUTTON') {
			return;
		}
		swiping = true;
		pointerStartRef = { x: event.clientX, y: event.clientY };
	}

	function onPointerUp() {
		if (swipeOut) {
			return;
		}

		pointerStartRef = null;
		const swipeAmount = Number(
			toastRef?.style
				.getPropertyValue('--swipe-amount')
				.replace('px', '') || 0
		);

		// Remove only if treshold is met
		if (Math.abs(swipeAmount) >= SWIPE_TRESHOLD) {
			offsetBeforeRemove = offset;
			toast.onDismiss?.(toast);
			deleteToast();
			swipeOut = true;
			return;
		}

		toastRef.style.setProperty('--swipe-amount', '0px');
		swiping = false;
	}

	function onPointerMove(event: PointerEvent) {
		if (!pointerStartRef) {
			return;
		}

		const yPosition = event.clientY - pointerStartRef!.y;
		const xPosition = event.clientX - pointerStartRef!.x;

		const clamp = coords[0] === 'top' ? Math.min : Math.max;
		const clampedY = clamp(0, yPosition);
		const swipeStartThreshold = event.pointerType === 'touch' ? 10 : 2;
		const isAllowedToSwipe = Math.abs(clampedY) > swipeStartThreshold;

		if (isAllowedToSwipe) {
			toastRef.style.setProperty('--swipe-amount', `${yPosition}px`);
		} else if (Math.abs(xPosition) > swipeStartThreshold) {
			// User is swiping in wrong direction so we disable swipe gesture
			// for the current pointer down interaction
			pointerStartRef = null;
		}
	}
</script>

<li
	bind:this={toastRef}
	aria-live={toast.important ? 'assertive' : 'polite'}
	aria-atomic="true"
	role="status"
	tabIndex={0}
	class={cn(
		$$props.class,
		toastClass,
		classes?.toast,
		toast?.classes?.toast,
		classes?.[toastType],
		toast?.classes?.[toastType]
	)}
	data-sonner-toast=""
	data-styled={!(toast.component || toast?.unstyled || unstyled)}
	data-mounted={mounted}
	data-promise={Boolean(toast.promise)}
	data-removed={removed}
	data-visible={isVisible}
	data-y-position={coords[0]}
	data-x-position={coords[1]}
	data-index={index}
	data-front={isFront}
	data-swiping={swiping}
	data-type={toastType}
	data-invert={invert}
	data-swipe-out={swipeOut}
	data-expanded={Boolean(expanded || (expandByDefault && mounted))}
	style={`${$$props.style} ${toast.style}`}
	style:--index={index}
	style:--toasts-before={index}
	style:--z-index={$toasts.length - index}
	style:--offset={`${removed ? offsetBeforeRemove : offset}px`}
	style:--initial-height={expandByDefault ? 'auto' : `${initialHeight}px`}
	on:pointerdown={onPointerDown}
	on:pointerup={onPointerUp}
	on:pointermove={onPointerMove}
>
	{#if closeButton && !toast.component}
		<button
			aria-label="Close toast"
			data-disabled={disabled}
			data-close-button
			on:click={disabled
				? undefined
				: () => {
						deleteToast();
						toast.onDismiss?.(toast);
					}}
			class={cn(classes?.closeButton, toast?.classes?.closeButton)}
		>
			<svg
				xmlns="http://www.w3.org/2000/svg"
				width="12"
				height="12"
				viewBox="0 0 24 24"
				fill="none"
				stroke="currentColor"
				stroke-width="1.5"
				stroke-linecap="round"
				stroke-linejoin="round"
			>
				<line x1="18" y1="6" x2="6" y2="18" />
				<line x1="6" y1="6" x2="18" y2="18" />
			</svg>
		</button>
	{/if}

	{#if toast.component}
		<svelte:component
			this={toast.component}
			{...toast.componentProps}
			on:closeToast={deleteToast}
		></svelte:component>
	{:else}
		{#if toastType !== 'default' || toast.icon || toast.promise}
			<div data-icon="">
				{#if (toast.promise || toastType === 'loading') && !toast.icon}
					<slot name="loading-icon" />
				{/if}
				{#if toast.icon}
					<svelte:component this={toast.icon}></svelte:component>
				{:else}
					{#if toastType === 'success'}
						<slot name="success-icon" />
					{:else if toastType === 'error'}
						<slot name="error-icon" />
					{:else if toastType === 'warning'}
						<slot name="warning-icon" />
					{:else if toastType === 'info'}
						<slot name="info-icon" />
					{/if}
				{/if}
			</div>
		{/if}
		<div data-content="">
			{#if toast.title}
				<div
					data-title=""
					class={cn(classes?.title, toast?.classes?.title)}
				>
					{#if typeof toast.title !== 'string'}
						<svelte:component
							this={toast.title}
							{...toast.componentProps}
						/>
					{:else}
						{toast.title}
					{/if}
				</div>
			{/if}
			{#if toast.description}
				<div
					data-description=""
					class={cn(
						descriptionClass,
						toastDescriptionClass,
						classes?.description,
						toast.classes?.description
					)}
				>
					{#if typeof toast.description !== 'string'}
						<svelte:component
							this={toast.description}
							{...toast.componentProps}
						/>
					{:else}
						{toast.description}
					{/if}
				</div>
			{/if}
		</div>
		{#if toast.cancel}
			<button
				data-button
				data-cancel
				style={cancelButtonStyle}
				class={cn(classes?.cancelButton, toast?.classes?.cancelButton)}
				on:click={() => {
					deleteToast();
					if (toast.cancel?.onClick) {
						toast.cancel.onClick();
					}
				}}
			>
				{toast.cancel.label}
			</button>
		{/if}
		{#if toast.action}
			<button
				data-button=""
				style={actionButtonStyle}
				class={cn(classes?.actionButton, toast?.classes?.actionButton)}
				on:click={(event) => {
					toast.action?.onClick(event);
					if (event.defaultPrevented) return;
					deleteToast();
				}}
			>
				{toast.action.label}
			</button>
		{/if}
	{/if}
</li>
