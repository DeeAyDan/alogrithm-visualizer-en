<script lang="ts">
	// @ts-nocheck
import { onMount } from 'svelte';
import { spring } from 'svelte/motion';
import { fade, fly } from 'svelte/transition';
import {
	currentStep,
	totalSteps,
	consoleLog,
	speed,
	algorithmStatus,
	resumeSignal,
	selectedAlgorithmSourceCode,
	activeLine
} from '../stores/store.svelte.js';
import Controls from '../routes/Controls.svelte';
import { get } from 'svelte/store';
import { flip } from 'svelte/animate';
import { waitUntilResume, delay, pauseIfNeeded, log } from '../stores/utils.js';

interface Node {
	value: number;
	left?: Node;
	right?: Node;
	height: number;
	x: number;
	y: number;
	id?: string;
	prevX?: number;
	prevY?: number;
}

const LAYOUT = {
	startX: 250,
	startY: 25,
	levelGapY: 80,
	nodeRadius: 18,
	maxDepth: 4
};

const ANIMATION = {
	delay: 800,
	duration: 600,
	easing: 'cubic-bezier(0.4, 0.0, 0.2, 1)'
};

let root: Node | null = null;
let highlightedNode: Node | null = null;
let elementValue: number | null = null;
let animating = false;
let nodes: Node[] = [];
activeLine.set({ start: -1, end: -1 });

function initStores(): void {
	currentStep.set(0);
	algorithmStatus.set('idle');
	consoleLog.set([]);
	activeLine.set({ start: -1, end: -1 });
	selectedAlgorithmSourceCode.set('Choose an operation!');
}

onMount(initStores);

function validateInput(): boolean {
	if (elementValue === null || elementValue === undefined) {
		log('Please enter a value!');
		return false;
	} else if (typeof elementValue !== 'number') {
		log('Please enter a number!');
		return false;
	} else if (elementValue < 0) {
		log('Please enter a positive number!');
		return false;
	} else if (elementValue > 100) {
		log('Please enter a number between 0 and 100!');
		return false;
	}
	return true;
}

function getAllNodes(node: Node | null): Node[] {
	const result: Node[] = [];
	function traverse(n: Node | null): void {
		if (!n) return;
		result.push(n);
		traverse(n.left);
		traverse(n.right);
	}
	traverse(node);
	return result;
}

function getOffsetX(depth: number): number {
	return 400 / Math.pow(2, depth + 1);
}

function saveCurrentPositions(node: Node | null): void {
	if (!node) return;
	node.prevX = node.x;
	node.prevY = node.y;
	saveCurrentPositions(node.left);
	saveCurrentPositions(node.right);
}

function calculatePositions(
	node: Node | null,
	depth = 0,
	offset = 0,
	parentX = LAYOUT.startX
): void {
	if (!node) return;

	const xOffset = getOffsetX(depth);
	node.x = parentX + offset * xOffset;
	node.y = LAYOUT.startY + depth * LAYOUT.levelGapY;

	calculatePositions(node.left, depth + 1, -1, node.x);
	calculatePositions(node.right, depth + 1, 1, node.x);
}

async function animateNodeMovements(): Promise<void> {
	if (!root) return;
	
	animating = true;
	
	await delay(ANIMATION.duration);
	await pauseIfNeeded();
	
	const allNodes = getAllNodes(root);
	allNodes.forEach((node) => {
		node.prevX = undefined;
		node.prevY = undefined;
	});
	
	nodes = [...allNodes];
	
	animating = false;
}

function updateInsertSourceCode(): void {
	selectedAlgorithmSourceCode.set(
		`function insertElement(tree, value) {
  const insert = (node, value, depth = 0) => {
    if (depth > ${LAYOUT.maxDepth - 1}) {
      return node;
    }

    if (!node) {
      return { value, height: 1, left: null, right: null, isNew: true };
    }

    if (value < node.value) {
      node.left = insert(node.left, value, depth + 1);
    }
    else if (value > node.value) {
      node.right = insert(node.right, value, depth + 1);
    }

  return insert(tree, value);
}`
	);
}

function updateSearchSourceCode(): void {
	selectedAlgorithmSourceCode.set(
		`function searchElement(tree, value) {
  const search = (node, value) => {
    if (!node) {
      return false;
    }
    
    if (value === node.value) {
      return true;
    } 
    else if (value < node.value) {
      return search(node.left, value);
    } 
    else {
      return search(node.right, value);
    }
  };

  return search(tree, value);
}`
	);
}

function updateDeleteSourceCode(): void {
	selectedAlgorithmSourceCode.set(
		`function deleteElement(tree, value) {
  const findMin = (node) => {
    let current = node;
    while (current.left) {
      current = current.left;
    }
    return current;
  };

  const remove = (node, value) => {
    if (!node) {
      return null;
    }

    if (value < node.value) {
      node.left = remove(node.left, value);
    }
    else if (value > node.value) {
      node.right = remove(node.right, value);
    } 
    else {
      if (!node.left && !node.right) {
        return null;
      }
      else if (!node.left) {
        return node.right;
      }
      else if (!node.right) {
        return node.left;
      }
      else {
        const successor = findMin(node.right);
        node.value = successor.value;
        node.right = remove(node.right, successor.value);
      }
    }

  return remove(tree, value);
}`
	);
}

async function insertElement(): Promise<void> {
    if (!validateInput() || animating) return;

    algorithmStatus.set('running');
    consoleLog.set([]);
    currentStep.set(0);
    log(`Insert: ${elementValue}`);
    updateInsertSourceCode();

    const insert = async (node: Node | null, value: number, depth = 0): Promise<Node> => {
        if (depth > LAYOUT.maxDepth - 1) {
            activeLine.set({ start: 3, end: 5 });
            log(`Tree depth cannot be greater than ${LAYOUT.maxDepth}.`);
            await delay(ANIMATION.delay);
            await pauseIfNeeded();
            return node;
        }

        await delay(ANIMATION.delay);
        await pauseIfNeeded();

        if (!node) {
            activeLine.set({ start: 7, end: 9 });
            log(`Inserted value: ${value}`);
            await delay(ANIMATION.delay);
            await pauseIfNeeded();
            return { value, height: 1, x: LAYOUT.startX, y: LAYOUT.startY };
        }

        highlightedNode = node;
        log(`Examined node: ${node.value}`);

        if (value < node.value) {
            activeLine.set({ start: 11, end: 13 });
            log(`${value} < ${node.value}, going left`);
            node.left = await insert(node.left, value, depth + 1);
        } else if (value > node.value) {
            activeLine.set({ start: 14, end: 16 });
            log(`${value} > ${node.value}, going right`);
            node.right = await insert(node.right, value, depth + 1);
        } else {
            activeLine.set({ start: 17, end: 19 });
            log('The value already exists in the tree.');
        }

        node.height = Math.max(height(node.left), height(node.right)) + 1;
        return node;
    };

    root = await insert(root, elementValue);
    
    saveCurrentPositions(root);
    calculatePositions(root);
    await animateNodeMovements();
    
    await delay(ANIMATION.delay);
    await pauseIfNeeded();
    
    highlightedNode = null;

    const checkAndBalance = async (node: Node | null): Promise<Node | null> => {
        if (!node) return null;
        
        node.left = await checkAndBalance(node.left);
        node.right = await checkAndBalance(node.right);
        
        const bf = balanceFactor(node);
        if (bf > 1) {
            const leftBf = balanceFactor(node.left!);
            if (leftBf < 0) {
                log(`Left-Right case: ${node.value}`);
                
                node.left = await rotateLeft(node.left!);
                
                saveCurrentPositions(root);
                calculatePositions(root);
                await animateNodeMovements();
                
                await delay(ANIMATION.delay);
                await pauseIfNeeded();
                
                const balanced = await rotateRight(node);
                saveCurrentPositions(root);
                calculatePositions(root);
                await animateNodeMovements();
                await delay(ANIMATION.delay);
                await pauseIfNeeded();
                return balanced;
            } else {
                log(`Left-Left case: ${node.value}`);
                
                const balanced = await rotateRight(node);
                
                return balanced;
            }
        } else if (bf < -1) {
            const rightBf = balanceFactor(node.right!);
            if (rightBf > 0) {
                log(`Right-Left case: ${node.value}`);
                
                node.right = await rotateRight(node.right!);
                
                saveCurrentPositions(root);
                calculatePositions(root);
                await animateNodeMovements();
                
                await delay(ANIMATION.delay);
                await pauseIfNeeded();
                
                const balanced = await rotateLeft(node);
                saveCurrentPositions(root);
                calculatePositions(root);
                await animateNodeMovements();
                await delay(ANIMATION.delay);
                await pauseIfNeeded();
                return balanced;
            } else {
                log(`Right-Right case: ${node.value}`);
                
                const balanced = await rotateLeft(node);
                
                return balanced;
            }
        }
        
        return node;
    };
    
    root = await checkAndBalance(root);

    saveCurrentPositions(root);
    calculatePositions(root);
    await animateNodeMovements();
    
    activeLine.set({ start: -1, end: -1 });
    algorithmStatus.set('idle');
}

async function searchElement(): Promise<void> {
	if (!validateInput() || animating) return;

	algorithmStatus.set('running');
	consoleLog.set([]);
	currentStep.set(0);
	log(`Search: ${elementValue}`);
	updateSearchSourceCode();

	const search = async (node: Node | null, value: number): Promise<boolean> => {
		if (!node) {
			activeLine.set({ start: 3, end: 5 });
			log('Element not found.');
			return false;
		}

		highlightedNode = node;
		log(`Examined node: ${node.value}`);
		await delay(ANIMATION.delay);
		await pauseIfNeeded();

		if (value === node.value) {
			activeLine.set({ start: 7, end: 9 });
			log(`Value found: ${value}`);
			await delay(ANIMATION.delay);
			await pauseIfNeeded();
			return true;
		}
		else if (value < node.value) {
			activeLine.set({ start: 10, end: 12 });
			log(`${value} < ${node.value}, searching left`);
			return await search(node.left, value);
		}
		else {
			activeLine.set({ start: 13, end: 15 });
			log(`${value} > ${node.value}, searching right`);
			return await search(node.right, value);
		}
	};

	await search(root, elementValue);
	highlightedNode = null;
	activeLine.set({ start: -1, end: -1 });
	algorithmStatus.set('idle');
}

async function deleteElement(): Promise<void> {
	if (!validateInput() || animating) return;

	algorithmStatus.set('running');
	consoleLog.set([]);
	log(`Delete: ${elementValue}`);
	updateDeleteSourceCode();

	await delay(ANIMATION.delay);
	await pauseIfNeeded();

	const findMin = async (node: Node): Promise<Node> => {
		let current = node;
		highlightedNode = current;
		log(`Finding minimum: ${current.value}`);
		await delay(ANIMATION.delay);
		await pauseIfNeeded();
		
		while (current.left) {
			current = current.left;
			highlightedNode = current;
			log(`Finding minimum: ${current.value}`);
			await delay(ANIMATION.delay);
			await pauseIfNeeded();
		}
		return current;
	};

	if (root) {
		saveCurrentPositions(root);
		calculatePositions(root);
	}

	const remove = async (node: Node | null, value: number): Promise<Node | null> => {
		if (!node) {
			activeLine.set({ start: 11, end: 13 });
			log('Element not found.');
			return null;
		}

		highlightedNode = node;
		log(`Examined node: ${node.value}`);
		await delay(ANIMATION.delay);
		await pauseIfNeeded();

		if (value < node.value) {
			activeLine.set({ start: 15, end: 17 });
			log(`${value} < ${node.value}, searching left`);
			node.left = await remove(node.left, value);
		} else if (value > node.value) {
			activeLine.set({ start: 18, end: 20 });
			log(`${value} > ${node.value}, searching right`);
			node.right = await remove(node.right, value);
		} else {
			if (!node.left && !node.right) {
				activeLine.set({ start: 22, end: 24 });
				log(`Leaf node deleted: ${value}`);
				return null;
			} else if (!node.left) {
				activeLine.set({ start: 25, end: 27 });
				log(`Node deleted (${value}), right child replaces it`);

				if (node.right) {
					node.right.prevX = node.right.x;
					node.right.prevY = node.right.y;
				}

				return node.right;
			} else if (!node.right) {
				activeLine.set({ start: 28, end: 30 });
				log(`Node deleted (${value}), left child replaces it`);

				if (node.left) {
					node.left.prevX = node.left.x;
					node.left.prevY = node.left.y;
				}

				return node.left;
			} else {
				activeLine.set({ start: 31, end: 35 });
				const successor = await findMin(node.right);
				log(`Replacing node with two children (${value}) with value ${successor.value}`);

				const oldValue = node.value;
				node.value = successor.value;
				
				await delay(ANIMATION.delay);
				await pauseIfNeeded();
				
				log(`Value changed: ${oldValue} -> ${successor.value}`);
				node.right = await remove(node.right, successor.value);
			}
		}

		node.height = Math.max(height(node.left), height(node.right)) + 1;
		
		const bf = balanceFactor(node);
		if (bf > 1 || bf < -1) {
			log(`Balancing required: ${node.value}`);
            
            saveCurrentPositions(root);
            
            const balanced = await balance(node);
            
            return balanced;
		}
		
		return node;
	};

	root = await remove(root, elementValue);
	
	if (root) {
		saveCurrentPositions(root);
		calculatePositions(root);
		await animateNodeMovements();
	}

	highlightedNode = null;
	activeLine.set({ start: -1, end: -1 });
	algorithmStatus.set('idle');
}

function resetTree(): void {
	root = null;
	consoleLog.set([]);
	currentStep.set(0);
	log('Tree deleted.');
	selectedAlgorithmSourceCode.set('Choose an operation!');
}

function createSampleTree(): void {
	root = { value: 50, height: 3, x: 0, y: 0 };
	root.left = { value: 25, height: 2, x: 0, y: 0 };
	root.right = { value: 75, height: 2, x: 0, y: 0 };
	root.left.left = { value: 12, height: 1, x: 0, y: 0 };
	root.left.right = { value: 37, height: 1, x: 0, y: 0 };
	root.right.left = { value: 62, height: 1, x: 0, y: 0 };
	root.right.right = { value: 87, height: 1, x: 0, y: 0 };

	calculatePositions(root);
	consoleLog.set([]);
	currentStep.set(0);
	log('Sample tree created.');
	selectedAlgorithmSourceCode.set('Choose an operation!');
}

function height(n: Node | undefined): number {
	return n ? n.height : 0;
}

function updateHeight(n: Node) {
	n.height = Math.max(height(n.left), height(n.right)) + 1;
}

function balanceFactor(n: Node): number {
	return height(n.left) - height(n.right);
}

function generateNodeId(value: number): string {
	return `node-${value}-${Math.random().toString(36).substr(2, 9)}`;
}

async function rotateLeft(x: Node): Promise<Node> {
    const y = x.right!;
    const t2 = y.left;
    
    log(`Rotate left: ${x.value}`);
    
    y.left = x;
    x.right = t2;
    
    updateHeight(x);
    updateHeight(y);
    
    return y;
}

async function rotateRight(y: Node): Promise<Node> {
    const x = y.left!;
    const t2 = x.right;
    
    log(`Rotate right: ${y.value}`);
    
    x.right = y;
    y.left = t2;
    
    updateHeight(y);
    updateHeight(x);
    
    return x;
}

async function balance(n: Node): Promise<Node> {
    const bf = balanceFactor(n);

    if (bf > 1) {
        if (balanceFactor(n.left!) < 0) {
            log(`Bal-Jobb eset: ${n.value}`);
            await delay(ANIMATION.delay);
            await pauseIfNeeded();
            
            n.left = await rotateLeft(n.left!);
            
            saveCurrentPositions(root);
            calculatePositions(root);
            await animateNodeMovements();
            
            await delay(ANIMATION.delay);
            await pauseIfNeeded();
            
            const balanced = await rotateRight(n);
            saveCurrentPositions(root);
            calculatePositions(root);
            await animateNodeMovements();
            await delay(ANIMATION.delay);
            await pauseIfNeeded();
            return balanced;
        }
        log(`Bal-Bal eset: ${n.value}`);
        await delay(ANIMATION.delay);
        await pauseIfNeeded();
        
        return await rotateRight(n);
    }

    if (bf < -1) {
        if (balanceFactor(n.right!) > 0) {
            log(`Jobb-Bal eset: ${n.value}`);
            await delay(ANIMATION.delay);
            await pauseIfNeeded();
            
            n.right = await rotateRight(n.right!);
            
            saveCurrentPositions(root);
            calculatePositions(root);
            await animateNodeMovements();
            
            await delay(ANIMATION.delay);
            await pauseIfNeeded();
            
            const balanced = await rotateLeft(n);
            saveCurrentPositions(root);
            calculatePositions(root);
            await animateNodeMovements();
            await delay(ANIMATION.delay);
            await pauseIfNeeded();
            return balanced;
        }
        log(`Jobb-Bal eset: ${n.value}`);
        await delay(ANIMATION.delay);
        await pauseIfNeeded();
        
        return await rotateLeft(n);
    }

    return n;
}

</script>

<div class="control-buttons">
	<input
		class="custom-input"
		type="number"
		bind:value={elementValue}
		placeholder="Node's value"
		disabled={$algorithmStatus !== 'idle'}
	/>
	<div>
		<button on:click={insertElement} disabled={$algorithmStatus !== 'idle'}>Insert Node</button>
		<button on:click={deleteElement} disabled={$algorithmStatus !== 'idle'}>Delete Node</button>
		<button on:click={searchElement} disabled={$algorithmStatus !== 'idle'}>Search Node</button>
		<button on:click={createSampleTree} disabled={$algorithmStatus !== 'idle'} class="special-button">Sample Tree</button>
		<button on:click={resetTree} disabled={$algorithmStatus !== 'idle'} class="clear-button">Delete Tree</button>

	</div>
</div>

<div class="algorithm-container">
	<svg class="svg" width="500" height="300">
		{#if root}
			{#each getAllNodes(root) as node (node.id || node.value)}
				{#if node.left}
					<line
						class="edge-line"
						x1={node.x}
						y1={node.y}
						x2={node.left.x}
						y2={node.left.y}
						stroke="#777"
						stroke-width="2"
					/>
				{/if}
				{#if node.right}
					<line
						class="edge-line"
						x1={node.x}
						y1={node.y}
						x2={node.right.x}
						y2={node.right.y}
						stroke="#777"
						stroke-width="2"
					/>
				{/if}
			{/each}
			{#each getAllNodes(root) as node (node.id || node.value)}
				<g
					transform="translate({node.x}, {node.y})"
					class="node-group"
					style="transition: transform 600ms cubic-bezier(0.4, 0.0, 0.2, 1)"
				>
					<circle
						class="node {node === highlightedNode ? 'highlighted' : ''}"
						r={LAYOUT.nodeRadius}
						fill={highlightedNode === node ? '#ffd700' : '#2f4f4f'}
						stroke="aliceblue"
						stroke-width="2"
					/>
					<text
						class="node-text"
						text-anchor="middle"
						dominant-baseline="middle"
						fill={highlightedNode === node ? 'black' : 'aliceblue'}
					>
						{node.value}
					</text>
				</g>
			{/each}
		{:else}
			<text x="250" y="150" text-anchor="middle" fill="#aaa" font-size="16"> The Tree is Empty </text>
		{/if}
	</svg>
</div>

<style>
	.control-buttons {
		display: flex;
		justify-content: space-around;
		align-items: center;
		padding: 1rem;
		border-bottom: 3px solid #505050;
		flex-wrap: wrap;
		gap: 10px;
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
	.control-buttons div {
		display: flex;
		gap: 10px;
		flex-wrap: wrap;
		justify-content: center;
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

	.custom-input:disabled {
		opacity: 0.6;
		cursor: not-allowed;
		border-color: #3a3a3a;
	}

	.algorithm-container {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 1rem;
	}

	.svg {
		margin: 1rem auto;
		border: 1px solid #666;
		border-radius: 4px;
		display: block;
		background-color: #2f2f2f;
	}

	.node-group {
		transition: transform 600ms cubic-bezier(0.4, 0.0, 0.2, 1);
	}

	.node {
		transition: all 600ms cubic-bezier(0.4, 0.0, 0.2, 1);
	}

	.node-text {
		user-select: none;
		pointer-events: none;
		transition: all 600ms cubic-bezier(0.4, 0.0, 0.2, 1);
	}

	.edge-line {
		transition: all 600ms cubic-bezier(0.4, 0.0, 0.2, 1);
	}
</style>