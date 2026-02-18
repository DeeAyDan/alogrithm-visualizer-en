<script>
	import { marked } from 'marked';
	import readme from '$lib/../static/README.md?raw';

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
