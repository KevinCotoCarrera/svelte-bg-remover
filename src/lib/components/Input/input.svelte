<script lang="ts">
	import type { InputTypes } from '$lib/types/input-type';

	export interface ExtendedInputTypes extends Omit<InputTypes, 'type'> {
		type?: InputTypes['type'] | 'textarea';
	}
	interface Props extends ExtendedInputTypes {
		label?: string;
		size: 'sm' | 'md' | 'lg';
		name?: string;
		placeholder?: string;
		value?: string;
		error?: string;
		id?: string;
	}

	let {
		value = $bindable(''),
		label = 'default label',
		size = 'sm',
		type = 'text',
		name = 'text',
		placeholder = 'Enter your',
		error = '',
		id = ''
	}: Props = $props();
	function autoGrowTextarea(event: Event) {
		const target = event.target as HTMLTextAreaElement;
		if (target) {
			target.style.height = 'auto';
			const lineHeight = parseFloat(getComputedStyle(target).lineHeight);
			const maxHeight = lineHeight * 8;
			target.style.height = `${Math.min(target.scrollHeight, maxHeight)}px`;

			target.style.overflowY = target.scrollHeight > maxHeight ? 'auto' : 'hidden';
		}
	}
	const sizeClasses = {
		sm: {
			label: 'text-tiny mb-1',
			input: 'min-w-36 py-2 text-tiny shadow-md'
		},
		md: {
			label: 'text-sm mb-2',
			input: 'min-w-52 py-3 text-sm shadow-lg'
		},
		lg: {
			label: 'text-base mb-3',
			input: 'min-w-60 py-4 text-base shadow-xl'
		}
	};

	let labelClass: string = sizeClasses[size as keyof typeof sizeClasses].label;
	let inputClass: string = sizeClasses[size as keyof typeof sizeClasses].input;
</script>

<div class="w-64">
	<label for={id} class={`flex flex-col ${labelClass} text-gray-800 dark:text-gray-200`}>
		{label}
		{#if type === 'textarea'}
			<textarea
				{id}
				bind:value
				class={`genericInput ${inputClass}`}
				{placeholder}
				rows="2"
				style="resize: none;"
				oninput={autoGrowTextarea}
			></textarea>
		{:else}
			<input {id} {type} bind:value class={`genericInput ${inputClass}`} {name} {placeholder} />
		{/if}

		{#if error}
			<p class="text-sm text-red-500">{error}</p>
		{/if}
	</label>
</div>
