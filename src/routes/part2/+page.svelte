<script lang="ts">
	import AQICustomChart from '$lib/AQICustomChart.svelte';
	import * as d3 from 'd3';
	import { writable, derived} from 'svelte/store';

	const datasets = {
		avalon: 'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BAvalon%5D_daily-avg.csv',
		glassport_high_street:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BGlassport%20High%20Street%5D_daily-avg.csv',
		lawrenceville:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLawrenceville%5D_daily-avg.csv',
		liberty_sahs:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLiberty%20(SAHS)%5D_daily-avg.csv',
		manchester:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BManchester%5D_daily-avg.csv',
		north_braddock:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BNorth%20Braddock%5D_daily-avg.csv',
		parkway_east_near_road:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BParkway%20East%20(Near%20Road)%5D_daily-avg.csv',
		usa_pennsylvania_pittsburgh:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BUSA-Pennsylvania-Pittsburgh%5D_daily-avg.csv'
	};

	const selectedDataset = writable<keyof typeof datasets>('avalon');

	function parseRow(d: any) {
		return {
	 		city: d.City,
	 		country: d.Country,
	 		mainPollutant: d['Main pollutant'],
			usAqi: +d['US AQI'],
	 		state: d.State,
	 		stationName: d['Station name'],
	 		timestamp: new Date(d['Timestamp(UTC)']),
	 	};
	}

	let dataPromise: Promise<any[]> = d3.csv(datasets['avalon'], parseRow);
	let showRaw: boolean = false;

	$: if ($selectedDataset) {
	  // re-run fetch whenever selectedDataset changes
	  dataPromise = d3.csv(datasets[$selectedDataset], parseRow);
	}
</script>

<h1>AQI Data Visualization</h1>
<a href="/">
	<button type="button">Go to Part 1 Chart &larr;</button>
</a>

<h2>Controls</h2>
<p>Select a dataset and toggle raw data view:</p>
<label>
  Dataset:
	<select on:change={(e) => selectedDataset.set((e.target as HTMLSelectElement).value as keyof typeof datasets)}>
		{#each Object.keys(datasets) as key}
			<option value={key} selected={key === 'avalon'}>{key}</option>
		{/each}
	</select>
</label>

<label>
    Show Raw Data
    <input type="checkbox" bind:checked={showRaw} />
</label>

{#await dataPromise}
	<p>loading data...</p>
{:then data}
	<br>
	<h2>AQI Custom Chart (Part 2)</h2>
	<p>Want to look at the chart a little closer? Drag over an area you want to zoom in! Double click to reset zoom.</p>
	<p>Select 'Show Raw Data' and hover over each data point to see more details.</p>
	<p>Toggle each AQI level to view raw data points that fall within that level.</p>
	<AQICustomChart {data} {showRaw} />
{:catch error}
	<p>Something went wrong: {error.message}</p>
{/await}

<style>
    * {
        font-family: sans-serif;
        display: flex;
        flex-direction: column;
        align-items: flex-start;
    }
</style>
