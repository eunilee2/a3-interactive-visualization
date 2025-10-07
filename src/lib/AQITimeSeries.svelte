<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import * as d3 from 'd3';

	// Expect aggregated monthly data: { date: Date, avg: number, p10: number, p90: number, count?: number }
	// If passing raw rows, they should have `timestamp` and `usAqi`/`pm25` fields.
	export let data: any[] = [];
	// showRaw: whether to render raw datapoints (only applicable if `data` is raw rows)
	export let showRaw: boolean = false;

	let container: HTMLDivElement;
	let ro: ResizeObserver | null = null;

	function draw() {
		if (!container) return;
		// clear
		d3.select(container).selectAll('*').remove();

		const margin = { top: 20, right: 120, bottom: 60, left: 60 };
		const width = Math.max(600, container.clientWidth || 800);
		const height = 420;

		const svg = d3
			.select(container)
			.append('svg')
			.attr('width', width)
			.attr('height', height)
			.attr('viewBox', `0 0 ${width} ${height}`)
			.attr('preserveAspectRatio', 'xMidYMid meet');

		const innerWidth = width - margin.left - margin.right;
		const innerHeight = height - margin.top - margin.bottom;

		const g = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`);

		if (!data || data.length === 0) {
			g.append('text').attr('x', innerWidth / 2).attr('y', innerHeight / 2).attr('text-anchor', 'middle').text('No data');
			return;
		}

		// support both aggregated or raw shapes
		const parseDate = (d: any) => d.date ?? d.timestamp ?? null;

		function alignTo15(d: Date) {
			return new Date(Date.UTC(d.getUTCFullYear(), d.getUTCMonth(), 15));
		}

		// if rows look like raw data (have timestamp/usAqi), aggregate to monthly buckets here
		let series: any[] = [];
		// keep original raw rows (if present) to optionally render points
		let rawRows: any[] | null = null;
		if (data && data.length > 0 && (data[0].timestamp || data[0].usAqi)) {
			// treat as raw rows
			rawRows = data
				.map((d: any) => ({
					timestamp: d.timestamp ? new Date(d.timestamp) : null,
					value: d.usAqi != null ? +d.usAqi : (d['US AQI'] != null ? +d['US AQI'] : (d.pm25 != null ? +d.pm25 : NaN))
				}))
				.filter((r: any) => r.timestamp && !isNaN(r.value));

			const grouped = d3.group(rawRows, (r: any) => alignTo15(r.timestamp).getTime());

			series = Array.from(grouped, ([k, vals]) => {
				const values = vals.map((v: any) => v.value).sort(d3.ascending);
				const count = values.length;
				return {
					date: new Date(Number(k)),
					count,
					avg: count ? d3.mean(values) : NaN,
					p10: count ? d3.quantileSorted(values, 0.1) : NaN,
					p90: count ? d3.quantileSorted(values, 0.9) : NaN
				};
			}).sort((a: any, b: any) => a.date.getTime() - b.date.getTime());
		} else {
			// assume already aggregated
			series = data.map((d: any) => ({
				date: new Date(parseDate(d)),
				avg: d.avg ?? d.usAqi ?? d.pm25,
				p10: d.p10 ?? d.avg ?? d.usAqi ?? d.pm25,
				p90: d.p90 ?? d.avg ?? d.usAqi ?? d.pm25,
				count: d.count ?? 0
			})).filter((d: any) => d.date && !isNaN(d.avg));
		}

		// debug: show a sample of computed series values (check p10/p90)
		// console.log('AQITimeSeries series sample:', series.slice(0, 10));

		// AQI levels to show as background bands and in the legend
		const aqiLevels = [
			{ name: 'Good', min: 0, max: 50, color: '#9cd84e' },
			{ name: 'Moderate', min: 51, max: 100, color: '#facf39' },
			{ name: 'Unhealthy for Sensitive Groups', min: 101, max: 150, color: '#f99049' },
			{ name: 'Unhealthy', min: 151, max: 200, color: '#f65e5f' },
			{ name: 'Very Unhealthy', min: 201, max: 300, color: '#a070b6' },
			{ name: 'Hazardous', min: 301, color: '#a06a7b' }
		];

	const x = d3.scaleTime().domain(d3.extent(series, d => d.date) as [Date, Date]).range([0, innerWidth]).nice();
	const yMax = d3.max(series, d => d.p90 ?? d.avg) ?? 0;

	// compute adaptive y-domain: prefer to cap near the data peak (p90) so the chart isn't dominated
	// by AQI level maxima that are far above actual data. Strategy:
	// - dataPeak = p90 peak
	// - marginAbs = 50 (absolute fallback), marginPct = 0.2 * dataPeak
	// - dataCap = dataPeak + max(marginAbs, marginPct)
	// - if the nearest levelMax is within 15% of dataCap, include levelMax; otherwise use dataCap
	const dataPeak = yMax;
	const marginAbs = 50;
	const marginPct = dataPeak * 0.2;
	const dataCap = dataPeak + Math.max(marginAbs, marginPct);

	const levelMax = d3.max(aqiLevels, (l: any) => l.max ?? 0) ?? 0;
	// if the AQI level max is reasonably close to the dataCap, keep it; else favor the dataCap
	const includeLevelMax = levelMax > 0 && (Math.abs(levelMax - dataCap) / Math.max(levelMax, 1) <= 0.15 || levelMax <= dataCap);

	const yDomainMax = includeLevelMax ? Math.min(dataCap, levelMax) : Math.min(dataCap, 350);
	const y = d3.scaleLinear().domain([0, yDomainMax]).range([innerHeight, 0]).nice();

		const xAxis = d3.axisBottom(x)
			.ticks(d3.timeMonth.every(1))
			.tickFormat(((d: Date | number, i: number) => {
				const date = d instanceof Date ? d : new Date(d);
				return d3.timeFormat('%b %Y')(date);
			}) as any);
		const yAxis = d3.axisLeft(y).ticks(5);

		g.append('g').attr('class', 'y axis').call(yAxis);

		g.append('g')
			.attr('class', 'x axis')
			.attr('transform', `translate(0,${innerHeight})`)
			.call(xAxis)
			.selectAll('text')
			.attr('transform', 'rotate(-35)')
			.style('text-anchor', 'end');


		// AQI background bands (draw before plotting data) - only draw visible levels
		const levelsGroup = g.append('g').attr('class', 'aqi-levels');
		const visibleLevels: any[] = [];
		const dataMin = d3.min(series, d => d.p10 ?? d.avg) ?? 0;
		const dataMax = d3.max(series, d => d.p90 ?? d.avg) ?? 0;
		const marginValue = (dataMax - dataMin) * 0.1 || 5;
		const thresholdPx = 10; // minimum pixel height to draw a band if it doesn't intersect data

		aqiLevels.forEach((lvl: any) => {
			const maxVal = lvl.max ?? yDomainMax; // extend to top if no max
			const top = y(maxVal);
			const bottom = y(lvl.min);
			const topClamped = Math.min(innerHeight, Math.max(0, top));
			const bottomClamped = Math.min(innerHeight, Math.max(0, bottom));
			const pixelHeight = bottomClamped - topClamped;

			// check whether this level intersects the data range (with small margin)
			const intersectsData = !(lvl.max !== undefined ? (lvl.max < (dataMin - marginValue) || lvl.min > (dataMax + marginValue)) : (lvl.min > (dataMax + marginValue)));

			if (intersectsData || pixelHeight >= thresholdPx) {
				visibleLevels.push(lvl);
				if (pixelHeight > 0.5) {
					levelsGroup.append('rect')
						.attr('x', 0)
						.attr('y', topClamped)
						.attr('width', innerWidth)
						.attr('height', pixelHeight)
						.attr('fill', lvl.color)
						.attr('opacity', 0.25);
				}
			}
		});

		// area band
		const area = d3.area()
			.x((d: any) => x(d.date))
			.y0((d: any) => y(d.p10))
			.y1((d: any) => y(d.p90))
			.curve(d3.curveMonotoneX as any);

		g.append('path')
			.datum(series)
			.attr('class', 'band')
			.attr('d', area as any)
			.attr('fill', '#aeaeae')
			.attr('opacity', 0.5);

		// avg line
		const line = d3.line()
			.x((d: any) => x(d.date))
			.y((d: any) => y(d.avg))
			.curve(d3.curveMonotoneX as any);

		g.append('path')
			.datum(series)
			.attr('class', 'line')
			.attr('d', line as any)
			.attr('fill', 'none')
			.attr('stroke', '#1f77b4')
			.attr('stroke-width', 2);

		// raw points (if available and enabled)
		if (showRaw && rawRows && rawRows.length > 0) {
			console.log('AQITimeSeries: showRaw=', showRaw, 'rawRows=', rawRows.length);
			// create a tooltip container
			// cast selection to any to avoid TS selection type mismatch
			let tooltip: any = (d3.select(container).select('.raw-tooltip') as any);
			if (!tooltip || tooltip.empty()) {
				tooltip = (d3.select(container) as any)
					.append('div')
					.attr('class', 'raw-tooltip')
					.style('position', 'absolute')
					.style('pointer-events', 'none')
					.style('background', 'rgba(0,0,0,0.7)')
					.style('color', '#fff')
					.style('padding', '4px 6px')
					.style('border-radius', '3px')
					.style('font-size', '12px')
					.style('display', 'none');
			}

			const pts = g.append('g').attr('class', 'raw-points');
			pts.selectAll('circle')
				.data(rawRows)
				.join('circle')
				.attr('cx', (d: any) => x(d.timestamp))
				.attr('cy', (d: any) => y(d.value))
				.attr('r', 3.5)
				.attr('fill', '#222')
				.attr('opacity', 0.6)
				.on('mouseover', function (event: any, d: any) {
					tooltip.style('display', 'block').text(`${d.timestamp.toISOString().slice(0, 16).replace('T',' ')} — ${d.value}`);
				})
				.on('mousemove', function (event: any) {
					// position tooltip relative to container using client coordinates
					const rect = container.getBoundingClientRect();
					const left = event.clientX - rect.left + 12;
					const top = event.clientY - rect.top + 12;
					tooltip.style('left', left + 'px').style('top', top + 'px');
				})
				.on('mouseout', function () {
					tooltip.style('display', 'none');
				});
		}

		// legend (top-right)
		const legend = svg.append('g').attr('transform', `translate(${width - margin.right + 10},${margin.top})`);
		let legendY = 0;
		// show only levels we rendered
		visibleLevels.forEach((lvl: any) => {
			const label = lvl.max ? `${lvl.name} (${lvl.min}–${lvl.max})` : `${lvl.name} (${lvl.min}+)`;
			legend.append('rect').attr('x', 0).attr('y', legendY).attr('width', 12).attr('height', 12).attr('fill', lvl.color).attr('opacity', 0.9);
			legend.append('text').attr('x', 18).attr('y', legendY + 10).text(label).attr('font-size', 12).attr('alignment-baseline', 'middle');
			legendY += 18;
		});

		// spacer
		legendY += 6;
		legend.append('rect').attr('x', 0).attr('y', legendY).attr('width', 12).attr('height', 12).attr('fill', '#aeaeae').attr('opacity', 0.5);
		legend.append('text').attr('x', 18).attr('y', legendY + 10).text('10%-90% band').attr('font-size', 12).attr('alignment-baseline', 'middle');
		legendY += 18;
		legend.append('line').attr('x1', 0).attr('y1', legendY + 6).attr('x2', 12).attr('y2', legendY + 6).attr('stroke', '#1f77b4').attr('stroke-width', 2);
		legend.append('text').attr('x', 18).attr('y', legendY + 10).text('Monthly avg').attr('font-size', 12).attr('alignment-baseline', 'middle');
	}

	onMount(() => {
		draw();
		ro = new ResizeObserver(() => draw());
		if (container) ro.observe(container);
	});

	onDestroy(() => {
		if (ro && container) ro.disconnect();
	});

// re-draw when inputs change (data or showRaw). This makes the checkbox reactive.
$: if (container) {
    // depend on data and showRaw so Svelte will run this when they change
    // eslint-disable-next-line @typescript-eslint/no-unused-expressions
    data, showRaw; 
    draw();
}
</script>

<div bind:this={container} class="aqi-timeseries" style="width:100%; position:relative"></div>

<style>
	:global(.band) { fill: #aeaeae; opacity: 0.5; }
	:global(.line) { stroke: rgb(0, 20, 36); fill: none; stroke-width: 1.5px; }
	/* keep global styles minimal to avoid "unused selector" warnings */
	:global(svg) { font-family: sans-serif; }
</style>
