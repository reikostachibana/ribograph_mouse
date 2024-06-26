<script setup lang="ts">
import { ref, onMounted, computed, watch } from 'vue'
import PlotlyPlot from './PlotlyPlot.vue'
import type Plotly from '../plotly'
import { generateRange, CODON_DICT, getCoverageData, DataArray2D, sliderLogic } from '../utils'
import { getOffsetComputed } from '../localStorageStore'

const { sliderPositionsRaw, sliderPositions, sliderPositionsSecondRaw, sliderPositionsSecond } = sliderLogic()

const props = defineProps<{
    gene: string, // the gene name
    ids: number[], // experiment ids
    geneSequence?: string[],
    useOffsets?: boolean
}>()

interface CoverageData {
    cdsRange: [number, number],
    coverage: {
        index: number[],
        columns: number[],
        data: number[][]
    },
    gene: string,
    experiment: string,
    min: number,
    totalReads: number,
    id: number,
    offset: number[],
    geneSequence?: string
}

const coverageData = ref<CoverageData[]>([])

function updateCoverageData() {
    if (props.gene) {
        coverageData.value = []
        props.ids.forEach(addExperiment)
    }
}

function calculateMean(values: number[]): number {
  const sum = values.reduce((acc, val) => acc + val, 0);
  return sum / values.length;
}

function calculateStandardDeviation(values: number[], mean: number): number {
  const squaredDiffSum = values.reduce((acc, val) => acc + Math.pow(val - mean, 2), 0);
  const variance = squaredDiffSum / values.length;
  return Math.sqrt(variance);
}

function calculateZScore(value: number, mean: number, standardDeviation: number): number {
  return (value - mean) / standardDeviation;
}


async function addExperiment(experiment: number) {
    getCoverageData(experiment, props.gene).then(async x => {
        if (x) {
            x.id = experiment
            x.offset = (await getOffsetComputed(experiment)).value
            coverageData.value.push(x)
        }
    })
}

updateCoverageData() // on init
watch(() => props.gene, () => {
    updateCoverageData()
})

// assume experiment ids can only be appended
watch(() => props.ids, (newIds, oldIds) => {
    if (newIds.length != 1 && newIds.length > oldIds.length) {
        addExperiment(newIds[newIds.length - 1])
    }
})

const cdsRange = computed(() => coverageData.value.length > 0 ? coverageData.value[0].cdsRange : null)


const min = computed(() => coverageData.value.length > 0 ? 
    Math.min(...coverageData.value.map(x => x.min)) : 15)
const max = computed(() => coverageData.value.length > 0 ? 
    Math.max(...coverageData.value.map(x => x.min + x.coverage.data.length - 1)) : 40)

const datasets = computed<Partial<Plotly.PlotData>[]>(() => {
  const geneData: Partial<Plotly.PlotData>[] = coverageData.value
    .filter(x => x.gene === props.gene)
    .map(x => {
      const coverage = (new DataArray2D(x.coverage.data, x.min))
        .sliceSum(...sliderPositions.value, props.useOffsets ? x.offset : null);

      const cdsCoverage = coverage.slice(cdsRange.value[0], cdsRange.value[1] + 1);

      // Calculate mean and standard deviation of coverage
      const mean = calculateMean(cdsCoverage);
      const stdDev = calculateStandardDeviation(cdsCoverage, mean);
      console.log(mean, stdDev)

      // Apply z-score algorithm to each coverage value
      const zScores = coverage.map(value => {
        const zScore = calculateZScore(value, mean, stdDev);
        return Math.max(zScore, 0);
      });

      return {
        x: generateRange(0, x.coverage.columns.length),
        y: zScores,
        type: 'bar',
        name: x.experiment,
        marker: {
          opacity: 0.5, // Adjust as needed
        },
        textposition: 'none',
      };
    });

  // Add gene sequence labels if available
  if (geneSequenceLabels.value) {
    geneData.unshift(geneSequenceLabels.value);
  }

  const geneDataSecond: Partial<Plotly.PlotData>[] = coverageData.value
    .filter(x => x.gene === props.gene)
    .map(x => {
      const coverage = (new DataArray2D(x.coverage.data, x.min))
        .sliceSum(...sliderPositionsSecond.value, props.useOffsets ? x.offset : null);

      const cdsCoverage = coverage.slice(cdsRange.value[0], cdsRange.value[1] + 1);

      // Calculate mean and standard deviation of coverage
      const mean = calculateMean(cdsCoverage);
      const stdDev = calculateStandardDeviation(cdsCoverage, mean);
      console.log(mean, stdDev)

      // Apply z-score algorithm to each coverage value
      const zScores = coverage.map(value => {
        const zScore = calculateZScore(value, mean, stdDev);
        return Math.max(zScore, 0);
      });

      return {
        x: generateRange(0, x.coverage.columns.length),
        y: zScores.map(z => -z),
        type: 'bar',
        name: x.experiment,
        marker: {
          opacity: 0.5, // Adjust as needed
        },
        textposition: 'none',
      };
    });

// Add gene sequence labels if available
  if (geneSequenceLabels.value) {
    geneDataSecond.unshift(geneSequenceLabels.value);
  }

  return geneData.concat(geneDataSecond);
});


const options = computed<Partial<Plotly.Layout>>(() => ({
    title: {
        text: props.gene,
        y: 0.95,
        yanchor: 'top'
    },
    showlegend: true,
    legend: { "orientation": "h" },
    margin: { t: 40, b: 50, l: 30, r: 0 },
    yaxis: {
        bargap: 0,
        fixedrange: true
    },
    // barmode: 'relative',
    barmode: 'overlay',
    uirevision: props.gene, // reset the view only when the gene name changes
    dragmode: 'pan',
    ...(cdsRange.value) && { // conditionally add if cdsRange is provided
        shapes: [
            {
                type: 'rect',
                xref: 'x',
                yref: 'paper',
                x0: cdsRange.value[0],
                y0: 0,
                x1: cdsRange.value[1],
                y1: 1,
                fillcolor: '#d3d3d3',
                opacity: 0.2,
                line: {
                    width: 0
                }
            },
        ]
    }
}))

const geneSequence = computed(() => {
    if (coverageData.value && coverageData.value.length > 0) {
        return coverageData.value[0].geneSequence
    }
    return null
})

const geneSequenceText = computed<string[]>(() => {

    if (coverageData.value.length == 0) {
        return []
    }

    if (!geneSequence.value || geneSequence.value.length === 0 || !cdsRange.value) {
        return []
    }

    const geneSequenceArray = Array.from(geneSequence.value)
    const geneSequenceText = Array.from(geneSequenceArray) // copy the array

    // generate codon labels
    for (let i = cdsRange.value[0]; i < cdsRange.value[1]; i += 3) {
        const codonName = CODON_DICT[geneSequenceArray.slice(i, i + 3).join("")]
        geneSequenceText[i] = `<b>${geneSequenceArray[i]}</b> - ${codonName}`
        geneSequenceText[i + 1] = `<b>${geneSequenceArray[i + 1]}</b> - ${codonName}`
        geneSequenceText[i + 2] = `<b>${geneSequenceArray[i + 2]}</b> - ${codonName}`
    }

    return geneSequenceText
})

// find the maximum value across all data
const maxValue = computed<number>(() => Math.max(...coverageData.value.map(x => x.coverage.data).flat(2)))

/**
 * Return an array of color strings in parallel with the geneSequence.
 * The colors are grouped in 3s, to represent each codon
 * @param length the length of the array 
 * @param cdsRange 
 */
function makeColorArray(length: number, cdsRange: [number, number]) {
    const UTR_COLOR = 'rgba(204,204,204,1)'
    const COLOR_1 = 'purple'
    const COLOR_2 = 'blue'

    const arr = Array(length).fill(UTR_COLOR) // the default
    let flipflop = true;

    for (let i = cdsRange[0]; i < cdsRange[1]; i += 3) {
        arr[i] = arr[i + 1] = arr[i + 2] = flipflop ? COLOR_1 : COLOR_2
        flipflop = !flipflop
    }

    return arr
}

const viewSize = ref(Infinity)
const GENE_VIEW_THRESHOLD = 45

const geneSequenceLabels = computed<Partial<Plotly.PlotData> | null>(() =>
    geneSequence.value &&
        geneSequence.value.length !== 0 &&
        cdsRange.value ? ({
            x: generateRange(0, geneSequence.value.length),
            // fill the label array with the max value * some negative constant 
            // so that they take up a constant proportion of the width
            y: Array(geneSequence.value.length).fill(-0.2 * maxValue.value),
            type: 'bar',
            name: 'Sequence',
            marker: {
                opacity: 0.5,
                color: makeColorArray(geneSequence.value.length, cdsRange.value)
            },
            text: Array.from(geneSequence.value),
            customdata: [geneSequenceText.value],
            textposition: viewSize.value < GENE_VIEW_THRESHOLD ? 'auto' : 'none',
            // insidetextanchor: 'start',
            visible: 'legendonly',
            hovertemplate: '<b>%{text}</b> %{x} <extra></extra>',
        }) : null);


function setTextViewState(eventdata: Plotly.PlotRelayoutEvent) {
    if (eventdata['xaxis.range[1]'] && eventdata['xaxis.range[0]']) {
        viewSize.value = eventdata['xaxis.range[1]'] - eventdata['xaxis.range[0]']
    }
}

</script>

<template>
  <div class="my-5">
    <div class="slider-container">
      <Slider v-model="sliderPositionsRaw" :min="min" :max="max" :lazy="false" />
    </div>
    <div class="slider-container">
      <Slider v-model="sliderPositionsSecondRaw" :min="min" :max="max" :lazy="false" />
    </div>
  </div>
  <PlotlyPlot :datasets="datasets" :options="options" @plotly_relayout="setTextViewState($event)" class="mb-4" />
</template>

<style>
.slider-container {
  margin-bottom: 50px; /* Adjust the margin as needed */
}
</style>