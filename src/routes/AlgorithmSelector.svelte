<script lang="ts">
	// @ts-nocheck

	import { isOpen, selectedAlgorithm } from '../stores/store.svelte.js';
	import { algorithmComponents } from '../stores/algorithmComponents';

	let algorithmGroups = {
		divideAndConquer: false,
		dynamicProgramming: false,
		greedyAlgorithms: false,
		sortingAlgorithms: false,
		searchTrees: false,
		stringAlgorithms: false,
		shortestPath: false,
		minimumSpanningTree: false,
		geometryAlgorithms: false
	};

	function selectAlgorithm(algorithm: string) {
		if (algorithmComponents[algorithm]) {
			selectedAlgorithm.set(algorithm);
		} else {
			console.error(`Algorithm '${algorithm}' not found.`);
		}
		isOpen.set(false);
	}

	function toggleGroup(group: keyof typeof algorithmGroups) {
		algorithmGroups[group] = !algorithmGroups[group];
	}
</script>

<div class="input-container">
	<div class="algorithm-group">
		<button class="algorithm-group-button" on:click={() => toggleGroup('divideAndConquer')}>
			Divide And Conquer
		</button>
		<div class="algorithm-list {algorithmGroups.divideAndConquer ? 'visible' : ''}">
			<button class="algorithm-button" on:click={() => selectAlgorithm('factorialCalculator')}>Factorial Calculation</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('towersOfHanoi')}>Towers of Hanoi</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('hilbertCurves')}>Hilbert Curves</button>
		</div>
	</div>

	<div class="algorithm-group">
		<button class="algorithm-group-button" on:click={() => toggleGroup('dynamicProgramming')}>
			Dynamic Programming
		</button>
		<div class="algorithm-list {algorithmGroups.dynamicProgramming ? 'visible' : ''}">
			<button class="algorithm-button" on:click={() => selectAlgorithm('maxSumPath')}>Maximum Sum Path</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('coinChange')}>Coin Change Algorithm</button>
		</div>
	</div>
	<div class="algorithm-group">
		<button class="algorithm-group-button" on:click={() => toggleGroup('greedyAlgorithms')}>
			Greedy Algorithms
		</button>
		<div class="algorithm-list {algorithmGroups.greedyAlgorithms ? 'visible' : ''}">
			<button class="algorithm-button" on:click={() => selectAlgorithm('fractionalKnapsack')}>Fractional Knapsack (Without Repetition)</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('fractionalKnapsackRep')}>Fractional Knapsack (With Repetition)</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('travelingProblem')}>Traveling Salesman Problem</button>
		</div>
	</div>
	<div class="algorithm-group">
		<button class="algorithm-group-button" on:click={() => toggleGroup('sortingAlgorithms')}>
			Sorting Algorithms
		</button>
		<div class="algorithm-list {algorithmGroups.sortingAlgorithms ? 'visible' : ''}">
			<button class="algorithm-button" on:click={() => selectAlgorithm('mergeSort')}>Merge Sort</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('insertionSort')}>Insertion Sort</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('heapSort')}>Heap Sort</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('quickSort')}>Quick Sort</button>
		</div>
	</div>
	<div class="algorithm-group">
		<button class="algorithm-group-button" on:click={() => toggleGroup('searchTrees')}>
			Tree Algorithms
		</button>
		<div class="algorithm-list {algorithmGroups.searchTrees ? 'visible' : ''}">
			<button class="algorithm-button" on:click={() => selectAlgorithm('binaryTree')}>Binary Tree</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('avlTree')}>AVL Tree</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('redBlackTree')}>Red–Black Tree</button>
		</div>
	</div>
	<div class="algorithm-group">
		<button class="algorithm-group-button" on:click={() => toggleGroup('stringAlgorithms')}>
			String Algorithms
		</button>
		<div class="algorithm-list {algorithmGroups.stringAlgorithms ? 'visible' : ''}">
			<button class="algorithm-button" on:click={() => selectAlgorithm('rabinKarp')}>Rabin–Karp Algorithm</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('knuthMorrisPratt')}>Knuth–Morris–Pratt Algorithm</button>
		</div>
	</div>
	<div class="algorithm-group">
		<button class="algorithm-group-button" on:click={() => toggleGroup('shortestPath')}>
			Shortest Path Search
		</button>
		<div class="algorithm-list {algorithmGroups.shortestPath ? 'visible' : ''}">
			<button class="algorithm-button" on:click={() => selectAlgorithm('dijkstraShortestPath')}>Dijkstra’s Shortest Path</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('bellmanFordShortestPath')}>Bellman–Ford Shortest Path</button>
		</div>
	</div>
	<div class="algorithm-group">
		<button class="algorithm-group-button" on:click={() => toggleGroup('minimumSpanningTree')}>
			Minimum Spanning Tree
		</button>
		<div class="algorithm-list {algorithmGroups.minimumSpanningTree ? 'visible' : ''}">
			<button class="algorithm-button" on:click={() => selectAlgorithm('primMinimumSpanningTree')}>Prim Algorithm</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('kruskalMinimumSpanningTree')}>Kruskal Algorithm</button>
		</div>
	</div>
	<div class="algorithm-group">
		<button class="algorithm-group-button" on:click={() => toggleGroup('geometryAlgorithms')}>
			Geometric Algorithms
		</button>
		<div class="algorithm-list {algorithmGroups.geometryAlgorithms ? 'visible' : ''}">
			<button class="algorithm-button" on:click={() => selectAlgorithm('convexHull')}>Convex Hull (Brute Force)</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('jarvisMarch')}>Jarvis March</button>
			<button class="algorithm-button" on:click={() => selectAlgorithm('grahamScan')}>Graham Scan</button>
		</div>
	</div>
</div>

<style>
	.input-container {
		display: inline-block;
		background-color: #484848;
		width: 100%;
		display: block;
		overflow: auto;
	}

	.algorithm-group-button {
		background-color: #404040;
		color: white;
		border: none;
		cursor: pointer;
		padding: 10px;
		width: 100%;
		text-align: left;
	}

	.algorithm-button {
		border: none;
		cursor: pointer;
		text-align: right;
		padding-right: 10px;
		padding: 5px 10px 5px 0px;
		background: none;
		width: 100%;
	}

	.algorithm-button:hover {
		background-color: #555555;
	}

	.algorithm-group-button:hover {
		background-color: #505050;
	}

	.algorithm-list {
		display: none;
	}

	.algorithm-list.visible {
		display: block;
	}
</style>
