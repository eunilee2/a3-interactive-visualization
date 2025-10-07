<script lang="ts">
	import AQIChart from '$lib/AQIChart.svelte';
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

	$: if ($selectedDataset) {
	  // re-run fetch whenever selectedDataset changes
	  dataPromise = d3.csv(datasets[$selectedDataset], parseRow);
	}
</script>

<h2>AQI Chart</h2>
<label>
  Dataset:
	<select on:change={(e) => selectedDataset.set((e.target as HTMLSelectElement).value as keyof typeof datasets)}>
		{#each Object.keys(datasets) as key}
			<option value={key} selected={key === 'avalon'}>{key}</option>
		{/each}
	</select>
</label>

{#await dataPromise}
	<p>loading data...</p>
{:then data}
	<AQIChart {data} />
{:catch error}
	<p>Something went wrong: {error.message}</p>
{/await}

<style>
	* {
		font-family: sans-serif;
	}
</style>
