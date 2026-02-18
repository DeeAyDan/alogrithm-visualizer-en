<script>
	import { marked } from 'marked';
	import readme from '$lib/content/README.md?raw';

	let content = marked(readme);
</script>

<div class="markdown">
	{@html content}
</div>

<style>
	.markdown {
		padding: 20px;
		height: 100%;
	}
</style>
