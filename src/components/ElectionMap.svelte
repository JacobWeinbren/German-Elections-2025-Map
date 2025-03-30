<script>
	import { onMount } from "svelte";
	import mapboxgl from "mapbox-gl";
	import MapboxGeocoder from "@mapbox/mapbox-gl-geocoder";
	import { Info } from "lucide-svelte";

	// Get token from environment variable
	const MAPBOX_TOKEN = import.meta.env.PUBLIC_MAPBOX_TOKEN;

	// Variables for map state
	let map;
	let selectedView = "leading";
	let showAbout = false;
	let hoverInfo = null;
	let isMobile = false;

	// Set fallback values directly
	let partyStats = {
		// Major parties: Results 16.4%, 20.8%, 28.5%
		// Breakpoints chosen to place these results in distinct middle ranges
		SPD: { breakpoints: [0, 15, 25, 35, 45] },
		UNION: { breakpoints: [0, 15, 25, 35, 45] },
		AfD: { breakpoints: [0, 15, 25, 35, 45] },

		// Greens: Result 11.6%
		GRÜNE: { breakpoints: [0, 10, 20, 30, 40] },

		// Smaller parties: Results 8.8%, 5.0%, 4.3%
		FDP: { breakpoints: [0, 5, 10, 15, 20] },
		LINKE: { breakpoints: [0, 5, 10, 15, 20] },
		BSW: { breakpoints: [0, 5, 10, 15, 20] },
		OTHER: { breakpoints: [0, 5, 10, 15, 20] },
	};

	// Party colors (standard German party colours)
	const partyColors = {
		SPD: "#E3000F", // Red
		UNION: "#000000", // Black
		GRÜNE: "#1AA037", // Green
		FDP: "#FFED00", // Yellow
		AfD: "#0489DB", // Blue
		LINKE: "#BE3075", // Purple/Magenta
		BSW: "#7A3B2E", // Brown
		OTHER: "#BBBBBB", // Gray
	};

	// Check if device is mobile
	function checkMobile() {
		isMobile = window.innerWidth < 768;
	}

	// Set up the map on component mount
	onMount(async () => {
		// Check for mobile device
		checkMobile();
		window.addEventListener("resize", checkMobile);

		// Initialize map
		mapboxgl.accessToken = MAPBOX_TOKEN;
		map = new mapboxgl.Map({
			container: "map",
			style: "mapbox://styles/mapbox/light-v11",
			center: [10.4515, 51.1657], // Approximate centre of Germany
			zoom: 5.5,
			minZoom: 4,
			maxZoom: 10,
		});

		// Add navigation controls
		map.addControl(new mapboxgl.NavigationControl(), "bottom-right");

		// Add geocoder for searching locations
		const geocoder = new MapboxGeocoder({
			accessToken: MAPBOX_TOKEN,
			mapboxgl: mapboxgl,
			placeholder: "Search for locations",
			countries: "de",
			marker: false,
		});
		document.getElementById("geocoder").appendChild(geocoder.onAdd(map));

		map.on("load", async () => {
			// Move map labels above other map elements
			const layers = map.getStyle().layers;
			const labelLayerId = layers.find(
				(layer) => layer.type === "symbol" && layer.layout["text-field"]
			).id;

			// Add source for electoral districts
			map.addSource("election-districts", {
				type: "vector",
				tiles: [
					"https://map.jacobweinbren.workers.dev/germany-2025/{z}/{x}/{y}.mvt",
				],
				attribution: "2025 Bundestag Election Data",
			});

			// Add layer for leading party view (default)
			addLeadingPartyLayer(labelLayerId);

			// Set up event listeners
			setupEventListeners();
		});

		return () => {
			window.removeEventListener("resize", checkMobile);
		};
	});

	// Set up map event listeners
	function setupEventListeners() {
		// Hover effect
		map.on("mousemove", "districts-fill", (e) => {
			if (e.features.length > 0) {
				const feature = e.features[0];

				hoverInfo = {
					SPD: feature.properties.SPD,
					UNION: feature.properties.UNION,
					GRÜNE: feature.properties.GRÜNE,
					FDP: feature.properties.FDP,
					AfD: feature.properties.AfD,
					LINKE: feature.properties.LINKE,
					BSW: feature.properties.BSW,
					OTHER: feature.properties.OTHER,
					x: e.point.x,
					y: e.point.y,
				};

				map.getCanvas().style.cursor = "pointer";
			}
		});

		map.on("mouseleave", "districts-fill", () => {
			hoverInfo = null;
			map.getCanvas().style.cursor = "";
		});
	}

	// Add the leading party layer to the map
	function addLeadingPartyLayer(beforeLayerId) {
		if (map.getLayer("districts-fill")) map.removeLayer("districts-fill");
		if (map.getLayer("districts-line")) map.removeLayer("districts-line");

		map.addLayer(
			{
				id: "districts-fill",
				type: "fill",
				source: "election-districts",
				"source-layer": "election_districts",
				paint: {
					"fill-color": [
						"match",
						["string", getLeadingParty()],
						"SPD",
						partyColors.SPD,
						"UNION",
						partyColors.UNION,
						"GRÜNE",
						partyColors.GRÜNE,
						"FDP",
						partyColors.FDP,
						"AfD",
						partyColors.AfD,
						"LINKE",
						partyColors.LINKE,
						"BSW",
						partyColors.BSW,
						partyColors.OTHER,
					],
					"fill-opacity": 0.7,
				},
			},
			beforeLayerId
		);

		map.addLayer(
			{
				id: "districts-line",
				type: "line",
				source: "election-districts",
				"source-layer": "election_districts",
				paint: {
					"line-color": "#ffffff",
					"line-width": 0.5,
				},
			},
			beforeLayerId
		);
	}

	// Helper function to determine leading party
	function getLeadingParty() {
		return [
			"case",
			[
				">=",
				["get", "SPD"],
				[
					"max",
					["get", "UNION"],
					["get", "GRÜNE"],
					["get", "FDP"],
					["get", "AfD"],
					["get", "LINKE"],
					["get", "BSW"],
					["get", "OTHER"],
				],
			],
			"SPD",
			[
				">=",
				["get", "UNION"],
				[
					"max",
					["get", "GRÜNE"],
					["get", "FDP"],
					["get", "AfD"],
					["get", "LINKE"],
					["get", "BSW"],
					["get", "OTHER"],
				],
			],
			"UNION",
			[
				">=",
				["get", "GRÜNE"],
				[
					"max",
					["get", "FDP"],
					["get", "AfD"],
					["get", "LINKE"],
					["get", "BSW"],
					["get", "OTHER"],
				],
			],
			"GRÜNE",
			[
				">=",
				["get", "FDP"],
				[
					"max",
					["get", "AfD"],
					["get", "LINKE"],
					["get", "BSW"],
					["get", "OTHER"],
				],
			],
			"FDP",
			[
				">=",
				["get", "AfD"],
				["max", ["get", "LINKE"], ["get", "BSW"], ["get", "OTHER"]],
			],
			"AfD",
			[">=", ["get", "LINKE"], ["max", ["get", "BSW"], ["get", "OTHER"]]],
			"LINKE",
			[">=", ["get", "BSW"], ["get", "OTHER"]],
			"BSW",
			"OTHER",
		];
	}

	// Generate a color with specific saturation/lightness based on the base color
	function generateColorVariant(baseColor, saturation, lightness) {
		// For black (CDU/CSU), we need to use gray tones
		if (baseColor === "#000000") {
			// Create a smoother gradient from light to dark
			if (lightness >= 90) return "#F5F5F5"; // Very light gray
			if (lightness >= 70) return "#D3D3D3"; // Light gray
			if (lightness >= 50) return "#A9A9A9"; // Medium gray
			if (lightness >= 30) return "#696969"; // Dark gray
			return "#2D2D2D"; // Very dark gray (not pure black)
		}

		// Convert hex to RGB
		let r = parseInt(baseColor.slice(1, 3), 16);
		let g = parseInt(baseColor.slice(3, 5), 16);
		let b = parseInt(baseColor.slice(5, 7), 16);

		// Convert RGB to HSL
		r /= 255;
		g /= 255;
		b /= 255;

		let max = Math.max(r, g, b);
		let min = Math.min(r, g, b);
		let h,
			s,
			l = (max + min) / 2;

		if (max === min) {
			h = s = 0; // achromatic
		} else {
			let d = max - min;
			s = l > 0.5 ? d / (2 - max - min) : d / (max + min);

			switch (max) {
				case r:
					h = (g - b) / d + (g < b ? 6 : 0);
					break;
				case g:
					h = (b - r) / d + 2;
					break;
				case b:
					h = (r - g) / d + 4;
					break;
			}

			h /= 6;
		}

		// Adjust saturation and lightness
		s = saturation / 100;
		l = lightness / 100;

		// Convert back to RGB
		let r1, g1, b1;

		if (s === 0) {
			r1 = g1 = b1 = l; // achromatic
		} else {
			const hue2rgb = (p, q, t) => {
				if (t < 0) t += 1;
				if (t > 1) t -= 1;
				if (t < 1 / 6) return p + (q - p) * 6 * t;
				if (t < 1 / 2) return q;
				if (t < 2 / 3) return p + (q - p) * (2 / 3 - t) * 6;
				return p;
			};

			let q = l < 0.5 ? l * (1 + s) : l + s - l * s;
			let p = 2 * l - q;

			r1 = hue2rgb(p, q, h + 1 / 3);
			g1 = hue2rgb(p, q, h);
			b1 = hue2rgb(p, q, h - 1 / 3);
		}

		// Convert RGB back to hex
		const toHex = (x) => {
			const hex = Math.round(x * 255).toString(16);
			return hex.length === 1 ? "0" + hex : hex;
		};

		return `#${toHex(r1)}${toHex(g1)}${toHex(b1)}`;
	}

	// Add party vote share layer with quantile-based party colors
	function addPartyVoteShareLayer(party, beforeLayerId) {
		if (map.getLayer("districts-fill")) map.removeLayer("districts-fill");
		if (map.getLayer("districts-line")) map.removeLayer("districts-line");

		const baseColor = partyColors[party];
		const stats = partyStats[party];

		if (!stats) return;

		// Generate color variants with decreasing lightness/increasing saturation
		const color1 = generateColorVariant(baseColor, 15, 90);
		const color2 = generateColorVariant(baseColor, 40, 75);
		const color3 = generateColorVariant(baseColor, 70, 50);
		const color4 = generateColorVariant(baseColor, 90, 35);
		const color5 = generateColorVariant(baseColor, 100, 20);

		map.addLayer(
			{
				id: "districts-fill",
				type: "fill",
				source: "election-districts",
				"source-layer": "election_districts",
				paint: {
					"fill-color": [
						"step",
						["get", party],
						color1,
						stats.breakpoints[1],
						color2,
						stats.breakpoints[2],
						color3,
						stats.breakpoints[3],
						color4,
						stats.breakpoints[4],
						color5,
					],
					"fill-opacity": 0.9,
				},
			},
			beforeLayerId
		);

		map.addLayer(
			{
				id: "districts-line",
				type: "line",
				source: "election-districts",
				"source-layer": "election_districts",
				paint: {
					"line-color": "#ffffff",
					"line-width": 0.3,
				},
			},
			beforeLayerId
		);
	}

	// Toggle the party selection dropdown on mobile
	let showPartyMenu = false;
	function togglePartyMenu() {
		showPartyMenu = !showPartyMenu;
	}

	// Handle view change
	function handleViewChange(newValue) {
		selectedView = newValue;
		showPartyMenu = false; // Close menu after selection on mobile

		const layers = map.getStyle().layers;
		const labelLayerId = layers.find(
			(layer) => layer.type === "symbol" && layer.layout["text-field"]
		).id;

		if (selectedView === "leading") {
			addLeadingPartyLayer(labelLayerId);
		} else {
			addPartyVoteShareLayer(selectedView, labelLayerId);
		}
	}
</script>

<div id="map" class="absolute inset-0 w-full h-full"></div>

<!-- Geocoder (smaller on mobile) -->
<div
	id="geocoder"
	class="absolute top-4 left-4 z-10 max-w-[250px] md:max-w-[300px]"
></div>

<!-- Party selection - Mobile: dropdown button that expands -->
<div class="md:hidden absolute left-4 top-20 z-20">
	<button
		class="bg-white rounded-md shadow-md px-3 py-2 flex items-center gap-2 text-sm font-medium"
		on:click={togglePartyMenu}
	>
		{selectedView === "leading"
			? "Leading Party"
			: selectedView === "UNION"
				? "CDU/CSU"
				: selectedView === "GRÜNE"
					? "Grüne"
					: selectedView === "LINKE"
						? "Linke"
						: selectedView}
		<svg
			xmlns="http://www.w3.org/2000/svg"
			width="16"
			height="16"
			viewBox="0 0 24 24"
			fill="none"
			stroke="currentColor"
			stroke-width="2"
			stroke-linecap="round"
			stroke-linejoin="round"
		>
			<path d={showPartyMenu ? "M18 15l-6-6-6 6" : "M6 9l6 6 6-6"}></path>
		</svg>
	</button>

	{#if showPartyMenu}
		<div
			class="absolute left-0 top-12 bg-white rounded-md shadow-md p-2 w-40 mt-1 z-30"
		>
			<div class="space-y-1">
				<button
					class="block w-full text-left py-1.5 px-2 rounded text-sm {selectedView ===
					'leading'
						? 'bg-gray-200 font-medium'
						: 'hover:bg-gray-100'}"
					on:click={() => handleViewChange("leading")}
				>
					Leading Party
				</button>
				{#each ["SPD", "UNION", "GRÜNE", "FDP", "AfD", "LINKE", "BSW", "OTHER"] as party}
					<button
						class="block w-full text-left py-1.5 px-2 rounded text-sm {selectedView ===
						party
							? 'bg-gray-200 font-medium'
							: 'hover:bg-gray-100'}"
						on:click={() => handleViewChange(party)}
					>
						{party === "UNION"
							? "CDU/CSU"
							: party === "GRÜNE"
								? "Grüne"
								: party === "LINKE"
									? "Linke"
									: party}
					</button>
				{/each}
			</div>
		</div>
	{/if}
</div>

<!-- Party selection controls - Desktop: sidebar -->
<div
	class="hidden md:block absolute left-4 top-20 z-10 bg-white rounded-md shadow-md p-3 max-w-[200px]"
>
	<div class="text-sm font-bold mb-2">German Election Results</div>
	<div class="space-y-1.5 text-xs">
		<button
			class="block w-full text-left py-1 px-2 rounded {selectedView ===
			'leading'
				? 'bg-gray-200 font-medium'
				: 'hover:bg-gray-100'}"
			on:click={() => handleViewChange("leading")}
		>
			Leading Party
		</button>
		{#each ["SPD", "UNION", "GRÜNE", "FDP", "AfD", "LINKE", "BSW", "OTHER"] as party}
			<button
				class="block w-full text-left py-1 px-2 rounded {selectedView ===
				party
					? 'bg-gray-200 font-medium'
					: 'hover:bg-gray-100'}"
				on:click={() => handleViewChange(party)}
			>
				{party === "UNION"
					? "CDU/CSU"
					: party === "GRÜNE"
						? "Grüne"
						: party === "LINKE"
							? "Linke"
							: party}
			</button>
		{/each}
	</div>
</div>

<!-- Legend for selected view - Positioned at bottom for mobile, top-right for desktop -->
{#if selectedView !== "leading"}
	<div
		class="absolute bottom-16 left-0 right-0 mx-auto md:bottom-auto md:top-4 md:right-4 md:left-auto z-10 bg-white rounded-md shadow-md p-3 max-w-[260px]"
	>
		<div class="text-base font-bold">
			{selectedView === "UNION"
				? "CDU/CSU"
				: selectedView === "GRÜNE"
					? "Grüne"
					: selectedView === "LINKE"
						? "Linke"
						: selectedView}
		</div>

		<div class="flex items-center mt-2">
			<!-- Color legend with 5 steps -->
			<div class="flex h-6 w-full">
				{#if partyStats[selectedView]}
					<div
						class="w-10 h-full"
						style="background-color: {generateColorVariant(
							partyColors[selectedView],
							15,
							90
						)}"
					></div>
					<div
						class="w-10 h-full"
						style="background-color: {generateColorVariant(
							partyColors[selectedView],
							40,
							75
						)}"
					></div>
					<div
						class="w-10 h-full"
						style="background-color: {generateColorVariant(
							partyColors[selectedView],
							70,
							50
						)}"
					></div>
					<div
						class="w-10 h-full"
						style="background-color: {generateColorVariant(
							partyColors[selectedView],
							90,
							35
						)}"
					></div>
					<div
						class="w-10 h-full"
						style="background-color: {generateColorVariant(
							partyColors[selectedView],
							100,
							20
						)}"
					></div>
				{/if}
			</div>
		</div>

		<!-- Percentage labels - fixed alignment -->
		{#if partyStats[selectedView]}
			<div class="flex text-xs w-full">
				<div class="w-10 text-left">0%</div>
				<div class="w-10 text-center">
					{partyStats[selectedView].breakpoints[1]}%
				</div>
				<div class="w-10 text-center">
					{partyStats[selectedView].breakpoints[2]}%
				</div>
				<div class="w-10 text-center">
					{partyStats[selectedView].breakpoints[3]}%
				</div>
				<div class="w-10 text-right">
					{partyStats[selectedView].breakpoints[4]}%+
				</div>
			</div>
		{/if}
	</div>
{:else if selectedView === "leading"}
	<div
		class="absolute bottom-16 left-0 right-0 mx-auto md:bottom-auto md:top-4 md:right-4 md:left-auto z-10 bg-white rounded-md shadow-md p-3 max-w-[260px]"
	>
		<div class="text-base font-bold mb-2">Leading Party</div>
		<div class="grid grid-cols-2 gap-x-2 gap-y-2">
			{#each Object.entries(partyColors) as [party, color]}
				<div class="flex items-center gap-2">
					<div
						class="w-4 h-4 rounded-sm border border-gray-200"
						style="background-color: {color}"
					></div>
					<span class="text-xs">
						{party === "UNION"
							? "CDU/CSU"
							: party === "GRÜNE"
								? "Grüne"
								: party === "LINKE"
									? "Linke"
									: party}
					</span>
				</div>
			{/each}
		</div>
	</div>
{/if}

<!-- Hover info - adjusted for better mobile display -->
{#if hoverInfo && selectedView !== "leading"}
	<div
		class="pointer-events-none absolute z-20 bg-white p-2 rounded-md shadow-md text-sm max-w-[200px]"
		style="top: {Math.min(
			hoverInfo.y + 10,
			window.innerHeight - 80
		)}px; left: {Math.min(hoverInfo.x + 10, window.innerWidth - 220)}px;"
	>
		<div class="font-medium">
			{selectedView === "UNION"
				? "CDU/CSU"
				: selectedView === "GRÜNE"
					? "Grüne"
					: selectedView === "LINKE"
						? "Linke"
						: selectedView} Vote Share: {hoverInfo[selectedView]}%
		</div>
	</div>
{:else if hoverInfo && selectedView === "leading"}
	<div
		class="pointer-events-none absolute z-20 bg-white p-3 rounded-md shadow-md text-sm max-w-[200px]"
		style="top: {Math.min(
			hoverInfo.y + 10,
			window.innerHeight - 200
		)}px; left: {Math.min(hoverInfo.x + 10, window.innerWidth - 220)}px;"
	>
		<div class="space-y-1.5">
			{#each Object.entries( { AfD: hoverInfo.AfD, UNION: hoverInfo.UNION, LINKE: hoverInfo.LINKE, SPD: hoverInfo.SPD, BSW: hoverInfo.BSW, GRÜNE: hoverInfo.GRÜNE, OTHER: hoverInfo.OTHER, FDP: hoverInfo.FDP } ).sort((a, b) => b[1] - a[1]) as [party, value]}
				<div class="flex items-center">
					<div
						class="w-3 h-3 rounded-full mr-2"
						style="background-color: {partyColors[party]}"
					></div>
					<div class="w-[70px] text-sm font-medium">
						{party === "UNION"
							? "CDU/CSU"
							: party === "GRÜNE"
								? "Grüne"
								: party === "LINKE"
									? "Linke"
									: party}:
					</div>
					<div class="flex-1 text-right font-medium tabular-nums">
						{value.toFixed(2)}%
					</div>
				</div>
			{/each}
		</div>
	</div>
{/if}
