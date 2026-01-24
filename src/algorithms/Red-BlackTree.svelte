<script lang="ts">
	// @ts-nocheck
	import { onMount } from 'svelte';
	import {
		selectedAlgorithmSourceCode,
		currentStep,
		totalSteps,
		consoleLog,
		speed,
		algorithmStatus,
		resumeSignal,
		selectedAlgorithm,
		activeLine
	} from '../stores/store.svelte.js';
	import Controls from '../routes/Controls.svelte';
	import { get } from 'svelte/store';
	import { algorithmDisplayNames } from '../stores/algorithmMap.js';
	import { waitUntilResume, delay, pauseIfNeeded, log } from '../stores/utils.js';

	// ==== Alapadatok ====

	currentStep.set(0);
	algorithmStatus.set('idle');
	consoleLog.set([]);
	activeLine.set({ start: -1, end: -1 });
	const displayName = algorithmDisplayNames[get(selectedAlgorithm)];

	let elementValue;

	const rbtreeInsertSource = `function insert(node, key) {
  const newNode = { value: key, color: 'red' };
  
  if (!node) {
    newNode.color = 'black';
    return newNode;
  }
  
  if (key < node.value) {
    node.left = insert(node.left, key);
  } else if (key > node.value) {
    node.right = insert(node.right, key);
  }
  
  return fixInsert(node, newNode);
}

function fixInsert(node) {
  while (node !== root && node.parent?.color === 'red') {
    const parent = node.parent;
    const grandparent = parent.parent;
    
    if (parent === grandparent.left) {
      const uncle = grandparent.right;
      if (uncle?.color === 'red') {
        parent.color = 'black';
        uncle.color = 'black';
        grandparent.color = 'red';
        node = grandparent;
      } else {
        if (node === parent.right) {
          rotateLeft(parent);
          node = parent;
        }
        parent.color = 'black';
        grandparent.color = 'red';
        rotateRight(grandparent);
      }
    } else {
      // Mirror case
    }
  }
  root.color = 'black';
}`;

	const rbtreeSearchSource = `function search(node, value) {
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
}`;

	const rbtreeDeleteSource = `function delete(node, key) {
  if (!node) return null;
  
  if (key < node.value) {
    node.left = delete(node.left, key);
  } else if (key > node.value) {
    node.right = delete(node.right, key);
  } else {
    // Deletion logic
    if (!node.left) return node.right;
    if (!node.right) return node.left;
    
    const successor = findMin(node.right);
    node.value = successor.value;
    node.right = delete(node.right, successor.value);
  }
  
  return fixDelete(node);
}`;

	type Color = 'red' | 'black';

	class RBNode {
		value: number;
		color: Color;
		left: RBNode | null = null;
		right: RBNode | null = null;
		parent: RBNode | null = null;
		x: number = 25;
		y: number = 25;
		targetX: number = 0;
		targetY: number = 0;
		highlighted: boolean = false;

		constructor(value: number, color: Color = 'red') {
			this.value = value;
			this.color = color;
		}
	}

	let root: RBNode | null = null;
	let animationSpeed = 500;
	let animationInProgress = false;
	
	let animationFrameId: number | null = null;
	let lastTime = 0;
	let nodeAnimations = new Map();

	// ==== Vizualizációs indexek ====

	function validateInput() {
		if (!elementValue) {
			log('Please enter a value!');
			return false;
		}
		return true;
	}

	// Elem beszúrása
	async function insertElement() {
		if (validateInput() && !animationInProgress) {
			animationInProgress = true;
			algorithmStatus.set('running');
			consoleLog.set([]);
			currentStep.set(0);
			selectedAlgorithmSourceCode.set(rbtreeInsertSource);
			activeLine.set({ start: 0, end: 0 });
			log(`Insert: ${elementValue}`);
			await insert(Number(elementValue));
			log(`Element added: ${elementValue}`);
			activeLine.set({ start: -1, end: -1 });
			algorithmStatus.set('idle');
			animationInProgress = false;
		}
	}

	async function insert(value: number) {
		selectedAlgorithmSourceCode.set(rbtreeInsertSource);
		activeLine.set({ start: 1, end: 3 });
		const newNode = new RBNode(value);
		if (!root) {
			activeLine.set({ start: 3, end: 5 });
			log(`New root: ${value}`);
			newNode.color = 'black';
			root = newNode;
			await calculatePositions(root);
			await animateNewNode(newNode);
		} else {
			let current = root;
			let parent = null;
			
			let path = [];
			
			while (current) {
				path.push(current);
				await highlightNode(current, true);
				await delay(animationSpeed / 2);
				
				parent = current;
				if (value < current.value) {
					activeLine.set({ start: 8, end: 10 });
					log(`Moving left: ${current.value}`);
					current = current.left;
				} else {
					activeLine.set({ start: 10, end: 12 });
					log(`Moving right: ${current.value}`);
					current = current.right;
				}
			}
			
			activeLine.set({ start: 8, end: 12 });
			newNode.parent = parent;
			if (value < parent.value) parent.left = newNode;
			else parent.right = newNode;

			await calculatePositions(root);
			await animateNewNode(newNode);
			
			for (const node of path) {
				await highlightNode(node, false);
			}

			activeLine.set({ start: 15, end: 15 });
			await fixInsert(newNode); // <-- balancing happens here
		}
		
		clearAllHighlights(root);
		await calculatePositions(root);
		drawTree();
	}

	async function fixInsert(node: RBNode) {
		selectedAlgorithmSourceCode.set(rbtreeInsertSource);
		activeLine.set({ start: 17, end: 18 });
		while (node !== root && node.parent?.color === 'red') {
			const parent = node.parent;
			const grandparent = parent.parent;

			if (!grandparent) break;

			activeLine.set({ start: 19, end: 21 });
			await highlightNode(node, true);
			await highlightNode(parent, true);
			await highlightNode(grandparent, true);
			await delay(animationSpeed / 2);

			if (parent === grandparent.left) {
				activeLine.set({ start: 23, end: 24 });
				const uncle = grandparent.right;

				if (uncle?.color === 'red') {
					activeLine.set({ start: 25, end: 28 });
					log('Case 1: Parent and uncle are both red');
					if (uncle) await highlightNode(uncle, true);
					
					parent.color = 'black';
					if (uncle) uncle.color = 'black';
					grandparent.color = 'red';
					
					await drawTreeWithAnimation();
					await delay(animationSpeed);
					
					if (uncle) await highlightNode(uncle, false);
					node = grandparent;
				} else {
					if (node === parent.right) {
						activeLine.set({ start: 30, end: 32 });
						log('Case 2: Left-right case - rotation');
						await rotateLeft(parent);
						node = parent;
					}
					activeLine.set({ start: 33, end: 35 });
					log('Case 3: Left-left case');
					node.parent!.color = 'black';
					grandparent.color = 'red';
					await drawTreeWithAnimation();
					await delay(animationSpeed);
					await rotateRight(grandparent);
				}
			} else {
				activeLine.set({ start: 37, end: 38 });
				const uncle = grandparent.left;

				if (uncle?.color === 'red') {
					activeLine.set({ start: 39, end: 42 });
					log('Mirror Case 1: Parent and uncle are both red');
					if (uncle) await highlightNode(uncle, true);
					
					parent.color = 'black';
					if (uncle) uncle.color = 'black';
					grandparent.color = 'red';
					
					await drawTreeWithAnimation();
					await delay(animationSpeed);
					
					if (uncle) await highlightNode(uncle, false);
					node = grandparent;
				} else {
					if (node === parent.left) {
						activeLine.set({ start: 44, end: 46 });
						log('Mirror Case 2: Right-left case - rotation');
						await rotateRight(parent);
						node = parent;
					}
					activeLine.set({ start: 47, end: 49 });
					log('Mirror Case 3: Right-right case');
					node.parent!.color = 'black';
					grandparent.color = 'red';
					await drawTreeWithAnimation();
					await delay(animationSpeed);
					await rotateLeft(grandparent);
				}
			}
			
			await highlightNode(node, false);
			if (parent && parent !== node) await highlightNode(parent, false);
			if (grandparent && grandparent !== node) await highlightNode(grandparent, false);
		}
		
		activeLine.set({ start: 51, end: 51 });
		root!.color = 'black';
		await drawTreeWithAnimation();
	}

	async function rotateLeft(x: RBNode) {
		log(`Rotate left: ${x.value}`);
		const y = x.right;
		if (!y) return;

		x.right = y.left;
		if (y.left) y.left.parent = x;

		y.parent = x.parent;
		if (!x.parent) {
			root = y;
		} else if (x === x.parent.left) {
			x.parent.left = y;
		} else {
			x.parent.right = y;
		}
		y.left = x;
		x.parent = y;
  
		await calculatePositions(root);
		await drawTreeWithAnimation();
		await delay(animationSpeed);
	}

	async function rotateRight(y: RBNode) {
		log(`Rotate right: ${y.value}`);
		const x = y.left;
		if (!x) return;

		y.left = x.right;
		if (x.right) x.right.parent = y;

		x.parent = y.parent;
		if (!y.parent) {
			root = x;
		} else if (y === y.parent.left) {
			y.parent.left = x;
		} else {
			y.parent.right = x;
		}
		x.right = y;
		y.parent = x;
		
		await calculatePositions(root);
		await drawTreeWithAnimation();
		await delay(animationSpeed);
	}

	async function deleteElement() {
		if (!validateInput() || animationInProgress) return;
		
		animationInProgress = true;
		algorithmStatus.set('running');
		consoleLog.set([]);
		currentStep.set(0);
		selectedAlgorithmSourceCode.set(rbtreeDeleteSource);
		activeLine.set({ start: 0, end: 0 });
		log(`Delete: ${elementValue}`);
		const value = Number(elementValue);
		const z = await searchElement(value);
		
		if (!z) {
			activeLine.set({ start: -1, end: -1 });
			algorithmStatus.set('idle');
			animationInProgress = false;
			return;
		}

		let y = z;
		let yOriginalColor = y.color;
		let x: RBNode | null;

		activeLine.set({ start: 7, end: 9 });
		await highlightNode(z, true);
		await delay(animationSpeed);
		
		log(`Deleting node: ${z.value}`);

		if (!z.left) {
			activeLine.set({ start: 10, end: 11 });
			x = z.right;
			await transplant(z, z.right);
		} else if (!z.right) {
			activeLine.set({ start: 12, end: 13 });
			x = z.left;
			await transplant(z, z.left);
		} else {
			activeLine.set({ start: 15, end: 17 });
			y = minimum(z.right);
			await highlightNode(y, true);
			log(`Replacing with successor: ${y.value}`);
			await delay(animationSpeed);
			
			yOriginalColor = y.color;
			x = y.right;

			if (y.parent === z) {
				if (x) x.parent = y;
			} else {
				await transplant(y, y.right);
				y.right = z.right;
				if (y.right) y.right.parent = y;
			}
			await transplant(z, y);
			y.left = z.left;
			if (y.left) y.left.parent = y;
			y.color = z.color;
			
			await highlightNode(y, false);
		}

		if (yOriginalColor === 'black') {
			activeLine.set({ start: 19, end: 19 });
			log('Fixing black height after deletion');
			await fixDelete(x, z.parent ?? null);
		}

		log(`Element deleted: ${value}`);
		activeLine.set({ start: -1, end: -1 });
		algorithmStatus.set('idle');
		
		await calculatePositions(root);
		await drawTreeWithAnimation();
		animationInProgress = false;
	}

	async function fixDelete(x: RBNode | null, parent: RBNode | null) {
		selectedAlgorithmSourceCode.set(rbtreeDeleteSource);
		activeLine.set({ start: 19, end: 19 });
		while (x !== root && (!x || x.color === 'black')) {
			if (!parent) break;
			
			if (x === parent?.left) {
				let w = parent.right;
				
				if (w) await highlightNode(w, true);
				await delay(animationSpeed / 2);
				
				if (w?.color === 'red') {
					activeLine.set({ start: 20, end: 22 });
					log('Delete Case 1: Sibling is red');
					w.color = 'black';
					parent.color = 'red';
					await drawTreeWithAnimation();
					await delay(animationSpeed);
					await rotateLeft(parent);
					w = parent.right;
					if (w) await highlightNode(w, true);
				}
				
				if ((!w?.left || w.left.color === 'black') && (!w?.right || w.right.color === 'black')) {
					activeLine.set({ start: 23, end: 25 });
					log('Delete Case 2: Sibling with black children');
					if (w) w.color = 'red';
					await drawTreeWithAnimation();
					if (w) await highlightNode(w, false);
					x = parent;
					parent = x.parent;
				} else {
					if (!w?.right || w.right.color === 'black') {
						activeLine.set({ start: 26, end: 28 });
						log('Delete Case 3: Sibling right child is black');
						if (w?.left) {
							await highlightNode(w.left, true);
							w.left.color = 'black';
							await delay(animationSpeed / 2);
							await highlightNode(w.left, false);
						}
						if (w) {
							w.color = 'red';
							await drawTreeWithAnimation();
							await delay(animationSpeed);
							await rotateRight(w);
							w = parent.right;
							if (w) await highlightNode(w, true);
						}
					}
					
					activeLine.set({ start: 29, end: 31 });
					log('Delete Case 4: Restructuring');
					if (w) w.color = parent.color;
					parent.color = 'black';
					if (w?.right) {
						await highlightNode(w.right, true);
						w.right.color = 'black';
						await delay(animationSpeed / 2);
						await highlightNode(w.right, false);
					}
					await drawTreeWithAnimation();
					await delay(animationSpeed);
					await rotateLeft(parent);
					if (w) await highlightNode(w, false);
					x = root;
				}
			} else {
				let w = parent?.left;
				
				if (w) await highlightNode(w, true);
				await delay(animationSpeed / 2);
				
				if (w?.color === 'red') {
					activeLine.set({ start: 20, end: 22 });
					log('Delete Mirror Case 1: Sibling is red');
					w.color = 'black';
					if (parent) parent.color = 'red';
					await drawTreeWithAnimation();
					await delay(animationSpeed);
					await rotateRight(parent);
					w = parent?.left;
					if (w) await highlightNode(w, true);
				}
				
				if ((!w?.right || w.right.color === 'black') && (!w?.left || w.left.color === 'black')) {
					activeLine.set({ start: 23, end: 25 });
					log('Delete Mirror Case 2: Sibling with black children');
					if (w) w.color = 'red';
					await drawTreeWithAnimation();
					if (w) await highlightNode(w, false);
					x = parent;
					parent = x?.parent ?? null;
				} else {
					if (!w?.left || w.left.color === 'black') {
						activeLine.set({ start: 26, end: 28 });
						log('Delete Mirror Case 3: Sibling left child is black');
						if (w?.right) {
							await highlightNode(w.right, true);
							w.right.color = 'black';
							await delay(animationSpeed / 2);
							await highlightNode(w.right, false);
						}
						if (w) {
							w.color = 'red';
							await drawTreeWithAnimation();
							await delay(animationSpeed);
							await rotateLeft(w);
							w = parent?.left;
							if (w) await highlightNode(w, true);
						}
					}
					
					activeLine.set({ start: 29, end: 31 });
					log('Delete Mirror Case 4: Restructuring');
					if (w) w.color = parent.color;
					if (parent) parent.color = 'black';
					if (w?.left) {
						await highlightNode(w.left, true);
						w.left.color = 'black';
						await delay(animationSpeed / 2);
						await highlightNode(w.left, false);
					}
					await drawTreeWithAnimation();
					await delay(animationSpeed);
					await rotateRight(parent);
					if (w) await highlightNode(w, false);
					x = root;
				}
			}
		}
		if (x) x.color = 'black';
		await drawTreeWithAnimation();
	}

	async function transplant(u: RBNode, v: RBNode | null) {
		if (!u.parent) {
			root = v;
		} else if (u === u.parent.left) {
			u.parent.left = v;
		} else {
			u.parent.right = v;
		}
		if (v) {
			v.parent = u.parent;
		}
		
		await calculatePositions(root);
		await drawTreeWithAnimation();
		await delay(animationSpeed / 2);
	}

	function minimum(node: RBNode): RBNode {
		let current = node;
		while (current.left) {
			current = current.left;
		}
		return current;
	}

	async function searchElement(value: number): Promise<RBNode | null> {
		if (!animationInProgress) {
			animationInProgress = true;
			algorithmStatus.set('running');
			consoleLog.set([]);
			currentStep.set(0);
			selectedAlgorithmSourceCode.set(rbtreeSearchSource);
			activeLine.set({ start: 0, end: 0 });
			log(`Search: ${value}`);
		}
		
		let current = root;
		
		while (current) {
			activeLine.set({ start: 3, end: 5 });
			await highlightNode(current, true);
			await delay(animationSpeed / 2);
			
			if (value === current.value) {
				activeLine.set({ start: 7, end: 9 });
				log(`Value found: ${value}`);
				await delay(animationSpeed / 2);
				activeLine.set({ start: -1, end: -1 });
				algorithmStatus.set('idle');
				animationInProgress = false;
				return current;
			} else if (value < current.value) {
				activeLine.set({ start: 10, end: 12 });
				log(`Searching left: ${current.value}`);
				await highlightNode(current, false);
				current = current.left;
			} else {
				activeLine.set({ start: 13, end: 15 });
				log(`Searching right: ${current.value}`);
				await highlightNode(current, false);
				current = current.right;
			}
		}
		
		activeLine.set({ start: 3, end: 5 });
		log(`Value not found: ${value}`);
		activeLine.set({ start: -1, end: -1 });
		algorithmStatus.set('idle');
		animationInProgress = false;
		return null;
	}
	
	async function createExampleTree() {
		if (animationInProgress) return;
		
		animationInProgress = true;
		algorithmStatus.set('running');
		consoleLog.set([]);
		currentStep.set(0);
		selectedAlgorithmSourceCode.set(rbtreeInsertSource);
		activeLine.set({ start: 0, end: 0 });
		await clearTree();
		
		log("Creating example tree...");
		
		const values = [10, 5, 15, 3, 7, 12, 20, 1, 4, 6, 8, 11, 13, 18, 25];
		
		for (const value of values) {
			elementValue = value;
			await insert(value);
			await delay(animationSpeed / 4); // Faster for example creation
		}
		
		clearAllHighlights(root);
		drawTree();
		
		log("Example tree created!");
		activeLine.set({ start: -1, end: -1 });
		algorithmStatus.set('idle');
		selectedAlgorithmSourceCode.set('Sample tree created.');
		animationInProgress = false;
	}
	
	async function clearTree() {
		if (animationInProgress && root !== null) return;
		
		root = null;
		consoleLog.set([]);
		activeLine.set({ start: -1, end: -1 });
		selectedAlgorithmSourceCode.set('Choose an operation!');
		log("Tree deleted!");
		drawTree();
	}
	
	async function animateNewNode(node: RBNode) {
		node.highlighted = true;
		await calculatePositions(root);
		drawTree();
		await delay(animationSpeed);
		node.highlighted = false;
		drawTree();
	}
	
	async function highlightNode(node: RBNode, highlight: boolean) {
		if (!node) return;
		node.highlighted = highlight;
		drawTree();
		if (highlight) {
			await delay(animationSpeed / 4);
		}
	}

	function clearAllHighlights(node: RBNode | null) {
		if (!node) return;
		node.highlighted = false;
		clearAllHighlights(node.left);
		clearAllHighlights(node.right);
	}

	// ==== Előkalkulált lépésszám ====
	onMount(() => {
		totalSteps.set(0);
		drawTree();
		
		return () => {
			if (animationFrameId !== null) {
				cancelAnimationFrame(animationFrameId);
			}
		};
	});

	// ==== Késleltetés és vezérlés ====
	async function restartAlgorithm() {
		if (get(algorithmStatus) === 'finished') {
			return new Promise((resolve) => {
				const unsub = resumeSignal.subscribe(() => {
					if (get(algorithmStatus) === 'idle') {
						consoleLog.set([]);
						currentStep.set(0);

						root = null;
						drawTree();

						unsub();
						resolve();
					}
				});
			});
		}
	}

	let svg: SVGSVGElement;
	
	function animationLoop(timestamp: number) {
		if (!lastTime) lastTime = timestamp;
		const deltaTime = timestamp - lastTime;
		lastTime = timestamp;
		
		let allSettled = true;
		
		const updatePositions = (node: RBNode | null) => {
			if (!node) return;
			
			const easing = 0.2;
			const dx = node.targetX - node.x;
			const dy = node.targetY - node.y;
			
			if (Math.abs(dx) > 0.1 || Math.abs(dy) > 0.1) {
				node.x += dx * easing;
				node.y += dy * easing;
				allSettled = false;
			} else {
				node.x = node.targetX;
				node.y = node.targetY;
			}
			
			updatePositions(node.left);
			updatePositions(node.right);
		};
		
		updatePositions(root);
		drawTree();
		
		if (!allSettled) {
			animationFrameId = requestAnimationFrame(animationLoop);
		} else {
			animationFrameId = null;
		}
	}
	
	async function drawTreeWithAnimation() {
		await calculatePositions(root);
		
		if (animationFrameId === null) {
			lastTime = 0;
			animationFrameId = requestAnimationFrame(animationLoop);
		}
		
		return new Promise<void>(resolve => {
			function checkAnimation() {
				if (animationFrameId === null) {
					resolve();
				} else {
					setTimeout(checkAnimation, 10);
				}
			}
			checkAnimation();
		});
	}
	
	const LAYOUT = {
		startX: 250,
		startY: 25,
		levelGapY: 80,
		nodeRadius: 15
	};

	function getOffsetX(depth: number): number {
		return 400 / Math.pow(2, depth + 1);
	}

	function calculatePositions(
		node: RBNode | null,
		depth = 0,
		offset = 0,
		parentX = LAYOUT.startX
	) {
		if (!node) return;

		const xOffset = getOffsetX(depth);
		node.targetX = parentX + offset * xOffset;
		node.targetY = LAYOUT.startY + depth * LAYOUT.levelGapY;

		if (typeof node.x !== 'number' || typeof node.y !== 'number') {
			node.x = node.targetX;
			node.y = node.targetY;
		}

		calculatePositions(node.left, depth + 1, -1, node.targetX);
		calculatePositions(node.right, depth + 1, 1, node.targetX);
	}

	function drawTree() {
		if (!svg) return;
		svg.innerHTML = '';
		
		drawNodeLines(svg, root);
		
		drawNodeCircles(svg, root);
	}
	
	function drawNodeLines(
		svg: SVGSVGElement,
		node: RBNode | null
	) {
		if (!node) return;
		
		if (node.left) {
			drawLine(svg, node.x, node.y, node.left.x, node.left.y);
			drawNodeLines(svg, node.left);
		}
		if (node.right) {
			drawLine(svg, node.x, node.y, node.right.x, node.right.y);
			drawNodeLines(svg, node.right);
		}
	}
	
	function drawNodeCircles(
		svg: SVGSVGElement,
		node: RBNode | null
	) {
		if (!node) return;
		
		const radius = 15;
		
		const circle = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
		circle.setAttribute('cx', node.x.toString());
		circle.setAttribute('cy', node.y.toString());
		circle.setAttribute('r', radius.toString());
		circle.setAttribute('fill', node.color === 'red' ? '#e74c3c' : '#2c3e50');
		
		if (node.highlighted) {
			circle.setAttribute('stroke', '#ffcc00');
			circle.setAttribute('stroke-width', '4');
		} else {
			circle.setAttribute('stroke', 'white');
			circle.setAttribute('stroke-width', '2');
		}
		
		svg.appendChild(circle);

		const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');
		text.setAttribute('x', node.x.toString());
		text.setAttribute('y', (node.y + 5).toString());
		text.setAttribute('text-anchor', 'middle');
		text.setAttribute('fill', 'white');
		text.setAttribute('font-size', '12');
		text.textContent = node.value.toString();
		svg.appendChild(text);
		
		drawNodeCircles(svg, node.left);
		drawNodeCircles(svg, node.right);
	}

	function drawLine(svg: SVGSVGElement, x1: number, y1: number, x2: number, y2: number) {
		const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
		line.setAttribute('x1', x1.toString());
		line.setAttribute('y1', y1.toString());
		line.setAttribute('x2', x2.toString());
		line.setAttribute('y2', y2.toString());
		line.setAttribute('stroke', 'white');
		line.setAttribute('stroke-width', '2');
		svg.appendChild(line);
	}

	function setAnimationSpeed(speed: string) {
		const speedValue = parseInt(speed);
		animationSpeed = 1400 - speedValue;
	}
	
	// ==== Forráskód megjelenítés ====
	selectedAlgorithmSourceCode.set('Choose an operation!');
</script>

<div class="control-buttons">
	<input class="custom-input" type="number" bind:value={elementValue} placeholder="Node's value" />
	<div>
		<button on:click={insertElement}>Insert Node</button>
		<button on:click={deleteElement}>Delete Node</button>
		<button on:click={async () => {
			if (validateInput() && !animationInProgress) {
				await searchElement(Number(elementValue));
			}
		}}>Search Node</button>
		<button on:click={createExampleTree} class="special-button">Sample Tree</button>
		<button on:click={clearTree} class="clear-button">Delete Tree</button>
	</div>
</div>

<!-- ==== Komponens markup ==== -->
<div class="algorithm-container">
	<div class="array-visual">
		<svg class="svg" width="500" height="300" bind:this={svg}></svg>
	</div>
</div>

<!-- ==== Stílus ==== -->
<style>
	.control-buttons {
		display: flex;
		justify-content: space-around;
		align-items: center;
		padding: 1rem;
		border-bottom: 3px solid #505050;
		flex-wrap: wrap;
		gap: 8px;
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

	.control-buttons button {
		padding: 0.5rem 1rem;
		background-color: #505050;
		color: white;
		border: none;
		border-radius: 5px;
		cursor: pointer;
		transition: background-color 0.2s;
	}

	.control-buttons div {
		display: flex;
		gap: 10px;
		flex-wrap: wrap;
		justify-content: center;
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
		border: 1px solid #ccc;
		border-radius: 4px;
		display: block;
		background-color: #2f2f2f;
	}
</style>
