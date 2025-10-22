<script lang="ts">
	import AQIChart from '$lib/AQIChart.svelte';
	import AQITimeSeries from '$lib/AQITimeSeries.svelte';
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
	 		pm25: +d['PM2.5'],
	 		state: d.State,
	 		stationName: d['Station name'],
	 		timestamp: new Date(d['Timestamp(UTC)']),
	 		usAqi: +d['US AQI']
	 	};
	}

	let dataPromise: Promise<any[]> = d3.csv(datasets['avalon'], parseRow);
	let showRaw: boolean = false;

	$: if ($selectedDataset) {
	  // re-run fetch whenever selectedDataset changes
	  dataPromise = d3.csv(datasets[$selectedDataset], parseRow);
	}
</script>

<div class="page-container">
	<h1>AQI Data Visualization</h1>
	<a href="/part2">
		<button type="button">Go to Part 2 Chart &rarr;</button>
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
		<!-- <AQIChart {data} /> -->
		<p>Showing <strong>{data.length}</strong> records</p>
		<h2>AQI Chart (Part 1)</h2>
		<AQITimeSeries {data} {showRaw} />
	{:catch error}
		<p>Something went wrong: {error.message}</p>
	{/await}
</div>

<style>
    .page-container {
        font-family: sans-serif;
        display: flex;
        flex-direction: column;
        align-items: flex-start;
    }
	button {
        padding: 0.5rem 1rem;
        border-radius: 4px;
        border: 1px solid #ccc;
        background-color: white;
        cursor: pointer;
        font-size: 1rem;
        margin-bottom: 2rem;
    }

    button:hover {
        background-color: #f0f0f0;
    }
</style>
