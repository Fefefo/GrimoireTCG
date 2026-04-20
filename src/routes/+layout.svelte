<script lang="ts">
	import './layout.css';
	import favicon from '$lib/assets/favicon.svg';
	import { ModeWatcher, setMode } from 'mode-watcher';

	setMode('dark');
	let { children } = $props();
</script>

<ModeWatcher />
<svelte:head><link rel="icon" href={favicon} /></svelte:head>
{@render children()}
