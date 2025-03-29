<script>
	import { createEventDispatcher } from "svelte";
	import Select from "svelte-select";

	export let selectedView;
	export let partyColors;
	export let partyStatistics;

	const dispatch = createEventDispatcher();

	const viewOptions = [
		{ value: "leading", label: "Leading Party" },
		{ value: "SPD", label: "SPD Vote Share" },
		{ value: "UNION", label: "CDU/CSU Vote Share" },
		{ value: "GRÜNE", label: "Grüne Vote Share" },
		{ value: "FDP", label: "FDP Vote Share" },
		{ value: "AfD", label: "AfD Vote Share" },
		{ value: "LINKE", label: "Linke Vote Share" },
		{ value: "BSW", label: "BSW Vote Share" },
		{ value: "OTHER", label: "Other Parties" },
	];

	function handleChange(event) {
		if (event.detail) {
			dispatch("changeView", event.detail.value);
		}
	}

	$: selectedOption = viewOptions.find(
		(option) => option.value === selectedView
	);
</script>

<div
	class="absolute top-4 right-16 z-10 bg-white p-5 rounded-md shadow-md w-72"
>
	<h1 class="text-lg font-bold mb-3">German Election Results</h1>
	<p class="text-xs text-gray-600 mb-3">
		2025 Bundestag Election - Second Vote
	</p>

	<div class="mb-4">
		<Select
			items={viewOptions}
			value={selectedOption}
			on:change={handleChange}
			containerStyles="z-20"
		/>
	</div>

	{#if selectedView === "leading"}
		<div>
			<h3 class="text-sm font-medium mb-2">Leading Party</h3>
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
	{:else if partyStatistics[selectedView]}
		<div>
			<h3 class="text-sm font-medium mb-2">
				{selectedView === "UNION"
					? "CDU/CSU"
					: selectedView === "GRÜNE"
						? "Grüne"
						: selectedView === "LINKE"
							? "Linke"
							: selectedView} Vote Share
			</h3>
			<div
				class="w-full h-6 rounded-sm mb-1"
				style="background: linear-gradient(to right, white, {partyColors[
					selectedView
				]})"
			></div>
			<div class="flex justify-between text-xs text-gray-600">
				<span>{partyStatistics[selectedView].q25}%</span>
				<span>{partyStatistics[selectedView].median}%</span>
				<span>{partyStatistics[selectedView].q75}%</span>
			</div>
			<div class="mt-3 text-xs text-gray-500">
				Colors represent quantiles (25th-75th percentile) of nationwide
				support
			</div>
		</div>
	{/if}
</div>
