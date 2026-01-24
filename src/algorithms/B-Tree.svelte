<script lang="ts">
	// @ts-nocheck
	import { onMount, tick } from 'svelte';
	import { cubicOut } from 'svelte/easing';
	import { fade, fly } from 'svelte/transition';
	import { tweened } from 'svelte/motion';
	import {
		currentStep,
		totalSteps,
		consoleLog,
		speed,
		algorithmStatus,
		selectedAlgorithmSourceCode,
		activeLine,
		runId
	} from '../stores/store.svelte.js';
	import { get } from 'svelte/store';
	import { waitUntilResume, delay, pauseIfNeeded, log, RunCancelled } from '../stores/utils.js';

	let maxElements = 5;
	let elementValue: number;
	let highlightedNode = null;
	let highlightedKeys = [];
	let svgNodes = [];
	let animationInProgress = false;
	let lastOperation = '';
	let animationPath = [];
	let splitAnimation = null;
	let insertKeyHighlight = null;
	let activeRunId = 0;
	let nodeIdCounter = 0;
	activeLine.set({ start: -1, end: -1 });
	
	let svgWidth = 800;
	let svgHeight = 500;
	let canvasWidth = 800;

const animDuration = tweened(1400 - get(speed) * 10, {
		duration: 0,
		easing: cubicOut
	});
$: animationMs = Math.max(500, get(animDuration));

const longDelay = () => Math.max(700, 1500 - get(speed) * 10);
const midDelay = () => Math.max(450, 1000 - get(speed) * 8);
const shortDelay = () => Math.max(280, 650 - get(speed) * 5);

	function beginRunSnapshot() {
		activeRunId = get(runId);
	}

	function cleanupRunState() {
		animationInProgress = false;
		algorithmStatus.set('idle');
		activeLine.set({ start: -1, end: -1 });
		highlightedNode = null;
		highlightedKeys = [];
		animationPath = [];
		splitAnimation = null;
		insertKeyHighlight = null;
	}

	async function withRunGuard(task) {
		beginRunSnapshot();
		try {
			await task();
		} catch (err) {
			if (err instanceof RunCancelled) {
				cleanupRunState();
				return;
			}
			throw err;
		}
	}

	let root = null;
	let animating = false;
	selectedAlgorithmSourceCode.set('Choose an operation!');

	const btreeInsertSource = `function insert(node, key) {
  
  if (node.keys.length === maxElements) {
    const newRoot = createNode();
    newRoot.children.push(node);
    newRoot.isLeaf = false;
    splitChild(newRoot, 0);
    root = newRoot;
    insertNonFull(newRoot, key);
  }
  else {
    insertNonFull(node, key);
  }
}

function insertNonFull(node, key) {
  if (node.isLeaf) {
    let i = node.keys.length - 1;

    while (i >= 0 && key < node.keys[i]) {
      node.keys[i + 1] = node.keys[i];
      i--;
    }
    node.keys[i + 1] = key;
  }
  else {
    let i = node.keys.length - 1;
    while (i >= 0 && key < node.keys[i]) i--;
    i++;

    if (node.children[i].keys.length === maxElements) {
      splitChild(node, i);

      if (key > node.keys[i]) i++;
    }
    insertNonFull(node.children[i], key);
  }
}`;

	const btreeSearchSource = `function search(node, key) {
  let i = 0;
		
  while (i < node.keys.length && key > node.keys[i]) i++;
    
  if (i < node.keys.length && key === node.keys[i]) {
    return true;
  }
  if (node.isLeaf) {
    return false;
  }
  return search(node.children[i], key);
}`;

	const btreeDeleteSource = `function delete(node, key) {
  let idx = 0;

  while (idx < node.keys.length && key > node.keys[idx]) idx++;
		
  if (idx < node.keys.length && node.keys[idx] === key) {
    if (node.isLeaf) {
      node.keys.splice(idx, 1);
    }
    else {
      // Successful deletion
    }
  }
  else {
    if (node.isLeaf) return;
    if (node.children[idx].keys.length < minKeys) {
      fixChild(node, idx);
    }
    delete(node.children[idx], key);
  }
}`;

	function resetTree() {
		root = null;
		highlightedNode = null;
		highlightedKeys = [];
		animationPath = [];
		splitAnimation = null;
		insertKeyHighlight = null;
		selectedAlgorithmSourceCode.set('Choose an operation!');
	}

	async function createSampleTree() {
		if (animationInProgress) return;
		await withRunGuard(async () => {
			animationInProgress = true;
			
			resetTree();
			algorithmStatus.set('running');
			consoleLog.set([]);
			currentStep.set(0);
			lastOperation = 'sample';
			
			const numElements = maxElements * 2;
			const elements = new Set();
			while (elements.size < numElements) {
				elements.add(Math.floor(Math.random() * 99) + 1);
			}
			const sortedElements = Array.from(elements).sort((a, b) => a - b);
			
		log('Generating sample tree...');
		await delay(longDelay());
		await pauseIfNeeded(activeRunId);
			
			for (const element of sortedElements) {
				log(`Insert: ${element}`);
				if (!root) {
					root = createNode();
					root.keys = [element];
					highlightedKeys = [{node: root, keyIndex: 0}];
					await updateTreeLayout();
				} else {
					await insert(root, element);
					await updateTreeLayout();
				}
				await delay(longDelay());
				await pauseIfNeeded(activeRunId);
			}
			
			log('Sample tree created.');
			highlightedNode = null;
			highlightedKeys = [];
			algorithmStatus.set('finished');
			animationInProgress = false;
			algorithmStatus.set('idle');
			selectedAlgorithmSourceCode.set('Sample B-tree created.');
		});
	}

	function createNode() {
		return {
			keys: [],
			children: [],
			isLeaf: true,
			x: canvasWidth / 2,
			y: 60,
			prevX: canvasWidth / 2,
			prevY: 60,
			new: true,
			id: ++nodeIdCounter
		};
	}

	onMount(() => {
		totalSteps.set(0);
		consoleLog.set([]);
		algorithmStatus.set('idle');
		speed.subscribe(value => {
			animDuration.set(1400 - value * 10, { duration: 0 });
		});
		
		updateCanvasSize();
	});
	
	function updateCanvasSize() {
		const treeDepth = estimateTreeDepth();
		const treeWidth = estimateTreeWidth();
		
		svgHeight = Math.max(300, treeDepth * 120 + 100);
		svgWidth = Math.max(500, treeWidth * 80 + 100);
		canvasWidth = svgWidth;
	}
	
	function estimateTreeDepth() {
		if (!root) return 1;
		
		function getDepth(node) {
			if (!node) return 0;
			if (node.isLeaf) return 1;
			
			let maxChildDepth = 0;
			for (const child of node.children) {
				maxChildDepth = Math.max(maxChildDepth, getDepth(child));
			}
			return 1 + maxChildDepth;
		}
		
		return getDepth(root);
	}
	
	function estimateTreeWidth() {
		if (!root) return 3;
		
		function countLeaves(node) {
			if (!node) return 0;
			if (node.isLeaf) return 1;
			
			let leaves = 0;
			for (const child of node.children) {
				leaves += countLeaves(child);
			}
			return Math.max(1, leaves);
		}
		
		return countLeaves(root) * 2;
	}

	function validateInput(): boolean {
		if (elementValue === undefined || elementValue === null) {
			log('Please enter a value!');
			return false;
		}
		if (typeof elementValue !== 'number') {
			log('Please enter a number!');
			return false;
		}
		if (elementValue < 0 || elementValue > 100) {
			log('Please enter a number between 0 and 100!');
			return false;
		}
		return true;
	}

	async function insertElement() {
		if (!validateInput() || animationInProgress) return;
		await withRunGuard(async () => {
			animationInProgress = true;
		log(`Insert: ${elementValue}`);
			algorithmStatus.set('running');
			consoleLog.set([]);
			currentStep.set(0);
			lastOperation = 'insert';
			insertKeyHighlight = elementValue;

			selectedAlgorithmSourceCode.set(btreeInsertSource);
			activeLine.set({ start: 0, end: 0 });

			if (!root) {
				root = createNode();
				root.keys = [elementValue];
				highlightedNode = root;
				highlightedKeys = [{node: root, keyIndex: 0}];
				await updateTreeLayout();
				await delay(longDelay());
				algorithmStatus.set('finished');
				animationInProgress = false;
				elementValue = undefined;
				algorithmStatus.set('idle');
				activeLine.set({ start: -1, end: -1 });
				return;
			}

			await insert(root, elementValue);
			elementValue = undefined;
			await updateTreeLayout();
			setTimeout(() => {
				highlightedNode = null;
				highlightedKeys = [];
				animationPath = [];
			}, 1000);
			algorithmStatus.set('finished');
			animationInProgress = false;
			algorithmStatus.set('idle');
			activeLine.set({ start: -1, end: -1 });
		});
	}

	async function insert(node, key) {
		selectedAlgorithmSourceCode.set(btreeInsertSource);
		activeLine.set({ start: 3, end: 3 });
		if (node.keys.length === maxElements) {
			log(`Root is full, splitting occurs`);
			highlightedNode = node;
			await delay(longDelay());
			await pauseIfNeeded(activeRunId);
			const newRoot = createNode();
			newRoot.children.push(node);
			newRoot.isLeaf = false;
			root = newRoot;
			await updateTreeLayout();
			await delay(500 - get(speed) * 4);
			activeLine.set({ start: 7, end: 7 });
			await splitChild(newRoot, 0);
			await updateTreeLayout();
			activeLine.set({ start: 8, end: 8 });
			await insertNonFull(newRoot, key);
		} else {
			activeLine.set({ start: 11, end: 13 });
			await insertNonFull(node, key);
		}
	}

	async function searchElement() {
		if (!validateInput() || animationInProgress) return;
		await withRunGuard(async () => {
			animationInProgress = true;
			consoleLog.set([]);
			currentStep.set(0);
			algorithmStatus.set('running');
			lastOperation = 'search';
			selectedAlgorithmSourceCode.set(btreeSearchSource);
			activeLine.set({ start: 0, end: 0 });
			log(`Starting search: ${elementValue}`);

			animationPath = [];
			highlightedKeys = [];
			await bTreeSearch(root, elementValue);
			log('Search complete!');

			setTimeout(() => {
				highlightedNode = null;
				highlightedKeys = [];
				animationPath = [];
			}, 1000);
			algorithmStatus.set('finished');
			animationInProgress = false;
			algorithmStatus.set('idle');
			activeLine.set({ start: -1, end: -1 });
		});
	}

	async function bTreeSearch(node, value) {
		selectedAlgorithmSourceCode.set(btreeSearchSource);
		activeLine.set({ start: 2, end: 2 });
		if (!node) return false;
		highlightedKeys = [];

		animationPath.push(node);
		await updateTreeLayout();
		highlightedNode = node;
		log(`Examined node: ${node.keys.join(', ')}`);
		await delay(longDelay());
		await pauseIfNeeded(activeRunId);
		let i = 0;
		while (i < node.keys.length && value > node.keys[i]) {
			activeLine.set({ start: 4, end: 4 });
			highlightedKeys = [{node: node, keyIndex: i}];
			await delay(shortDelay());
			i++;
		}
		if (i < node.keys.length && value === node.keys[i]) {
			activeLine.set({ start: 6, end: 8 });
			highlightedKeys = [{node: node, keyIndex: i}];
			log(`Value found: ${value}`);
			await delay(longDelay());
			return true;
		}
		if (node.isLeaf) {
			activeLine.set({ start: 9, end: 11 });
			log(`Value ${value} not found - leaf level.`);
			await delay(longDelay());
			return false;
		}
		activeLine.set({ start: 12, end: 12 });
		log(`Moving to child ${i}...`);
		await delay(midDelay());
		return await bTreeSearch(node.children[i], value);
	}

	async function insertNonFull(node, key) {
		selectedAlgorithmSourceCode.set(btreeInsertSource);
		activeLine.set({ start: 16, end: 16 });
		log(`Inserting into node: [${node.keys.join(', ')}]`);
		await delay(longDelay());
		await pauseIfNeeded(activeRunId);
		let i = node.keys.length - 1;
		if (node.isLeaf) {
			activeLine.set({ start: 17, end: 18 });
			log(`Inserting into leaf node`);
			node.keys.push(0);
			await updateTreeLayout();
			while (i >= 0 && key < node.keys[i]) {
				activeLine.set({ start: 20, end: 23 });
				highlightedKeys = [{node: node, keyIndex: i}];
				await delay(shortDelay());
				node.keys[i + 1] = node.keys[i];
				i--;
			}
			node.keys[i + 1] = key;
			highlightedKeys = [{node: node, keyIndex: i + 1}];
			insertKeyHighlight = null;
			await tick();
			await delay(longDelay());
			await pauseIfNeeded(activeRunId);
		} else {
			activeLine.set({ start: 26, end: 27 });
			while (i >= 0 && key < node.keys[i]) {
				activeLine.set({ start: 28, end: 28});
				highlightedKeys = [{node: node, keyIndex: i}];
				await delay(shortDelay());
				i--;
			}
			i++;
			activeLine.set({ start: 31, end: 31 });
			if (node.children[i].keys.length === maxElements) {
				await splitChild(node, i);
				if (key > node.keys[i]) i++;
			}
			activeLine.set({ start: 36, end: 36 });
			await insertNonFull(node.children[i], key);
		}
	}

	async function splitChild(parent, index) {
		const t = Math.floor(maxElements / 2);
		const fullChild = parent.children[index];
		
		splitAnimation = {
			parent: parent,
			childIndex: index,
			fullChild: {...fullChild}
		};
		
		log(`Splitting node: [${fullChild.keys.join(', ')}]`);
		
		highlightedNode = fullChild;
		highlightedKeys = [];
		await delay(longDelay());
		
		const newNode = createNode();
		newNode.new = true;
		newNode.isLeaf = fullChild.isLeaf;
		const middleKey = fullChild.keys[t];
		
		highlightedKeys = [{node: fullChild, keyIndex: t}];
		await delay(longDelay());
		
		newNode.keys = fullChild.keys.slice(t + 1);
		if (!fullChild.isLeaf) {
			newNode.children = fullChild.children.slice(t + 1);
		}
		fullChild.keys = fullChild.keys.slice(0, t);
		if (!fullChild.isLeaf) {
			fullChild.children = fullChild.children.slice(0, t + 1);
		}
		
		parent.children.splice(index + 1, 0, newNode);
		parent.keys.splice(index, 0, middleKey);
		
		highlightedNode = parent;
		highlightedKeys = [{node: parent, keyIndex: index}]; 

		await updateTreeLayout();
		await delay(longDelay());
		await pauseIfNeeded(activeRunId);
		
		splitAnimation = null;
	}

	async function updateTreeLayout() {
		updateCanvasSize();
		svgNodes = [];
		calculateTreeLayout(root);
		await tick();
		requestAnimationFrame(() => {
			svgNodes.forEach(({ node }) => (node.new = false));
		});
	}

	function calculateTreeLayout(root) {
		const NODE_WIDTH = 70;
		const NODE_GAP = 30;
		const LEVEL_HEIGHT = 120;

		function calculateNodeWidth(node) {
			if (!node) return 0;
			if (node.isLeaf) {
				return Math.max(node.keys.length * NODE_WIDTH, NODE_WIDTH);
			}
			let width = 0;
			for (let child of node.children) {
				width += calculateNodeWidth(child) + NODE_GAP;
			}
			width -= NODE_GAP;
			return Math.max(width, node.keys.length * NODE_WIDTH);
		}

		function positionNodes(node, depth = 0, xStart = 0) {
			if (!node) return { width: 0, center: 0 };
			if (node.isLeaf) {
				const width = Math.max(node.keys.length * NODE_WIDTH, NODE_WIDTH);
				node.prevX = node.x || xStart + width / 2;
				node.prevY = node.y || depth * LEVEL_HEIGHT + 60;
				node.x = xStart + width / 2;
				node.y = depth * LEVEL_HEIGHT + 60;
				svgNodes.push({ node });
				return { width, center: node.x };
			}
			let currentX = xStart;
			const childCenters = [];
			let totalWidth = 0;
			for (let i = 0; i < node.children.length; i++) {
				const child = node.children[i];
				const childResult = positionNodes(child, depth + 1, currentX);
				childCenters.push(childResult.center);
				currentX += childResult.width + NODE_GAP;
				totalWidth += childResult.width + NODE_GAP;
			}
			totalWidth -= NODE_GAP;
			const nodeWidth = Math.max(totalWidth, node.keys.length * NODE_WIDTH);
			const leftMost = childCenters.length > 0 ? childCenters[0] : xStart + nodeWidth / 2;
			const rightMost = childCenters.length > 0 ? childCenters[childCenters.length - 1] : xStart + nodeWidth / 2;
			node.prevX = node.x || (leftMost + rightMost) / 2;
			node.prevY = node.y || depth * LEVEL_HEIGHT + 60;
			node.x = (leftMost + rightMost) / 2;
			node.y = depth * LEVEL_HEIGHT + 60;
			svgNodes.push({ node });
			return { width: nodeWidth, center: node.x };
		}

		const totalWidth = calculateNodeWidth(root);
		const xOffset = (svgWidth - totalWidth) / 2;
		positionNodes(root, 0, xOffset);
	}

	function getNodeColor(node) {
		if (node === highlightedNode) {
			return "#ffcc00";
		}
		return "#4CAF50";
	}

	function getNodeGradient(node) {
		if (node === highlightedNode) {
			return "url(#highlightGradient)";
		}
		if (node.new) {
			return "url(#newNodeGradient)";
		}
		return "url(#nodeGradient)";
	}

	function isKeyHighlighted(node, keyIndex) {
		return highlightedKeys.some(k => k.node === node && k.keyIndex === keyIndex);
	}

	function isNodeInPath(node) {
		return animationPath.includes(node);
	}

	function handleMaxElementsChange(e) {
		maxElements = +e.target.value;
		resetTree();
		consoleLog.set([]);
		log(`B-tree maximum elements changed to: ${maxElements}. New tree created.`);
	}

	async function deleteElement() {
		if (!validateInput() || animationInProgress) return;
		await withRunGuard(async () => {
			animationInProgress = true;
			log(`Delete: ${elementValue}`);
			algorithmStatus.set('running');
			consoleLog.set([]);
			currentStep.set(0);
			lastOperation = 'delete';
			selectedAlgorithmSourceCode.set(btreeDeleteSource);
			activeLine.set({ start: 1, end: 1 });
			if (!root) {
				log('The tree is empty.');
				animationInProgress = false;
				algorithmStatus.set('idle');
				activeLine.set({ start: -1, end: -1 });
				return;
			}
			await bTreeDelete(root, elementValue);
			await updateTreeLayout();
			log('Delete complete!');
			setTimeout(() => {
				highlightedNode = null;
				highlightedKeys = [];
				animationPath = [];
			}, 1000);
			algorithmStatus.set('finished');
			animationInProgress = false;
			algorithmStatus.set('idle');
			activeLine.set({ start: -1, end: -1 });
		});
	}

	async function bTreeDelete(node, key) {
		selectedAlgorithmSourceCode.set(btreeDeleteSource);
		if (!node) return;
		highlightedKeys = [];
		animationPath.push(node);
		highlightedNode = node;
		let idx = 0;
		while (idx < node.keys.length && key > node.keys[idx]) {
			highlightedKeys = [{node, keyIndex: idx}];
			await delay(shortDelay());
			idx++;
		}
		if (idx < node.keys.length && node.keys[idx] === key) {
			if (node.isLeaf) {
				log(`Deleting key from leaf: ${key}`);
				highlightedNode = node;
				highlightedKeys = [{node, keyIndex: idx}];
				await delay(longDelay());
				node.keys.splice(idx, 1);
				await updateTreeLayout();
				return;
			} else {
				log(`Deleting key from internal node: ${key}`);
				highlightedNode = node;
				highlightedKeys = [{node, keyIndex: idx}];
				await delay(longDelay());
				if (node.children[idx].keys.length >= Math.ceil(maxElements / 2)) {
					let pred = node.children[idx];
					while (!pred.isLeaf) {
						highlightedNode = pred;
						await updateTreeLayout();
							await delay(shortDelay());
						pred = pred.children[pred.keys.length];
					}
					const predKey = pred.keys[pred.keys.length - 1];
					log(`Replacing with predecessor key: ${predKey}`);
					node.keys[idx] = predKey;
					await bTreeDelete(node.children[idx], predKey);
				} else if (node.children[idx + 1].keys.length >= Math.ceil(maxElements / 2)) {
					let succ = node.children[idx + 1];
					while (!succ.isLeaf) {
						highlightedNode = succ;
						await updateTreeLayout();
							await delay(shortDelay());
						succ = succ.children[0];
					}
					const succKey = succ.keys[0];
					log(`Replacing with successor key: ${succKey}`);
					node.keys[idx] = succKey;
					await bTreeDelete(node.children[idx + 1], succKey);
				} else {
					log('Merge required');
					await mergeChildren(node, idx);
					await bTreeDelete(node.children[idx], key);
				}
				await updateTreeLayout();
				return;
			}
		}
		if (node.isLeaf) {
			log('Key not found in tree.');
			await delay(longDelay());
			return;
		}
		log(`Moving to child ${idx}...`);
		let child = node.children[idx];
		if (child.keys.length < Math.ceil(maxElements / 2)) {
			await fixChild(node, idx);
			child = node.children[idx];
		}
		await bTreeDelete(child, key);
		await updateTreeLayout();
	}

	async function mergeChildren(node, idx) {
		const child = node.children[idx];
		const sibling = node.children[idx + 1];
		log(`Merging: ${child.keys.join(', ')} + ${node.keys[idx]} + ${sibling.keys.join(', ')}`);
		
		child.keys.push(node.keys[idx], ...sibling.keys);
		if (!child.isLeaf) child.children.push(...sibling.children);
		node.keys.splice(idx, 1);
		node.children.splice(idx + 1, 1);
		
		await updateTreeLayout();
	}

	async function fixChild(node, idx) {
		const minKeys = Math.ceil(maxElements / 2);
		let child = node.children[idx];
		
		if (idx > 0 && node.children[idx - 1].keys.length >= minKeys) {
			const left = node.children[idx - 1];
			log('Borrowing from left');
			
			child.keys.unshift(node.keys[idx - 1]);
			node.keys[idx - 1] = left.keys.pop();
			if (!left.isLeaf) child.children.unshift(left.children.pop());
			
			await updateTreeLayout();
		} else if (idx < node.children.length - 1 && node.children[idx + 1].keys.length >= minKeys) {
			const right = node.children[idx + 1];
			log('Borrowing from right');
			
			child.keys.push(node.keys[idx]);
			node.keys[idx] = right.keys.shift();
			if (!right.isLeaf) child.children.push(right.children.shift());
			
			await updateTreeLayout();
		} else {
			log('Merge required');
			
			if (idx < node.children.length - 1) {
				await mergeChildren(node, idx);
			} else {
				await mergeChildren(node, idx - 1);
			}
		}
	}
</script>

<div class="visualizer-container">
	<div class="control-buttons">
		<input
			class="custom-input"
			type="number"
			bind:value={elementValue}
			placeholder="Node's value"
			disabled={$algorithmStatus !== 'idle'}
		/>
		<div class="degree-container">
			<label for="max-elements-select">Max Width:</label>
			<select
				id="max-elements-select"
				value={maxElements}
				on:change={handleMaxElementsChange}
				disabled={$algorithmStatus !== 'idle'}
				class="degree-select"
			>
				{#each [2, 3, 4, 5] as elements}
					<option value={elements}>{elements}</option>
				{/each}
			</select>
		</div>
		<div>
			<button on:click={insertElement} disabled={$algorithmStatus !== 'idle'}>Insert Node</button>
			<button on:click={searchElement} disabled={$algorithmStatus !== 'idle'}>Search Node</button>
			<button on:click={deleteElement} disabled={$algorithmStatus !== 'idle'}>Delete Node</button>
			<button on:click={createSampleTree} disabled={$algorithmStatus !== 'idle'} class="special-button">Sample Tree</button>
			<button on:click={resetTree} disabled={$algorithmStatus !== 'idle'} class="clear-button">Delete Tree</button>

		</div>
	</div>

	<div class="canvas-container">
		<svg class="svg" width={svgWidth} height={svgHeight}>
			<defs>
				<linearGradient id="nodeGradient" x1="0%" y1="0%" x2="0%" y2="100%">
					<stop offset="0%" stop-color="#2f4f4f" />
					<stop offset="100%" stop-color="#254040" />
				</linearGradient>
				<linearGradient id="highlightGradient" x1="0%" y1="0%" x2="0%" y2="100%">
					<stop offset="0%" stop-color="#ffd700" />
					<stop offset="100%" stop-color="#d8b400" />
				</linearGradient>
				<linearGradient id="newNodeGradient" x1="0%" y1="0%" x2="0%" y2="100%">
					<stop offset="0%" stop-color="#32cd32" />
					<stop offset="100%" stop-color="#2a9d2a" />
				</linearGradient>
				<linearGradient id="pathGradient" x1="0%" y1="0%" x2="0%" y2="100%">
					<stop offset="0%" stop-color="#ffd700" />
					<stop offset="100%" stop-color="#d8b400" />
				</linearGradient>
				<filter id="dropShadow" x="-20%" y="-20%" width="140%" height="140%">
					<feGaussianBlur in="SourceAlpha" stdDeviation="3"/>
					<feOffset dx="2" dy="2" result="offsetblur"/>
					<feComponentTransfer>
						<feFuncA type="linear" slope="0.5"/>
					</feComponentTransfer>
					<feMerge> 
						<feMergeNode/>
						<feMergeNode in="SourceGraphic"/> 
					</feMerge>
				</filter>
				<marker id="arrowHead" viewBox="0 0 10 10" refX="5" refY="5"
						markerWidth="6" markerHeight="6" orient="auto-start-reverse">
					<path d="M 0 0 L 10 5 L 0 10 z" fill="#ffd700"/>
				</marker>
			</defs>
			
			{#if lastOperation === 'search' && animationPath.length > 1}
				{#each animationPath as node, i}
					{#if i < animationPath.length - 1}
						<line
							x1={node.x}
							y1={node.y + 25}
							x2={animationPath[i+1].x}
							y2={animationPath[i+1].y - 25}
							stroke="#ffd700"
							stroke-width="3"
							stroke-dasharray="5,3"
							marker-end="url(#arrowHead)"
							opacity="0.7"
						/>
					{/if}
				{/each}
			{/if}
			
			{#if root}
				{#each svgNodes as { node }}
					{#if !node.isLeaf}
						{#each node.children as child}
							<line
								x1={node.x}
								y1={node.y + 25}
								x2={child.x}
								y2={child.y - 25}
								stroke={isNodeInPath(node) && isNodeInPath(child) ? "#ffd700" : "#2f4f4f"}
								stroke-width={isNodeInPath(node) && isNodeInPath(child) ? "3" : "2"}
								opacity={isNodeInPath(node) && isNodeInPath(child) ? "0.85" : "0.6"}
							/>
						{/each}
					{/if}
				{/each}
				
				{#each svgNodes as { node } (node.id)}
					<g 
						class="btree-node"
						class:node-animated={node === highlightedNode || node.new}
						class:node-new={node.new}
						style={`--btree-anim:${animationMs}ms; transform: translate(${node.x}px, ${node.y}px); transition: transform ${animationMs}ms cubic-bezier(0.22, 0.68, 0, 1);`}
					>
						<rect
							x={-(node.keys.length * 27.5 + 20) / 2}
							y="-25"
							width={node.keys.length * 27.5 + 20}
							height="50"
							rx="10"
							fill={node.new ? "url(#newNodeGradient)" : (node === highlightedNode ? "url(#highlightGradient)" : "url(#nodeGradient)")}
							filter="url(#dropShadow)"
							class:pulse-animation={node.new}
						/>
						
						{#each node.keys as key, i}
							<g class:key-highlight={isKeyHighlighted(node, i) || (insertKeyHighlight === key && lastOperation === 'insert')}>
								{#if isKeyHighlighted(node, i) || (insertKeyHighlight === key && lastOperation === 'insert')}
									<circle
										cx={(i - (node.keys.length - 1) / 2) * 27.5}
										cy="0"
										r="15"
										fill="#ffd700"
										opacity="0.6"
										class="pulse-circle"
									/>
								{/if}
								<text
									x={(i - (node.keys.length - 1) / 2) * 27.5}
									y="6"
									text-anchor="middle"
									font-size="16"
									font-weight="bold"
									fill={isKeyHighlighted(node, i) || (insertKeyHighlight === key && lastOperation === 'insert') ? "black" : "aliceblue"}
									style="paint-order: stroke; stroke: white; stroke-width: 2px;"
								>
									{key}
								</text>
							</g>
						{/each}
					</g>
				{/each}
				
				{#if splitAnimation}
					<g opacity="0.7">
						<path
							d="M {splitAnimation.fullChild.x},{splitAnimation.fullChild.y} 
							   Q {splitAnimation.fullChild.x},{splitAnimation.parent.y - 20} 
							     {splitAnimation.parent.x},{splitAnimation.parent.y}"
							fill="none"
							stroke="#ff9800"
							stroke-width="2"
							stroke-dasharray="5,3"
							marker-end="url(#arrowHead)"
						/>
					</g>
				{/if}
			{/if}
		</svg>
	</div>
</div>

<style>
	.visualizer-container {
		width: 100%;
		max-width: 1000px;
		margin: 0 auto;
	}
	
	.canvas-container {
		position: relative;
		margin-top: 20px;
		display: flex;
		justify-content: center;
	}
	
	.control-buttons {
		display: flex;
		justify-content: space-around;
		align-items: center;
		padding: 1rem;
		border-bottom: 3px solid #505050;
		flex-wrap: wrap;
		gap: 10px;
	}
	
	.control-buttons input {
		width: 150px;
		padding: 0.5rem;
		margin-right: 10px;
		border-radius: 5px;
		background-color: #2f2f2f;
		border: 3px solid #505050;
		color: white;
	}

	.control-buttons div button {
		padding: 0.5rem 1rem;
		background-color: #505050;
		color: white;
		border: none;
		border-radius: 5px;
		cursor: pointer;
		transition: background-color 0.3s;
	}

	.control-buttons div button:hover:not([disabled]) {
		background-color: #45a049;
	}
	
	.control-buttons div button[disabled] {
		opacity: 0.5;
		cursor: not-allowed;
	}
	.control-buttons div {
		display: flex;
		gap: 10px;
		flex-wrap: wrap;
		justify-content: center;
	}
	.custom-input:disabled {
		opacity: 0.6;
		cursor: not-allowed;
		border-color: #3a3a3a;
	}

	.control-buttons button:hover {
		background-color: #45a049;
	}
	
	.special-button {
		background-color: #3498db !important;
	}
	
	.special-button:hover {
		background-color: #2980b9 !important;
	}
	
	.clear-button {
		background-color: #e74c3c !important;
	}
	
	.clear-button:hover {
		background-color: #c0392b !important;
	}
	
	.svg {
		margin: 1rem auto;
		border: 1px solid #505050;
		border-radius: 4px;
		display: block;
		box-shadow: 0 6px 18px rgba(0, 0, 0, 0.35);
	}
	
	.degree-container {
		display: flex;
		align-items: center;
		background-color: #333;
		padding: 0.5rem;
		border-radius: 5px;
		margin-left: 0.5rem;
		border: 1px solid #555;
	}
	
	.degree-container label {
		color: white;
		margin-right: 0.5rem;
	}
	
	.degree-select {
		background-color: #505050;
		color: white;
		border: 1px solid #757575;
		border-radius: 4px;
		padding: 0.3rem;
		font-size: 0.9rem;
	}
	
	.degree-select option {
		background-color: #333;
		color: white;
	}
	
	.btree-node {
		transform-origin: center;
		transition: transform var(--btree-anim, 450ms) cubic-bezier(0.22, 0.68, 0, 1);
	}

	.node-animated rect {
		filter: url(#dropShadow);
	}

	.node-new rect {
		animation: pop-in 260ms ease-out;
	}

	.key-highlight .pulse-circle {
		animation: pulse 900ms ease-in-out infinite;
	}

	.key-highlight text {
		transform: scale(1.06);
		transition: transform 150ms ease;
	}

	line {
		transition: stroke 200ms ease, stroke-width 200ms ease, opacity 200ms ease;
	}

	@keyframes pulse {
		0% { transform: scale(0.95); opacity: 0.7; }
		50% { transform: scale(1.05); opacity: 1; }
		100% { transform: scale(0.95); opacity: 0.7; }
	}

	@keyframes pop-in {
		0% { transform: scale(0.8); opacity: 0; }
		100% { transform: scale(1); opacity: 1; }
	}
</style>