<script>
	import StreetCard from '$lib/components/StreetCard.svelte';
	import { onMount } from 'svelte';

	let radius = $state(5);
	let filterMode = $state('priority-high');
	let isFilterOpen = $state(false);
	let totalEarnings = $state(0);

	const USER_LOCATION = { lat: 43.6850, lng: -79.7596 };

	let mapElement;
	let map;
	let markers = [];
	let verifyingTask = $state(null);
	let taskSuccess = $state(false);
	let selectedStreetId = $state(null);

	const filterOptions = [
		{ value: 'alpha-asc', label: 'A-Z' },
		{ value: 'alpha-desc', label: 'Z-A' },
		{ value: 'payout-high', label: 'Payout (High-Low)' },
		{ value: 'payout-low', label: 'Payout (Low-High)' },
		{ value: 'priority-high', label: 'Priority (High-Low)' },
		{ value: 'priority-low', label: 'Priority (Low-High)' }
	];

	function toggleFilter() {
		isFilterOpen = !isFilterOpen;
	}

	function selectFilter(value) {
		filterMode = value;
		isFilterOpen = false;
	}

	// Close dropdown when clicking outside
	function handleClickOutside(e) {
		if (!e.target.closest('.filter-container')) {
			isFilterOpen = false;
		}
	}

	// Calculate dynamic price based on priority and time elapsed
	function calculatePrice(basePriority, createdAt) {
		const BASE_PRICE = 100;
		const minutesElapsed = (Date.now() - createdAt) / (1000 * 60);

		// Priority 0: 0.1% per minute, Priority 10: 1% per minute
		const growthRatePerMinute = 0.001 + (basePriority / 10) * 0.009; // 0.1% to 1%

		const currentPrice = BASE_PRICE * Math.pow(1 + growthRatePerMinute, minutesElapsed);
		return Math.round(currentPrice);
	}

	// Color earnings based on amount (more green = higher)
	function getEarningsColor(amount) {
		if (amount === 0) return '#666';
		const intensity = Math.min(amount / 1000, 1); // Scale to 1000
		const r = Math.round(16 - intensity * 6);
		const g = Math.round(185 - intensity * 65);
		const b = Math.round(129 - intensity * 100);
		return `rgb(${r}, ${g}, ${b})`;
	}

	// Color average payout (more green = higher)
	function getPayoutColor(avgPayout) {
		const intensity = Math.min(avgPayout / 200, 1); // Scale to 200
		const r = Math.round(107 - intensity * 97);
		const g = Math.round(114 - intensity * -71);
		const b = Math.round(128 - intensity * -1);
		return `rgb(${r}, ${g}, ${b})`;
	}

	// Color priority score (0 = green, 10 = red) - same as in StreetCard
	function getPriorityColor(score) {
		const clampedScore = Math.max(0, Math.min(10, score));
		const ratio = clampedScore / 10;
		const r = Math.round(10 + ratio * (239 - 10));
		const g = Math.round(185 - ratio * (185 - 68));
		const b = Math.round(129 - ratio * (129 - 68));
		return `rgb(${r}, ${g}, ${b})`;
	}

	async function handleCardComplete(price, isCompleted, streetData) {
		if (isCompleted) {
			verifyingTask = { streetName: streetData.streetName };

			await new Promise(resolve => setTimeout(resolve, 5000));

			verifyingTask = null;
			taskSuccess = true;

			if (typeof window !== 'undefined' && window.confetti) {
				window.confetti({
					particleCount: 100,
					spread: 70,
					origin: { y: 0.6 }
				});
			}

			totalEarnings += price;
			streetData.priorityScore = 0;
			streetData.completed = true;

			// Update all map points for this street
			allMapPoints.forEach(point => {
				if (point.streetName === streetData.streetName) {
					point.priorityScore = 0;
				}
			});

			setTimeout(() => {
				taskSuccess = false;
			}, 2000);
		} else {
			totalEarnings -= price;
		}
	}

	// Calculate distance between two coordinates in kilometers using Haversine formula
	function calculateDistance(lat1, lng1, lat2, lng2) {
		const R = 6371; // Earth's radius in kilometers
		const dLat = (lat2 - lat1) * Math.PI / 180;
		const dLng = (lng2 - lng1) * Math.PI / 180;
		const a =
			Math.sin(dLat / 2) * Math.sin(dLat / 2) +
			Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
			Math.sin(dLng / 2) * Math.sin(dLng / 2);
		const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
		return R * c; // Distance in kilometers
	}

	// Convert lat/lng to SVG coordinates
	// Map bounds: Brampton area (roughly 43.6-43.8 lat, -79.9 to -79.6 lng)
	function latLngToSVG(lat, lng) {
		// Map boundaries for Brampton area
		const minLat = 43.6;
		const maxLat = 43.8;
		const minLng = -79.9;
		const maxLng = -79.6;

		// Normalize to 0-1 range
		const x = (lng - minLng) / (maxLng - minLng);
		const y = 1 - (lat - minLat) / (maxLat - minLat); // Invert Y for SVG coordinates

		// Map to SVG viewBox (0-430 width, 0-932 height)
		return {
			x: x * 430,
			y: y * 932
		};
	}

	// Load and process CSV data
	const now = Date.now();
	let streetsData = $state([]);
	let allMapPoints = $state([]); // Store all points
	let hoveredPoint = $state(null);
	let isDataLoading = $state(true);

	onMount(() => {
		loadCSVData();
		initMap();
	});

	function initMap() {
		if (typeof window === 'undefined' || !window.L) return;

		map = window.L.map(mapElement, {
			center: [USER_LOCATION.lat, USER_LOCATION.lng],
			zoom: 12,
			zoomControl: true
		});

		window.L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
			attribution: '© OpenStreetMap'
		}).addTo(map);
	}

	let radiusCircle;

	function updateMapMarkers() {
		if (!map || !window.L) return;

		markers.forEach(m => map.removeLayer(m));
		markers = [];

		if (radiusCircle) map.removeLayer(radiusCircle);
		radiusCircle = window.L.circle([USER_LOCATION.lat, USER_LOCATION.lng], {
			radius: radius * 1000,
			color: '#4DB6ED',
			fillOpacity: 0.1
		}).addTo(map);

		mapPoints().forEach(point => {
			const color = getPriorityColor(point.priorityScore);
			const marker = window.L.circleMarker([point.coordinates.lat, point.coordinates.lng], {
				radius: 6,
				fillColor: color,
				color: '#fff',
				weight: 1,
				fillOpacity: 0.8
			}).addTo(map);

			marker.bindPopup(`<strong>${point.streetName}</strong><br>Payout: $${calculatePrice(point.priorityScore, point.createdAt)}<br>Priority: ${point.priorityScore}`);
			marker.on('click', () => {
				const street = streetsData.find(s => s.streetName === point.streetName);
				if (street) selectedStreetId = street.id;
			});
			markers.push(marker);
		});
	}

	$effect(() => {
		radius;
		updateMapMarkers();
	});

	$effect(() => {
		if (selectedStreetId) {
			document.getElementById(`street-${selectedStreetId}`)?.scrollIntoView({ behavior: 'smooth', block: 'center' });
		}
	});

	async function loadCSVData() {
		try {
			const response = await fetch('/data/ndsi_data.csv');
			const csvText = await response.text();

			// Parse CSV (simple parsing - skip header, split by lines and commas)
			const lines = csvText.trim().split('\n');
			const headers = lines[0].split(',');

			// Parse all rows with data
			const rows = lines.slice(1).map(line => {
				const values = line.split(',');
				return {
					seg_id: values[0],
					vertex_seq: values[1],
					lon: parseFloat(values[2]),
					lat: parseFloat(values[3]),
					traffic_score: parseFloat(values[4]),
					ndsi: parseFloat(values[5]),
					used_image_date: values[6],
					street_address: values[7]
				};
			});

			// Filter: only rows with street addresses and valid ndsi values
			const validRows = rows.filter(row =>
				row.street_address &&
				row.street_address.trim() !== '' &&
				!isNaN(row.ndsi) &&
				!isNaN(row.traffic_score)
			);

			console.log(`Loaded ${validRows.length} valid locations from CSV`);

			// Calculate raw priority scores (traffic_score * ndsi)
			const rawScores = validRows.map(row => row.traffic_score * row.ndsi);

			// Calculate mean and standard deviation for normalization
			const mean = rawScores.reduce((a, b) => a + b, 0) / rawScores.length;
			const variance = rawScores.reduce((sum, score) => sum + Math.pow(score - mean, 2), 0) / rawScores.length;
			const stdDev = Math.sqrt(variance);

			// Normalize to 1-10 scale using z-score normalization
			// Then map to 1-10 range (clamping outliers)
			const processedData = validRows.map((row, index) => {
				const rawScore = rawScores[index];
				const zScore = (rawScore - mean) / stdDev;

				// Map z-score to 1-10 range
				// Most z-scores fall between -3 and +3, map this to 1-10
				const normalizedScore = Math.max(1, Math.min(10, 5.5 + (zScore * 1.5)));

				return {
					id: `${row.seg_id}-${row.vertex_seq}`,
					coordinates: { lat: row.lat, lng: row.lon },
					streetName: row.street_address,
					priorityScore: parseFloat(normalizedScore.toFixed(1)),
					createdAt: now,
					completed: false,
					rawScore: rawScore
				};
			});

			// Group by street address and take the entry with highest priority per street
			const streetMap = new Map();
			processedData.forEach(location => {
				const existing = streetMap.get(location.streetName);
				if (!existing || location.priorityScore > existing.priorityScore) {
					streetMap.set(location.streetName, location);
				}
			});

			// Convert to array and assign sequential IDs
			streetsData = Array.from(streetMap.values()).map((street, idx) => ({
				...street,
				id: idx + 1
			}));

			// Store all map points for visualization
			allMapPoints = processedData;

			isDataLoading = false;
			console.log(`Created ${streetsData.length} unique street cards and ${allMapPoints.length} total map points`);
		} catch (error) {
			console.error('Error loading CSV data:', error);
			isDataLoading = false;
		}
	}

	// Calculate current prices for all streets
	let streetsWithPrices = $derived(() => {
		// Trigger recalculation when priceUpdateTrigger changes
		priceUpdateTrigger;
		return streetsData.map(street => ({
			...street,
			price: calculatePrice(street.priorityScore, street.createdAt)
		}));
	});

	// Filtered and sorted streets based on selected filter and radius
	let streets = $derived(() => {
		// First filter by radius distance from user location and exclude completed
		const withinRadius = streetsWithPrices().filter(street => {
			const distance = calculateDistance(
				USER_LOCATION.lat,
				USER_LOCATION.lng,
				street.coordinates.lat,
				street.coordinates.lng
			);
			return distance <= radius && !street.completed;
		});

		// Then sort based on selected filter
		const sorted = [...withinRadius];

		switch (filterMode) {
			case 'alpha-asc':
				return sorted.sort((a, b) => a.streetName.localeCompare(b.streetName));
			case 'alpha-desc':
				return sorted.sort((a, b) => b.streetName.localeCompare(a.streetName));
			case 'payout-high':
				return sorted.sort((a, b) => b.price - a.price);
			case 'payout-low':
				return sorted.sort((a, b) => a.price - b.price);
			case 'priority-high':
				return sorted.sort((a, b) => b.priorityScore - a.priorityScore);
			case 'priority-low':
				return sorted.sort((a, b) => a.priorityScore - b.priorityScore);
			default:
				return sorted;
		}
	});

	// Filter map points by radius for performance (only render visible points)
	let mapPoints = $derived(() => {
		return allMapPoints.filter(point => {
			const distance = calculateDistance(
				USER_LOCATION.lat,
				USER_LOCATION.lng,
				point.coordinates.lat,
				point.coordinates.lng
			);
			return distance <= radius && point.priorityScore > 0;
		});
	});

	// Calculate statistics for peek mode (only for streets within radius)
	let stats = $derived(() => {
		const incomplete = streets().filter(s => !s.completed);
		const avgPayout = incomplete.length > 0
			? Math.round(incomplete.reduce((sum, s) => sum + s.price, 0) / incomplete.length)
			: 0;
		const avgPriority = incomplete.length > 0
			? (incomplete.reduce((sum, s) => sum + s.priorityScore, 0) / incomplete.length).toFixed(1)
			: 0;

		return {
			earnings: totalEarnings,
			jobCount: incomplete.length,
			avgPayout,
			avgPriority: parseFloat(avgPriority)
		};
	});

	// Update prices every minute with countdown
	let priceUpdateTrigger = $state(0);
	let countdown = $state(60);

	$effect(() => {
		// Countdown every second
		const countdownInterval = setInterval(() => {
			countdown -= 1;
			if (countdown <= 0) {
				countdown = 60;
				priceUpdateTrigger = Date.now();
			}
		}, 1000);

		return () => clearInterval(countdownInterval);
	});

</script>

<div class="page-wrapper">
	<div class="left-panel">
		<div class="stats-header">
			<h2>Earned <span style="color: {getEarningsColor(totalEarnings)};">${totalEarnings}</span></h2>
			<div class="filter-container">
				<button class="filter-icon-button" onclick={toggleFilter}>
					<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 16 16" fill="currentColor">
						<path d="M8 16A8 8 0 1 0 8 0a8 8 0 0 0 0 16zM3.5 5h9a.5.5 0 0 1 0 1h-9a.5.5 0 0 1 0-1zM5 8.5a.5.5 0 0 1 .5-.5h5a.5.5 0 0 1 0 1h-5a.5.5 0 0 1-.5-.5zm2 3a.5.5 0 0 1 .5-.5h1a.5.5 0 0 1 0 1h-1a.5.5 0 0 1-.5-.5z"/>
					</svg>
				</button>
				{#if isFilterOpen}
					<div class="filter-dropdown-menu">
						{#each filterOptions as option}
							<button class="filter-option" class:active={filterMode === option.value} onclick={() => selectFilter(option.value)}>
								{option.label}
							</button>
						{/each}
					</div>
				{/if}
			</div>
		</div>
		<div class="radius-control">
			<label>Radius: {radius}km</label>
			<input type="range" min="0.25" max="6" step="0.25" bind:value={radius} />
		</div>
		<p>{countdown}s to boost</p>
		<div class="street-list">
			{#each streets() as street}
				<div id="street-{street.id}" class:highlighted={selectedStreetId === street.id}>
					<StreetCard
						coordinates={street.coordinates}
						streetName={street.streetName}
						price={street.price}
						priorityScore={street.priorityScore}
						onComplete={(price, isCompleted) => {
							const streetData = streetsData.find(s => s.id === street.id);
							if (streetData) {
								handleCardComplete(price, isCompleted, streetData);
							}
						}}
					/>
				</div>
			{/each}
		</div>
	</div>
	<div class="right-panel">
		<div bind:this={mapElement} class="map"></div>
	</div>

	{#if verifyingTask}
		<div class="modal">
			<div class="modal-content">
				<div class="spinner"></div>
				<h3>Verifying with satellite...</h3>
				<p>{verifyingTask.streetName}</p>
			</div>
		</div>
	{/if}

	{#if taskSuccess}
		<div class="modal">
			<div class="modal-content success">
				<div class="checkmark">✓</div>
				<h3>Task Successfully Done!</h3>
			</div>
		</div>
	{/if}
</div>

<svelte:window onclick={handleClickOutside} />

<style>
	.page-wrapper {
		display: flex;
		height: 100vh;
	}

	.left-panel {
		width: 40%;
		padding: 24px;
		overflow-y: auto;
		background: #fff;
	}

	.left-panel::-webkit-scrollbar {
		width: 8px;
		display: block;
	}

	.left-panel::-webkit-scrollbar-track {
		background: transparent;
	}

	.left-panel::-webkit-scrollbar-thumb {
		background: #ccc;
		border-radius: 4px;
		transition: all 0.2s;
	}

	.left-panel::-webkit-scrollbar-thumb:hover {
		background: #555;
		width: 12px;
		border-radius: 6px;
	}

	.right-panel {
		width: 60%;
		position: relative;
	}

	.map {
		width: 100%;
		height: 100%;
	}

	.stats-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 16px;
	}

	.stats-header h2 {
		font-family: var(--font-title);
		font-size: 28px;
		margin: 0;
	}

	.filter-container {
		position: relative;
	}

	.filter-icon-button {
		width: 32px;
		height: 32px;
		border: none;
		background: transparent;
		cursor: pointer;
		color: #6b7280;
		border-radius: 6px;
		padding: 0;
	}

	.filter-icon-button:hover {
		background: #f3f4f6;
	}

	.filter-dropdown-menu {
		position: absolute;
		top: calc(100% + 4px);
		right: 0;
		background: white;
		border: 1px solid #e5e5e5;
		border-radius: 10px;
		box-shadow: 0 4px 20px rgba(0,0,0,0.12);
		padding: 6px;
		min-width: 180px;
		z-index: 1000;
	}

	.filter-option {
		width: 100%;
		padding: 10px 12px;
		border: none;
		background: transparent;
		text-align: left;
		font-size: 14px;
		cursor: pointer;
		border-radius: 6px;
	}

	.filter-option:hover {
		background: #f3f4f6;
	}

	.filter-option.active {
		background: #eff6ff;
		color: #007aff;
		font-weight: 500;
	}

	.left-panel p {
		color: #6b7280;
		margin: 0 0 16px;
	}

	.radius-control {
		margin: 16px 0;
	}

	.radius-control label {
		display: block;
		margin-bottom: 8px;
		font-size: 14px;
		color: #374151;
	}

	.radius-control input {
		width: 100%;
		cursor: pointer;
	}

	.street-list {
		display: flex;
		flex-direction: column;
		gap: 16px;
	}

	.highlighted {
		outline: 2px solid #4DB6ED;
		border-radius: 12px;
		animation: pulse 0.5s ease-in-out;
	}

	@keyframes pulse {
		0%, 100% { transform: scale(1); }
		50% { transform: scale(1.02); }
	}

	.modal {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.5);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 10000;
	}

	.modal-content {
		background: white;
		padding: 40px;
		border-radius: 16px;
		text-align: center;
		box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
	}

	.spinner {
		width: 48px;
		height: 48px;
		border: 4px solid #e5e5e5;
		border-top: 4px solid #4DB6ED;
		border-radius: 50%;
		animation: spin 1s linear infinite;
		margin: 0 auto 16px;
	}

	@keyframes spin {
		to { transform: rotate(360deg); }
	}

	.modal-content h3 {
		margin: 0 0 8px;
		font-size: 20px;
	}

	.modal-content p {
		margin: 0;
		color: #6b7280;
	}

	.checkmark {
		width: 64px;
		height: 64px;
		background: #10b981;
		color: white;
		border-radius: 50%;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 40px;
		margin: 0 auto 16px;
	}

	.modal-content.success h3 {
		color: #10b981;
	}
</style>
