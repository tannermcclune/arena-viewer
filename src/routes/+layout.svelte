<script lang="ts">
	import '../app.css';
	import { injectAnalytics } from '@vercel/analytics/sveltekit';
	
	let { children } = $props();
	
	// Call the function to inject analytics
	injectAnalytics();
</script>

{@render children()}
