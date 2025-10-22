<script lang="ts">
    import { onMount, onDestroy } from 'svelte';
    import * as d3 from 'd3';

    export let data: any[] = [];
    export let secondaryData: any[] = [];
    export let showRaw: boolean = false;
    export let showSecondary: boolean = true;

    function formatStationName(name: string | undefined, city?: string, state?: string, country?: string): string {
        if (name) {
            // Special case for USA
            return name
                .split('_')
                .map(word => {
                    if (word.toLowerCase() === 'usa') return 'USA';
                    return word.charAt(0).toUpperCase() + word.slice(1).toLowerCase();
                })
                .join(' ');
        }
        
        // If no name but we have location data, construct a name
        if (city || state || country) {
            const parts = [];
            if (city) parts.push(city.charAt(0).toUpperCase() + city.slice(1).toLowerCase());
            if (state) parts.push(state.charAt(0).toUpperCase() + state.slice(1).toLowerCase());
            if (country) {
                parts.push(country.toLowerCase() === 'usa' ? 'USA' : 
                          country.charAt(0).toUpperCase() + country.slice(1).toLowerCase());
            }
            return parts.join(', ');
        }
        
        return "Unknown Station";
        // Special case for USA
        // return name
        //     .split('_')
        //     .map(word => {
        //         if (word.toLowerCase() === 'usa') return 'USA';
        //         return word.charAt(0).toUpperCase() + word.slice(1).toLowerCase();
        //     })
        //     .join(' ');
    }

    let container: HTMLDivElement;
    let ro: ResizeObserver | null = null;

    const aqiLevels = [
        { name: 'Good', min: 0, max: 50, color: '#9cd84e' },
        { name: 'Moderate', min: 51, max: 100, color: '#facf39' },
        { name: 'Unhealthy for Sensitive Groups', min: 101, max: 150, color: '#f99049' },
        { name: 'Unhealthy', min: 151, max: 200, color: '#f65e5f' },
        { name: 'Very Unhealthy', min: 201, max: 300, color: '#a070b6' },
        { name: 'Hazardous', min: 301, color: '#a06a7b' }
    ];

    let selectedLevel: number | null = null; // min of selected level, or null for all

    let currentXDomain: [Date, Date] | null = null;

    function draw() {
        if (!container) return;
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

        const clipId = `clip-${Math.floor(Math.random() * 1e9)}`;
        svg.append('defs').append('clipPath').attr('id', clipId).append('rect').attr('width', innerWidth).attr('height', innerHeight);

        if (!data || data.length === 0) {
            g.append('text').attr('x', innerWidth / 2).attr('y', innerHeight / 2).attr('text-anchor', 'middle').text('No data');
            return;
        }

        const parseDate = (d: any) => d.date ?? d.timestamp ?? null;
        function alignTo15(d: Date) {
            return new Date(Date.UTC(d.getUTCFullYear(), d.getUTCMonth(), 15));
        }

        let series: any[] = [];
        let rawRows: any[] | null = null;
        if (data && data.length > 0 && (data[0].timestamp || data[0].usAqi)) {
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
            series = data.map((d: any) => ({
                date: new Date(parseDate(d)),
                avg: d.avg ?? d.usAqi ?? d.pm25,
                p10: d.p10 ?? d.avg ?? d.usAqi ?? d.pm25,
                p90: d.p90 ?? d.avg ?? d.usAqi ?? d.pm25,
                count: d.count ?? 0
            })).filter((d: any) => d.date && !isNaN(d.avg));
        }

        // --- FILTER DATA BASED ON SELECTED LEVEL ---
		let filteredSeries = series;
		let filteredRawRows = rawRows;
		if (selectedLevel !== null) {
			const level = aqiLevels.find(l => l.min === selectedLevel);
			if (level) {
				const max = level.max ?? Infinity;
				const tempSeries = series.filter(d => d.avg >= level.min && d.avg <= max);
				const tempRawRows = rawRows ? rawRows.filter(d => d.value >= level.min && d.value <= max) : null;
				// If there is data in this level, use the filtered data; otherwise, show all data
				if (tempSeries.length > 0) {
					filteredSeries = tempSeries;
					filteredRawRows = tempRawRows;
				} else {
					filteredSeries = series;
					filteredRawRows = rawRows;
				}
			}
		}

        const fullXExtent = d3.extent(filteredSeries, d => d.date) as [Date, Date];
        if (currentXDomain) {
            const [c0, c1] = currentXDomain;
            const n0 = c0 < fullXExtent[0] ? fullXExtent[0] : c0;
            const n1 = c1 > fullXExtent[1] ? fullXExtent[1] : c1;
            currentXDomain = [n0, n1];
        }
        const x = d3.scaleTime().domain(currentXDomain ?? fullXExtent).range([0, innerWidth]);
        const yMax = d3.max(filteredSeries, d => d.p90 ?? d.avg) ?? 0;

        const dataPeak = yMax;
        const marginAbs = 50;
        const marginPct = dataPeak * 0.2;
        const dataCap = dataPeak + Math.max(marginAbs, marginPct);

        const levelMax = d3.max(aqiLevels, (l: any) => l.max ?? 0) ?? 0;
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

        g.append('g').attr('class', 'y axis')
		.call(yAxis)
		// Y-axis label
		.append('text')
		.attr('class', 'axis-label')
		.attr('transform', `rotate(-90)`)
		.attr('y', -50)
		.attr('x', -innerHeight / 2)
		.attr('dy', '1em')
		.attr('fill', '#222')
		.attr('text-anchor', 'middle')
		.text('AQI');

		// .rotate labels
		g.append('g')
			.attr('class', 'x axis')
			.attr('transform', `translate(0,${innerHeight})`)
			.call(xAxis)
			.selectAll('text')
			.attr('transform', 'rotate(-35)')
			.style('text-anchor', 'end');

		g.append('g')
            .attr('class', 'x axis')
            .attr('transform', `translate(0,${innerHeight})`)
			.append('text')
			.attr('class', 'axis-label')
			.attr('x', innerWidth / 2)
			.attr('y', 55)
			.attr('fill', '#222')
			.attr('text-anchor', 'middle')
			.text('Date');

        let brush: any;
        let brushG: any;

        function brushed(event: any) {
            if (!event.selection) return;
            const [x0, x1] = event.selection;
            if (Math.abs(x1 - x0) < 2) {
                if (brushG) (brushG as any).call(brush.move, null);
                return;
            }
            const d0 = x.invert(x0);
            const d1 = x.invert(x1);
            currentXDomain = [d0, d1];
            if (brushG) (brushG as any).call(brush.move, null);
            draw();
        }

        svg.on('dblclick', () => {
            currentXDomain = null;
            draw();
        });

        // --- DRAW BANDS, HIGHLIGHT SELECTED ---
        const levelsGroup = g.append('g').attr('class', 'aqi-levels').attr('clip-path', `url(#${clipId})`);
        const visibleLevels: any[] = [];
        const dataMin = d3.min(filteredSeries, d => d.p10 ?? d.avg) ?? 0;
        const dataMax = d3.max(filteredSeries, d => d.p90 ?? d.avg) ?? 0;
        const marginValue = (dataMax - dataMin) * 0.1 || 5;
        const thresholdPx = 10;

        aqiLevels.forEach((lvl: any) => {
            const maxVal = lvl.max ?? yDomainMax;
            const top = y(maxVal);
            const bottom = y(lvl.min);
            const topClamped = Math.min(innerHeight, Math.max(0, top));
            const bottomClamped = Math.min(innerHeight, Math.max(0, bottom));
            const pixelHeight = bottomClamped - topClamped;

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
                        .attr('opacity', selectedLevel === lvl.min ? 0.5 : 0.25)
                        .attr('stroke', selectedLevel === lvl.min ? '#222' : 'none')
                        .attr('stroke-width', selectedLevel === lvl.min ? 2 : 0);
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
            .attr('opacity', 0.5)
            .attr('clip-path', `url(#${clipId})`);

        // avg line for primary data
        const line = d3.line()
            .x((d: any) => x(d.date))
            .y((d: any) => y(d.avg))
            .curve(d3.curveMonotoneX as any);

        g.append('path')
            .datum(series)
            .attr('class', 'line primary')
            .attr('d', line as any)
            .attr('fill', 'none')
            .attr('stroke', '#1f77b4')
            .attr('stroke-width', 2)
            .attr('clip-path', `url(#${clipId})`);

            // Create legend group
		const legendGroup = g.append('g')
            .attr('class', 'legend')
            .attr('transform', `translate(${innerWidth + 10}, 0)`);

        // Legend sections title
        const stationsTitle = legendGroup.append('text')
            .attr('x', 0)
            .attr('y', -10)
            .attr('fill', '#222')
            .style('font-size', '12px')
            .style('font-weight', 'bold')
            .text('Data Type');

        // Add visualization type legend (area band and line average)
        const visualTypeGroup = legendGroup.append('g')
            .attr('transform', 'translate(0, 15)');

        // Area band legend
        visualTypeGroup.append('rect')
            .attr('x', 0)
            .attr('y', -8)
            .attr('width', 20)
            .attr('height', 16)
            .attr('fill', '#aeaeae')
            .attr('opacity', 0.5);

        visualTypeGroup.append('text')
            .attr('x', 25)
            .attr('y', 0)
            .attr('fill', '#222')
            .style('font-size', '12px')
            .text('10%-90% band');

        // Line average legend
        const lineAvgGroup = legendGroup.append('g')
            .attr('transform', 'translate(0, 40)');

        lineAvgGroup.append('line')
            .attr('x1', 0)
            .attr('y1', 0)
            .attr('x2', 20)
            .attr('y2', 0)
            .attr('stroke', '#1f77b4')
            .attr('stroke-width', 2);

        lineAvgGroup.append('text')
            .attr('x', 25)
            .attr('y', 4)
            .attr('fill', '#222')
            .style('font-size', '12px')
            .text('Monthly average');

        // Stations section title
        legendGroup.append('text')
            .attr('x', 0)
            .attr('y', 70)
            .attr('fill', '#222')
            .style('font-size', '12px')
            .style('font-weight', 'bold')
            .text('Stations');

        // Add primary station to legend
        const primaryLegend = legendGroup.append('g')
            .attr('transform', 'translate(0, 95)');

        // Primary station area band
        primaryLegend.append('rect')
            .attr('x', 0)
            .attr('y', -8)
            .attr('width', 20)
            .attr('height', 16)
            .attr('fill', '#aeaeae')
            .attr('opacity', 0.5);

        // Primary station line
        primaryLegend.append('line')
            .attr('x1', 0)
            .attr('y1', 0)
            .attr('x2', 20)
            .attr('y2', 0)
            .attr('stroke', '#1f77b4')
            .attr('stroke-width', 2.5);

        // Primary station name
        primaryLegend.append('text')
            .attr('x', 25)
            .attr('y', 4)
            .attr('fill', '#222')
            .style('font-size', '12px')
            .text(formatStationName(
                data[0]?.stationName,
                data[0]?.city,
                data[0]?.state,
                data[0]?.country
            ));



        // Draw secondary data if available
        if (showSecondary && secondaryData && secondaryData.length > 0) {
            // Add secondary station to legend
            const secondaryLegend = legendGroup.append('g')
                .attr('transform', 'translate(0, 125)');

            // Secondary area band in legend
            secondaryLegend.append('rect')
                .attr('x', 0)
                .attr('y', -8)
                .attr('width', 20)
                .attr('height', 16)
                .attr('fill', '#d4a680')
                .attr('opacity', 0.2);

            // Secondary line in legend
            secondaryLegend.append('line')
                .attr('x1', 0)
                .attr('y1', 0)
                .attr('x2', 20)
                .attr('y2', 0)
                .attr('stroke', '#8b4513')
                .attr('stroke-width', 1);

            // Secondary station name
            secondaryLegend.append('text')
                .attr('x', 25)
                .attr('y', 4)
                .attr('fill', '#222')
                .style('font-size', '12px')
                .text(formatStationName(
                    secondaryData[0]?.stationName,
                    secondaryData[0]?.city,
                    secondaryData[0]?.state,
                    secondaryData[0]?.country
                ));

            // Process secondary data
            let secondarySeries: any[] = [];
            if (secondaryData[0].timestamp || secondaryData[0].usAqi) {
                const secondaryRawRows = secondaryData
                    .map((d: any) => ({
                        timestamp: d.timestamp ? new Date(d.timestamp) : null,
                        value: d.usAqi != null ? +d.usAqi : (d['US AQI'] != null ? +d['US AQI'] : (d.pm25 != null ? +d.pm25 : NaN))
                    }))
                    .filter((r: any) => r.timestamp && !isNaN(r.value));

                const groupedSecondary = d3.group(secondaryRawRows, (r: any) => alignTo15(r.timestamp).getTime());

                secondarySeries = Array.from(groupedSecondary, ([k, vals]) => {
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
                secondarySeries = secondaryData.map((d: any) => ({
                    date: new Date(parseDate(d)),
                    avg: d.avg ?? d.usAqi ?? d.pm25,
                    p10: d.p10 ?? d.avg ?? d.usAqi ?? d.pm25,
                    p90: d.p90 ?? d.avg ?? d.usAqi ?? d.pm25,
                    count: d.count ?? 0
                })).filter((d: any) => d.date && !isNaN(d.avg));
            }

            // Draw secondary area band
            const secondaryArea = d3.area<any>()
                .x(d => x(d.date))
                .y0(d => y(d.p10))
                .y1(d => y(d.p90))
                .curve(d3.curveMonotoneX);

            g.append('path')
                .datum(secondarySeries)
                .attr('class', 'band secondary')
                .attr('d', secondaryArea)
                .attr('fill', '#d4a680')  // light brown for area band
                .attr('opacity', 0.2)      // lighter opacity
                .attr('clip-path', `url(#${clipId})`);

            // Draw secondary avg line
            const secondaryLine = d3.line()
                .x((d: any) => x(d.date))
                .y((d: any) => y(d.avg))
                .curve(d3.curveMonotoneX as any);

            g.append('path')
                .datum(secondarySeries)
                .attr('class', 'line secondary')
                .attr('d', secondaryLine as any)
                .attr('fill', 'none')
                .attr('stroke', '#8b4513')  // darker brown for the line
                .attr('stroke-width', 1)
                .attr('clip-path', `url(#${clipId})`);
        }

        // Add brush for zooming
        brush = (d3.brushX() as any).extent([[0, 0], [innerWidth, innerHeight]]).on('end', brushed as any);
        brushG = g.append('g')
            .attr('class', 'x-brush')
            .style('cursor', 'crosshair')
            .call(brush as any);

        // Draw raw points if enabled
        if (showRaw && filteredRawRows && filteredRawRows.length > 0) {
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

            const pts = g.append('g').attr('class', 'raw-points').attr('clip-path', `url(#${clipId})`);
            pts.selectAll('circle')
                .data(filteredRawRows)
                .join('circle')
                .attr('cx', (d: any) => x(d.timestamp))
                .attr('cy', (d: any) => y(d.value))
                .attr('r', 3.5)
                // .attr('fill', '#222')
				.attr('fill', (d: any) => {
					if (selectedLevel !== null) {
						const level = aqiLevels.find(l => l.min === selectedLevel);
						const max = level?.max ?? Infinity;
						if (level && d.value >= level.min && d.value <= max) {
							return d3.rgb(level.color).darker(0.7).formatHex();
						} else {
							return '#444'; // gray for out-of-range
						}
					} else {
						return '#444'; // gray when no level is selected
					}
				})
                .attr('opacity', 0.6)
                .on('mouseover', function (event: any, d: any) {
                    tooltip
					.style('display', 'block')
					.html(`Timestamp: ${d.timestamp.toISOString().slice(0, 16).replace('T',' ')}<br>AQI: ${d.value}`);
                })
                .on('mousemove', function (event: any) {
                    const rect = container.getBoundingClientRect();
                    const left = event.clientX - rect.left + 12;
                    const top = event.clientY - rect.top + 12;
                    tooltip.style('left', left + 'px').style('top', top + 'px');
                })
                .on('mouseout', function () {
                    tooltip.style('display', 'none');
                });
        }
    }

    onMount(() => {
        draw();
        ro = new ResizeObserver(() => draw());
        if (container) ro.observe(container);
    });

    onDestroy(() => {
        if (ro && container) ro.disconnect();
    });

    // re-draw when inputs change
    $: if (container) {
        data, showRaw, selectedLevel, showSecondary, secondaryData;
        draw();
    }
</script>

<!-- AQI Level Legend -->
<div class="aqi-legend">
    <div class="aqi-levels">
        {#each aqiLevels as level}
            <button
                type="button"
                class="aqi-level {selectedLevel === level.min ? 'selected' : ''}"
                style="background-color: {level.color}; cursor: pointer;"
                aria-pressed={selectedLevel === level.min}
                on:click={() => selectedLevel = selectedLevel === level.min ? null : level.min}
            >
                <span class="level-name">{level.name}</span>
                <span class="level-range">
                    {level.min}-{level.max ?? '500+'}
                </span>
            </button>
        {/each}
    </div>
</div>

<div bind:this={container} class="aqi-timeseries" style="width:100%; position:relative"></div>

<style>
    /* :global(.band) { fill: #aeaeae; opacity: 0.5; } */
    /* :global(.line) { stroke: #1f77b4; fill: none; stroke-width: 1.5px; } */
    :global(svg) { font-family: sans-serif; }
    .aqi-legend {
        margin: 20px 0;
    }
    .aqi-levels {
        display: flex;
        flex-direction: row;
        gap: 4px;
        flex-wrap: wrap;
    }
    button.aqi-level {
        padding: 8px 12px;
        border-radius: 4px;
        color: black;
        min-width: 100px;
        text-align: center;
        display: flex;
        flex-direction: column;
        font-size: 12px;
        transition: outline 0.2s, box-shadow 0.2s;
    }
    button.aqi-level.selected {
        outline: 2px solid #222;
        box-shadow: 0 0 0 2px #fff, 0 0 0 4px #222;
    }
    .level-name {
        font-weight: bold;
        margin-bottom: 2px;
    }
    .level-range {
        font-size: 11px;
        opacity: 0.9;
    }
</style>