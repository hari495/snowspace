<script>
	import Drawer from '$lib/components/Drawer.svelte';
	import StreetCard from '$lib/components/StreetCard.svelte';

	let radius = $state(10); // Default radius in km
	let filterMode = $state('priority-high'); // Default filter
	let isFilterOpen = $state(false);
	let totalEarnings = $state(0);

	// Brampton City Hall coordinates (user location)
	const USER_LOCATION = { lat: 43.6850, lng: -79.7596 };

	// Map state
	let isDrawerPeekMode = $state(false);

	// Calculate radius in pixels for SVG (approximate conversion for visualization)
	let radiusPx = $derived(radius * 8); // Rough approximation: 1km = ~8px at this zoom level

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

	function handleCardComplete(price, isCompleted) {
		if (isCompleted) {
			totalEarnings += price;
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

	// Load CSV data on mount
	$effect(() => {
		if (typeof window !== 'undefined') {
			loadCSVData();
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
		// First filter by radius distance from user location
		const withinRadius = streetsWithPrices().filter(street => {
			const distance = calculateDistance(
				USER_LOCATION.lat,
				USER_LOCATION.lng,
				street.coordinates.lat,
				street.coordinates.lng
			);
			return distance <= radius;
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
			return distance <= radius;
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
	<!-- Simple SVG Map -->
	<div class="map-container" class:map-interactive={isDrawerPeekMode}>
		<!-- Pure SVG map - white background with location and radius -->
		<svg class="map-svg" viewBox="0 0 430 932" preserveAspectRatio="xMidYMid slice">
			<!-- White background -->
			<rect width="430" height="932" fill="#ffffff" />

			<!-- Radius circle -->
			<circle
				cx="215"
				cy="466"
				r={radiusPx}
				fill="#4DB6ED"
				fill-opacity="0.1"
				stroke="#4DB6ED"
				stroke-width="2"
			/>

			<!-- Data points from CSV (filtered by radius for performance) -->
			{#each mapPoints() as point (point.id)}
				{@const svgCoords = latLngToSVG(point.coordinates.lat, point.coordinates.lng)}
				{@const price = calculatePrice(point.priorityScore, point.createdAt)}
				<circle
					cx={svgCoords.x}
					cy={svgCoords.y}
					r="4"
					fill={getPriorityColor(point.priorityScore)}
					stroke="#ffffff"
					stroke-width="1"
					class="map-point"
					onmouseenter={() => hoveredPoint = { ...point, price, x: svgCoords.x, y: svgCoords.y }}
					onmouseleave={() => hoveredPoint = null}
					role="button"
					tabindex="0"
				/>
			{/each}

			<!-- User location marker (on top) -->
			<circle
				cx="215"
				cy="466"
				r="8"
				fill="#4DB6ED"
				stroke="#ffffff"
				stroke-width="2"
			/>

			<!-- Hover tooltip -->
			{#if hoveredPoint}
				<g class="tooltip">
					<!-- Tooltip background -->
					<rect
						x={hoveredPoint.x + 10}
						y={hoveredPoint.y - 60}
						width="140"
						height="54"
						rx="8"
						fill="#ffffff"
						stroke="#e5e5e5"
						stroke-width="1"
						filter="drop-shadow(0 2px 8px rgba(0,0,0,0.15))"
					/>
					<!-- Tooltip text -->
					<text
						x={hoveredPoint.x + 16}
						y={hoveredPoint.y - 42}
						font-family="var(--font-body)"
						font-size="11"
						font-weight="600"
						fill="#000"
					>
						{hoveredPoint.streetName.length > 18 ? hoveredPoint.streetName.slice(0, 18) + '...' : hoveredPoint.streetName}
					</text>
					<text
						x={hoveredPoint.x + 16}
						y={hoveredPoint.y - 28}
						font-family="var(--font-body)"
						font-size="10"
						fill="#666"
					>
						Payout: ${hoveredPoint.price}
					</text>
					<text
						x={hoveredPoint.x + 16}
						y={hoveredPoint.y - 14}
						font-family="var(--font-body)"
						font-size="10"
						fill={getPriorityColor(hoveredPoint.priorityScore)}
					>
						Priority: {hoveredPoint.priorityScore}
					</text>
				</g>
			{/if}
		</svg>
	</div>

	<!-- Radius Slider -->
	<div class="radius-slider-container">
		<div class="radius-display">Radius</div>
		<div class="slider-wrapper">
			<span class="label-min">10km</span>
			<input
				type="range"
				min="10"
				max="250"
				bind:value={radius}
				class="radius-slider"
				aria-label="Map radius in kilometers"
			/>
			<span class="label-max">250km</span>
		</div>
	</div>

	<Drawer bind:isPeekMode={isDrawerPeekMode}>
		{#snippet peekContent()}
			<div class="peek-stats">
				<div class="stat-item">
					<span class="stat-value" style="color: {getEarningsColor(stats().earnings)};">${stats().earnings}</span>
					<span class="stat-label">Earned</span>
				</div>
				<div class="stat-item">
					<span class="stat-value">{stats().jobCount}</span>
					<span class="stat-label">Jobs</span>
				</div>
				<div class="stat-item">
					<span class="stat-value" style="color: {getPayoutColor(stats().avgPayout)};">${stats().avgPayout}</span>
					<span class="stat-label">Avg Pay</span>
				</div>
				<div class="stat-item">
					<span class="stat-value" style="color: {getPriorityColor(stats().avgPriority)}; font-weight: 600;">{stats().avgPriority}</span>
					<span class="stat-label">Avg Priority</span>
				</div>
			</div>
		{/snippet}

		<div class="drawer-inner">
			<div class="drawer-header">
				<h2>You've Earned <span style="color: {getEarningsColor(totalEarnings)};">${totalEarnings}</span> Today.</h2>
				<div class="filter-container">
					<button class="filter-icon-button" onclick={toggleFilter} aria-label="Filter options">
						<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 16 16" fill="currentColor" class="filter-icon">
							<path d="M8 16A8 8 0 1 0 8 0a8 8 0 0 0 0 16zM3.5 5h9a.5.5 0 0 1 0 1h-9a.5.5 0 0 1 0-1zM5 8.5a.5.5 0 0 1 .5-.5h5a.5.5 0 0 1 0 1h-5a.5.5 0 0 1-.5-.5zm2 3a.5.5 0 0 1 .5-.5h1a.5.5 0 0 1 0 1h-1a.5.5 0 0 1-.5-.5z"/>
						</svg>
					</button>
					{#if isFilterOpen}
						<div class="filter-dropdown-menu">
							{#each filterOptions as option}
								<button
									class="filter-option"
									class:active={filterMode === option.value}
									onclick={() => selectFilter(option.value)}
								>
									{option.label}
								</button>
							{/each}
						</div>
					{/if}
				</div>
			</div>
			<p>{countdown} seconds to payout boosts!</p>

			<div class="street-list">
				{#each streets() as street}
					<StreetCard
						coordinates={street.coordinates}
						streetName={street.streetName}
						price={street.price}
						priorityScore={street.priorityScore}
						onComplete={(price, isCompleted) => {
							const streetData = streetsData.find(s => s.id === street.id);
							if (streetData) {
								streetData.completed = isCompleted;
								handleCardComplete(price, isCompleted);
							}
						}}
					/>
				{/each}
			</div>
		</div>
	</Drawer>
</div>

<svelte:window onclick={handleClickOutside} />

<style>
	.page-wrapper {
		position: absolute;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
	}

	/* Map Styles */
	.map-container {
		position: absolute;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		z-index: 1;
		pointer-events: none; /* Always disabled - map is decorative */
		overflow: hidden;
	}

	.map-svg {
		width: 100%;
		height: 100%;
		pointer-events: none;
	}

	/* Map point styles */
	:global(.map-point) {
		cursor: pointer;
		pointer-events: auto;
		transition: r 0.2s, opacity 0.2s;
	}

	:global(.map-point:hover) {
		r: 6;
		opacity: 0.9;
	}

	:global(.tooltip) {
		pointer-events: none;
	}

	/* Radius Slider Styles */
	.radius-slider-container {
		position: absolute;
		top: var(--spacing-md);
		left: 50%;
		transform: translateX(-50%);
		z-index: 100;
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 6px;
		width: 240px;
		background: rgba(255, 255, 255, 0.95);
		padding: var(--spacing-sm);
		border-radius: 16px;
		box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15);
	}

	.radius-display {
		font-family: var(--font-title);
		font-size: 18px;
		font-weight: normal;
		color: #000000;
		letter-spacing: 0.3px;
	}

	.slider-wrapper {
		width: 100%;
		display: flex;
		align-items: center;
		gap: var(--spacing-xs);
	}

	.label-min,
	.label-max {
		font-family: var(--font-body);
		font-size: 11px;
		color: #9ca3af;
		flex-shrink: 0;
	}

	.radius-slider {
		flex: 1;
		height: 3px;
		-webkit-appearance: none;
		appearance: none;
		background: #d1d5db;
		border-radius: 1.5px;
		outline: none;
		cursor: pointer;
	}

	.radius-slider::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		width: 14px;
		height: 14px;
		background: #9ca3af;
		border: none;
		border-radius: 50%;
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.radius-slider::-webkit-slider-thumb:hover {
		background: #6b7280;
		transform: scale(1.15);
	}

	.radius-slider::-webkit-slider-thumb:active {
		background: #4b5563;
		transform: scale(1.1);
	}

	.radius-slider::-moz-range-thumb {
		width: 14px;
		height: 14px;
		background: #9ca3af;
		border: none;
		border-radius: 50%;
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.radius-slider::-moz-range-thumb:hover {
		background: #6b7280;
		transform: scale(1.15);
	}

	.radius-slider::-moz-range-thumb:active {
		background: #4b5563;
		transform: scale(1.1);
	}

	/* Peek state styles */
	.peek-stats {
		display: flex;
		gap: var(--spacing-md);
		justify-content: space-around;
		padding: var(--spacing-xs) 0;
	}

	.stat-item {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 4px;
	}

	.stat-value {
		font-family: var(--font-body);
		font-size: 20px;
		font-weight: 600;
		color: var(--color-text);
	}

	.stat-label {
		font-family: var(--font-body);
		font-size: 12px;
		color: #6b7280;
	}

	/* Expanded drawer content */
	.drawer-inner {
		height: 100%;
		padding: var(--spacing-md);
		display: flex;
		flex-direction: column;
		gap: var(--spacing-md);
	}

	.drawer-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: var(--spacing-sm);
	}

	.drawer-inner h2 {
		font-family: var(--font-title);
		font-size: 28px;
		font-weight: normal;
		margin: 0;
		flex: 1;
		min-width: 0;
	}

	.filter-container {
		position: relative;
		flex-shrink: 0;
	}

	.filter-icon-button {
		width: 32px;
		height: 32px;
		border: none;
		background: transparent;
		display: flex;
		align-items: center;
		justify-content: center;
		cursor: pointer;
		color: #6b7280;
		border-radius: 6px;
		transition: all 0.2s ease;
		padding: 0;
	}

	.filter-icon-button:hover {
		background: #f3f4f6;
		color: #374151;
	}

	.filter-icon-button:active {
		transform: scale(0.95);
	}

	.filter-dropdown-menu {
		position: absolute;
		top: calc(100% + 4px);
		right: 0;
		background: white;
		border: 1px solid var(--color-border);
		border-radius: 10px;
		box-shadow: 0 4px 20px rgba(0, 0, 0, 0.12);
		padding: 6px;
		min-width: 180px;
		z-index: 1000;
		animation: dropdownFadeIn 0.15s ease-out;
	}

	@keyframes dropdownFadeIn {
		from {
			opacity: 0;
			transform: translateY(-8px);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	.filter-option {
		width: 100%;
		padding: 10px 12px;
		border: none;
		background: transparent;
		text-align: left;
		font-family: var(--font-body);
		font-size: 14px;
		color: #374151;
		cursor: pointer;
		border-radius: 6px;
		transition: all 0.15s ease;
		display: block;
	}

	.filter-option:hover {
		background: #f3f4f6;
	}

	.filter-option.active {
		background: #eff6ff;
		color: var(--color-accent);
		font-weight: 500;
	}

	.drawer-inner p {
		font-family: var(--font-body);
		font-size: 16px;
		color: #6b7280;
		margin: 0;
	}

	.street-list {
		display: flex;
		flex-direction: column;
		gap: var(--spacing-sm);
	}
</style>
