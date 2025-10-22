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
	const selectedSecondaryDataset = writable<keyof typeof datasets | null>(null);

	function parseRow(d: any) {
		return {
	 		city: d.City,
	 		country: d.Country,
	 		mainPollutant: d['Main pollutant'],
	 		state: d.State,
	 		stationName: d['Station name'],
	 		timestamp: new Date(d['Timestamp(UTC)']),
	 		usAqi: +d['US AQI']
	 	};
	}

	let dataPromise: Promise<any[]> = d3.csv(datasets['avalon'], parseRow);
	let secondaryDataPromise: Promise<any[]> | null = null;
	let showRaw: boolean = false;

	$: if ($selectedDataset) {
		// re-run fetch whenever selectedDataset changes
		dataPromise = d3.csv(datasets[$selectedDataset], parseRow);
	}

	$: if ($selectedSecondaryDataset) {
		// re-run fetch whenever selectedSecondaryDataset changes
		secondaryDataPromise = d3.csv(datasets[$selectedSecondaryDataset], parseRow);
	} else {
		secondaryDataPromise = null;
	}
</script>

<h1>AQI Data Visualization</h1>
<a href="/">
	<button type="button">Go to Part 1 Chart &larr;</button>
</a>

<a href="/report">
	<button type="button">Go to the Report Writeup &rarr;</button>
</a>

<h2>Controls</h2>
<div class="controls-container">
    <div class="primary-controls">
        <label class="control-label">
            Primary Dataset:
            <select on:change={(e) => selectedDataset.set((e.target as HTMLSelectElement).value as keyof typeof datasets)}>
                {#each Object.keys(datasets) as key}
                    <option value={key} selected={key === 'avalon'}>{key}</option>
                {/each}
            </select>
        </label>

        <label class="control-label raw-data">
            Show Raw Data (Primary)
            <input type="checkbox" bind:checked={showRaw} />
        </label>
    </div>

    <div class="secondary-controls">
        <label class="control-label">
            Compare with:
            <select on:change={(e) => {
                const value = (e.target as HTMLSelectElement).value;
                selectedSecondaryDataset.set(value === '' ? null : value as keyof typeof datasets);
            }}>
                <option value="">None</option>
                {#each Object.keys(datasets) as key}
                    {#if key !== $selectedDataset}
                        <option value={key}>{key}</option>
                    {/if}
                {/each}
            </select>
        </label>
    </div>
</div>

{#await Promise.all([dataPromise, secondaryDataPromise || Promise.resolve(null)])}
	<p>loading data...</p>
{:then [data, secondaryData]}
	<br>
	<h2>AQI Custom Chart (Part 2)</h2>
	<p>Want to look at the chart a little closer? Drag over an area you want to zoom in! Double click to reset zoom.</p>
	<p>Select 'Show Raw Data' and hover over each data point to see more details.</p>
	<p>Toggle each AQI level to view raw data points that fall within that level.</p>
	<p>A secondary dataset can be compared against the primary dataset. See the legend for encoding details.</p>
	<AQICustomChart 
		{data} 
		secondaryData={secondaryData || []} 
		{showRaw}
		showSecondary={!!secondaryData} 
	/>
{:catch error}
	<p>Something went wrong: {error.message}</p>
{/await}

<style>
    :global(*) {
        font-family: sans-serif;
    }

    h1, h2, p {
        margin-bottom: 1rem;
    }

    .controls-container {
        display: grid;
        grid-template-columns: minmax(30vw, 1fr) minmax(30vw, 1fr);
        gap: 2rem;
        width: 100%;
        max-width: 800px;
        margin-bottom: 2rem;
    }

    .primary-controls,
    .secondary-controls {
        display: flex;
        flex-direction: column;
        gap: 1rem;
        width: 100%;
        padding: 1rem;
        background: #f8f8f8;
        border-radius: 8px;
        min-height: 120px; /* Ensure minimum height for visual balance */
    }

    .control-label {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
        font-weight: bold;
        width: 100%;
    }

    .raw-data {
        flex-direction: row;
        align-items: center;
        gap: 0.5rem;
    }

    select {
        padding: 0.5rem;
        border-radius: 4px;
        border: 1px solid #ccc;
        font-size: 1rem;
        width: 100%;
        min-width: 200px;
    }

    input[type="checkbox"] {
        width: 1.2rem;
        height: 1.2rem;
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
