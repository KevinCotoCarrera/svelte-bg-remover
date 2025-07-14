<script lang="ts">
	import '../app.css';
	import Footer from '$lib/components/Footer/footer.svelte';
	let { children } = $props();
</script>

<div class="flex min-h-screen flex-col">
	<main class="flex-grow">
		{@render children()}
	</main>
	<Footer />
</div>
