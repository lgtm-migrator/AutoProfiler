<script lang="ts">
    /**
     * TimestampDetail.svelte
     * ----------------------
     * This component is a diagnostic plot of the count(*) over time of a timestamp column.
     * The goal is to enable users to understand abnormalities and trends in the timestamp columns
     * of a dataset. As such, this component can:
     * - zoom into a specified scrub region – if the user ctrl + clicks + drags, the component
     * will zoom into a specific region, enabling the user to better understand weird data.
     * - panning – after zooming, the user may pan around to better situate the viewport.
     * - shift + clicking – users can copy the timestamp value.
     *
     * The graph will contain an unsmoothed series (showing noise * abnormalities) by default, and
     * a smoothed series (showing the trend) if the time series merits it.
     */
    import _ from 'lodash';
    import { onMount, setContext } from 'svelte';
    import { guidGenerator } from '../../utils/guid';
    import { spring } from 'svelte/motion';
    import { fly, fade } from 'svelte/transition';
    import { cubicOut as easing } from 'svelte/easing';
    import { scaleLinear } from 'd3-scale';
    import type { ScaleLinear } from 'd3-scale';
    import { DEFAULT_COORDINATES } from '../../utils/constants';
    import { createScrubAction } from '../../actions/scrub-action-factory';
    import { extent, bisector, max, min } from 'd3-array';
    import { outline } from '../../actions/outline';
    import type { Interval, TimeBin } from '../../../common/exchangeInterfaces';
    import { writable } from 'svelte/store';
    import type { Writable } from 'svelte/store';
    import { createExtremumResolutionStore } from './extremum-resolution-store';

    import TimestampBound from './TimestampBound.svelte';
    import TimestampProfileSummary from './TimestampProfileSummary.svelte';

    import TimestampMouseoverAnnotation from './TimestampMouseoverAnnotation.svelte';
    import TimestampPaths from './TimestampPaths.svelte';

    import type { PlotConfig } from '../../utils/constants';
    import ZoomWindow from './ZoomWindow.svelte';

    const id = guidGenerator();

    // data is assumed to be sorted
    export let data: TimeBin[];

    export let width = 360;
    export let height = 120;
    export let curve = 'curveLinear';
    export let smooth = true;

    export let separate = true;
    $: separateQuantity = separate ? 0.25 : 0;

    export let xAccessor: string;
    export let yAccessor: string;

    // rowsize for table
    export let left = 1;
    export let right = 1;
    export let top = 12;
    export let bottom = 4;
    export let buffer = 0;

    /** text elements */
    export let fontSize = 12;
    // the gap b/t text nodes
    export let textGap = 4;

    /** the datatype of this column */
    // export let type: string;

    /** zoom elements */
    export let zoomWindowColor = 'hsla(217, 90%, 60%, .2)';

    /** rollup grain, time range, etc. */
    export let interval: Interval;
    // export let rollupGrain: string;
    // export let estimatedSmallestTimeGrain: string;

    let devicePixelRatio = 1;
    onMount(() => {
        devicePixelRatio = window.devicePixelRatio;
    });

    /** These are our global scales, X and Y. */
    const X: Writable<ScaleLinear<number, number>> = writable(undefined);
    const Y: Writable<ScaleLinear<number, number>> = writable(undefined);
    /** make them available to the children. */
    setContext('rill:data-graphic:X', X);
    setContext('rill:data-graphic:Y', Y);

    const coordinates = writable(DEFAULT_COORDINATES);

    const plotConfig: Writable<PlotConfig> = writable({
        top,
        bottom,
        left,
        right,
        buffer,
        width,
        height,
        devicePixelRatio,
        plotTop: top + buffer,
        plotBottom: height - buffer - bottom,
        plotLeft: left + buffer,
        plotRight: width - right - buffer,
        fontSize: fontSize,
        textGap: textGap,
        id
    });

    setContext('rill:data-graphic:plot-config', plotConfig);

    $: $plotConfig.devicePixelRatio = devicePixelRatio;
    $: $plotConfig.width = width;
    $: $plotConfig.height = height;
    $: $plotConfig.top = top;
    $: $plotConfig.bottom = bottom;
    $: $plotConfig.left = left;
    $: $plotConfig.right = right;
    $: $plotConfig.buffer = buffer;
    $: $plotConfig.plotTop = top + buffer;
    $: $plotConfig.plotBottom = height - buffer - bottom;
    $: $plotConfig.plotLeft = left + buffer;
    $: $plotConfig.plotRight = width - right - buffer;
    $: $plotConfig.fontSize = fontSize;
    $: $plotConfig.textGap = textGap;

    /**
     * The scrub action creates a scrubbing event that enables the user to
     */
    const {
        coordinates: zoomCoords,
        scrubAction,
        isScrubbing: isZooming,
        updatePlotBounds: updatePlotBoundsForScrubber
    } = createScrubAction({
        plotLeft: $plotConfig.plotLeft,
        plotRight: $plotConfig.plotRight,
        plotTop: $plotConfig.plotTop,
        plotBottom: $plotConfig.plotBottom,
        startPredicate: (event: MouseEvent) => event.ctrlKey,
        movePredicate: (event: MouseEvent) => event.ctrlKey,
        endEventName: 'scrub'
    });

    /**
     * This scroll action creates a scrolling event that will be used in the svg container.
     * The main requirement is this event does not have the shiftKey in use.
     */
    const {
        scrubAction: scrollAction,
        isScrubbing: isScrolling,
        updatePlotBounds: updatePlotBoundsForScrolling
    } = createScrubAction({
        plotLeft: $plotConfig.plotLeft,
        plotRight: $plotConfig.plotRight,
        plotTop: $plotConfig.plotTop,
        plotBottom: $plotConfig.plotBottom,
        startPredicate: (event: MouseEvent) =>
            !event.ctrlKey && !event.shiftKey,
        movePredicate: (event: MouseEvent) => !event.ctrlKey && !event.shiftKey,
        moveEventName: 'scrolling'
    });

    /** update these plot bounds for scrolling and scrubbing, assuming they change. */
    $: updatePlotBoundsForScrubber({
        plotLeft: $plotConfig.plotLeft,
        plotRight: $plotConfig.plotRight,
        plotTop: $plotConfig.plotTop,
        plotBottom: $plotConfig.plotBottom
    });

    $: updatePlotBoundsForScrolling({
        plotLeft: $plotConfig.plotLeft,
        plotRight: $plotConfig.plotRight,
        plotTop: $plotConfig.plotTop,
        plotBottom: $plotConfig.plotBottom
    });

    let isZoomed = false;

    let zoomedXStart: Date;
    let zoomedXEnd: Date;
    // establish basis values
    let xExtents = [data[0]?.ts_start, data[data.length - 1]?.ts_end];
    // $: xExtents = extent(data, d => d[xAccessor]);
    $: xExtents = [data[0]?.ts_start, data[data.length - 1]?.ts_end];

    const xMin = createExtremumResolutionStore(xExtents[0], {
        duration: 300,
        easing,
        direction: 'min',
        alwaysOverrideInitialValue: true
    });
    const xMax = createExtremumResolutionStore(xExtents[1], {
        duration: 300,
        easing,
        direction: 'max',
        alwaysOverrideInitialValue: true
    });

    $: xMin.setWithKey('x', zoomedXStart || xExtents[0]);
    $: xMax.setWithKey('x', zoomedXEnd || xExtents[1]);

    // this adaptive smoothing should be a function?

    // Let's set the X Scale based on the $xMin and $xMax, or if the
    $: X.set(
        scaleLinear()
            .domain([$xMin, $xMax])
            .range([$plotConfig.plotLeft, $plotConfig.plotRight])
    );

    // Generate the line density by dividing the total available pixels by the window length.
    // We will scale by window.pixelDensityRatio.

    // Generate our Y Scale.
    let yExtents = extent(data, d => d[yAccessor]);
    $: yExtents = extent(data, d => d[yAccessor]);
    const yMax = createExtremumResolutionStore(Math.max(5, yExtents[1]));

    // Set Y if there's a new yMax or the range changes.
    $: Y.set(
        scaleLinear()
            .domain([0, $yMax])
            .range([$plotConfig.plotBottom, $plotConfig.plotTop])
    );

    // get the nearest point to where the cursor is.

    const bisectDate = bisector(d => d[xAccessor]).center;
    $: nearestPoint = data[bisectDate(data, $X.invert($coordinates.x))];

    function clearMouseMove() {
        coordinates.set(DEFAULT_COORDINATES);
    }

    function handleMouseMove(event: MouseEvent) {
        if (
            event.offsetX > $plotConfig.plotLeft &&
            event.offsetX < $plotConfig.plotRight
        ) {
            coordinates.set({ x: event.offsetX, y: event.offsetY });
        }
    }

    function setCursor(isZooming: boolean, isScrolling: boolean) {
        if (isZooming) return 'text';
        if (isScrolling) return 'grab';
        return 'inherit';
    }

    // when zooming / panning, get the total number of zoomed rows.
    let zoomedRows;

    // find the total number of rows currently visible in the zoom.
    $: if ($zoomCoords.start.x && $zoomCoords.stop.x) {
        const xStart = $X.invert(
            Math.min($zoomCoords.start.x, $zoomCoords.stop.x)
        );
        const xEnd = $X.invert(
            Math.max($zoomCoords.start.x, $zoomCoords.stop.x)
        );
        zoomedRows = ~~data
            .filter(di => {
                return di[xAccessor] >= xStart && di[xAccessor] <= xEnd;
            })
            .reduce((sum, di) => (sum += di[yAccessor]), 0);
    } else if (zoomedXStart && zoomedXEnd) {
        zoomedRows = ~~data
            .filter(di => {
                return (
                    di[xAccessor] >= zoomedXStart && di[xAccessor] <= zoomedXEnd
                );
            })
            .reduce((sum, di) => (sum += di[yAccessor]), 0);
    }

    // Tooltip & timestamp range variables.
    const tooltipSparkWidth = 84;
    const tooltipSparkHeight = 12;
    const tooltipPanShakeAmount = spring(0, { stiffness: 0.1, damping: 0.9 });
    let movementTimeout: ReturnType<typeof setTimeout>;

    $: zoomMinBound =
        ($zoomCoords.start.x
            ? $X.invert(Math.min($zoomCoords.start.x, $zoomCoords.stop.x))
            : min([zoomedXStart, zoomedXEnd])) || xExtents[0];

    $: zoomMaxBound =
        ($zoomCoords.start.x
            ? $X.invert(Math.max($zoomCoords.start.x, $zoomCoords.stop.x))
            : max([zoomedXStart, zoomedXEnd])) || xExtents[1];
</script>

<div style:width="100%">
    <TimestampProfileSummary
        {interval}
        numZoomedRows={zoomedRows}
        numTotalRows={~~data.reduce((a, b) => a + b[yAccessor], 0)}
        zoomed={!_.isUndefined($zoomCoords.start.x) ||
            !_.isUndefined(zoomedXStart)}
    />
    <svg
        width="100%"
        {height}
        style:cursor={setCursor($isZooming, $isScrolling)}
        use:scrubAction
        use:scrollAction
        on:scrolling={event => {
            if (isZoomed) {
                // clear the tooltip shake effect zeroing timeout.
                clearTimeout(movementTimeout);
                // shake the word "pan" in the tooltip here.
                tooltipPanShakeAmount.set(event.detail.movementX / 8);
                // set this timeout to resolve back to 0 if the user stops dragging.
                movementTimeout = setTimeout(() => {
                    tooltipPanShakeAmount.set(0);
                }, 150);

                const timeDistance =
                    $X.invert(event.detail.clientX + event.detail.movementX) -
                    $X.invert(event.detail.clientX);
                const oldXStart = new Date(+zoomedXStart);
                const oldXEnd = new Date(+zoomedXEnd);
                zoomedXStart = new Date(+zoomedXStart - +timeDistance);
                zoomedXEnd = new Date(+zoomedXEnd - +timeDistance);

                if (zoomedXStart < xExtents[0] || zoomedXEnd >= xExtents[1]) {
                    zoomedXStart = oldXStart;
                    zoomedXEnd = oldXEnd;
                }
            }
        }}
        on:scrub={event => {
            // set max and min here.
            zoomedXStart = new Date(
                $X.invert(Math.min(event.detail.start.x, event.detail.stop.x))
            );
            zoomedXEnd = new Date(
                $X.invert(Math.max(event.detail.start.x, event.detail.stop.x))
            );
            // mark that this graphic has been scrubbed.
            setTimeout(() => {
                isZoomed = true;
            }, 100);
        }}
        on:mousemove={handleMouseMove}
        on:mouseleave={clearMouseMove}
        on:contextmenu|preventDefault|stopPropagation
    >
        <defs>
            <linearGradient id="left-side">
                <stop offset="0%" stop-color="white" />
                <stop offset="100%" stop-color="rgba(255,255,255,0)" />
            </linearGradient>
            <linearGradient id="right-side">
                <stop offset="0%" stop-color="rgba(255,255,255,0)" />
                <stop offset="100%" stop-color="white" />
            </linearGradient>
        </defs>
        <clipPath id="data-graphic-{$plotConfig.id}">
            <rect
                x={$plotConfig.plotLeft}
                y={$plotConfig.plotTop}
                width={$plotConfig.plotRight - $plotConfig.plotLeft}
                height={$plotConfig.plotBottom - $plotConfig.plotTop}
            />
        </clipPath>
        <g clip-path="url(#data-graphic-{id})">
            <!-- core geoms -->
            <TimestampPaths {curve} {data} {xAccessor} {yAccessor} {smooth} />

            {#if isZoomed}
                <!-- fadeout gradients on each side? -->
                <rect
                    transition:fade
                    x={$plotConfig.plotLeft}
                    y={$plotConfig.plotTop}
                    width={20}
                    height={$plotConfig.plotBottom - $plotConfig.plotTop}
                    fill="url(#left-side)"
                />
                <rect
                    transition:fade
                    x={$plotConfig.plotRight - 20}
                    y={$plotConfig.plotTop}
                    width={20}
                    height={$plotConfig.plotBottom - $plotConfig.plotTop}
                    fill="url(#right-side)"
                />
            {/if}
            <!-- add baseline -->
            <line
                x1={$X?.range()[0]}
                x2={$X?.range()[1]}
                y1={$Y && $Y(0)}
                y2={$Y && $Y(0)}
                stroke="rgb(100,100,100)"
            />
        </g>
        <g>
            {#if $zoomCoords.start.x && $zoomCoords.stop.x}
                <ZoomWindow
                    start={$zoomCoords.start.x}
                    stop={$zoomCoords.stop.x}
                    color={zoomWindowColor}
                />
            {/if}
        </g>
        <!-- mouseover annotation -->
        {#if $coordinates.x}
            <TimestampMouseoverAnnotation
                point={nearestPoint}
                {xAccessor}
                {yAccessor}
            />
        {/if}
        <!-- scrub-clearing click region -->
        {#if zoomedXStart && zoomedXEnd}
            <text
                font-size={fontSize}
                x={$plotConfig.plotRight}
                y={$plotConfig.fontSize * 2 + $plotConfig.textGap}
                text-anchor="end"
                style:font-style="italic"
                style:user-select="none"
                style:cursor="pointer"
                class="transition-color fill-gray-500 hover:fill-black"
                in:fly={{ duration: 200, x: 16, delay: 200 }}
                out:fly={{ duration: 200, x: 16 }}
                use:outline
                on:click={() => {
                    zoomedXStart = undefined;
                    zoomedXEnd = undefined;
                    isZoomed = false;
                }}
            >
                clear zoom
            </text>
        {/if}
    </svg>

    <!-- Bottom time horizon labels -->
    <div class="select-none grid grid-cols-2 space-between">
        <TimestampBound align="left" value={zoomMinBound} />
        <TimestampBound align="right" value={zoomMaxBound} />
    </div>
</div>
