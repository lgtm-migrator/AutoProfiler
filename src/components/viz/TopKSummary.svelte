<script lang="ts">
    import { format } from 'd3-format';
    import BarAndLabel from './BarAndLabel.svelte';
    import { config } from '../utils/sizes';

    import type { ValueCount } from '../../common/exchangeInterfaces';

    // export let displaySize: string = 'md';
    export let totalRows: number;
    export let topK: ValueCount[]; // FIXME
    export let color: string;
    export let containerWidth: number;

    $: smallestPercentage = Math.min(
        ...topK.slice(0, 5).map(entry => entry.count / totalRows)
    );
    $: formatPercentage =
        smallestPercentage < 0.01 ? format('0.2%') : format('0.1%');

    $: formatCount = format(',');

    // time to create a single way to get the width of an element.
</script>

<div class="w-full">
    <div
        class="grid w-full"
        style="
        grid-template-columns: auto  max-content; 
        grid-auto-rows: 19px;
        justify-items: stretch; 
        justify-content: stretch; 
        grid-column-gap: 1rem;"
    >
        {#each topK.slice(0, 10) as { value, count }}
            {@const printValue = value === null ? ' null ∅' : value}
            <div class="text-ellipsis overflow-hidden whitespace-nowrap">
                {printValue}
            </div>

            {@const negligiblePercentage = count / totalRows < 0.0002}
            {@const percentage = negligiblePercentage
                ? '<.01%'
                : formatPercentage(count / totalRows)}
            <div>
                <BarAndLabel value={count / totalRows} {color}>
                    <span
                        class:text-gray-500={negligiblePercentage &&
                            containerWidth >= config.hideRight}
                    >
                        {formatCount(count)}
                        {#if !containerWidth || containerWidth >= config.hideRight}
                            {#if percentage.length < 6}&nbsp;{/if}{#if percentage.length < 5}&nbsp;{/if}&nbsp;<span
                                class:text-gray-600={!negligiblePercentage}
                                >({percentage})</span
                            >
                        {/if}
                    </span>
                </BarAndLabel>
            </div>
        {/each}
    </div>
</div>
