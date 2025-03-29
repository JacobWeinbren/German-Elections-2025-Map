<script>
	export let district;
	export let partyStatistics;
	export let onClose;

	// Sort parties by vote share to get the results in descending order
	$: sortedParties = [
		{ name: "SPD", id: "SPD", value: district.SPD, color: "#E3000F" },
		{
			name: "CDU/CSU",
			id: "UNION",
			value: district.UNION,
			color: "#000000",
		},
		{ name: "Grüne", id: "GRÜNE", value: district.GRÜNE, color: "#1AA037" },
		{ name: "FDP", id: "FDP", value: district.FDP, color: "#FFED00" },
		{ name: "AfD", id: "AfD", value: district.AfD, color: "#0489DB" },
		{ name: "Linke", id: "LINKE", value: district.LINKE, color: "#BE3075" },
		{ name: "BSW", id: "BSW", value: district.BSW, color: "#7A3B2E" },
		{ name: "Other", id: "OTHER", value: district.OTHER, color: "#BBBBBB" },
	].sort((a, b) => b.value - a.value);

	// Determine the leading party
	$: leadingParty = sortedParties[0];

	// Calculate bar width based on party's position in the nationwide distribution
	function getBarWidth(party) {
		const stats = partyStatistics[party.id];
		if (!stats) return "0%";

		// Calculate relative position based on quantiles
		if (party.value <= stats.q25) {
			return (
				Math.max(
					10,
					((party.value - stats.min) / (stats.q25 - stats.min)) * 25
				) + "%"
			);
		} else if (party.value <= stats.median) {
			return (
				25 +
				((party.value - stats.q25) / (stats.median - stats.q25)) * 25 +
				"%"
			);
		} else if (party.value <= stats.q75) {
			return (
				50 +
				((party.value - stats.median) / (stats.q75 - stats.median)) *
					25 +
				"%"
			);
		} else {
			return (
				Math.min(
					100,
					75 +
						((party.value - stats.q75) / (stats.max - stats.q75)) *
							25
				) + "%"
			);
		}
	}
</script>

<div
	class="fixed bottom-6 left-1/2 transform -translate-x-1/2 bg-white rounded-lg shadow-lg p-5 z-30 w-[450px] max-w-[90vw]"
>
	<div class="flex justify-between items-center mb-4">
		<h3 class="text-xl font-bold">Postal Code: {district.plz}</h3>
		<button
			class="text-gray-500 hover:text-gray-700 transition"
			on:click={onClose}
		>
			<svg
				class="w-5 h-5"
				fill="none"
				stroke="currentColor"
				viewBox="0 0 24 24"
			>
				<path
					stroke-linecap="round"
					stroke-linejoin="round"
					stroke-width="2"
					d="M6 18L18 6M6 6l12 12"
				></path>
			</svg>
		</button>
	</div>

	<h4 class="text-sm font-medium mb-4">
		Party Vote Shares - 2025 Bundestag Election
	</h4>

	<div class="space-y-3">
		{#each sortedParties as party}
			<div class="flex items-center gap-3">
				<div class="w-20 text-sm font-medium">{party.name}</div>
				<div
					class="flex-1 bg-gray-100 rounded-full h-6 overflow-hidden"
				>
					<div
						class="h-full rounded-full transition-all duration-300 flex items-center justify-end pr-1"
						style="width: {getBarWidth(
							party
						)}; background-color: {party.color}"
					>
						{#if party.value > 5}
							<span
								class="text-xs font-bold {party.id ===
									'UNION' || party.id === 'FDP'
									? 'text-white'
									: ''}">{party.value}%</span
							>
						{/if}
					</div>
				</div>
				<div class="w-12 text-sm font-medium text-right">
					{party.value}%
				</div>
			</div>
		{/each}
	</div>

	<div class="mt-4 pt-3 border-t border-gray-200 text-xs text-gray-600">
		<div>
			Bar widths represent relative performance on a national quantile
			scale (25-75%)
		</div>
		<div class="text-xs italic mt-1">
			2025 Bundestag Election - Second Vote (Zweitstimme)
		</div>
	</div>
</div>
