<style scoped>
.card-chart {
  max-height: 70vh;
  overflow: hidden;
}
.content {
  position: relative;
}

.content > canvas {
  position: absolute;
}
</style>

<template>
  <v-container fluid>
    <v-row>
      <v-col>
        <v-alert v-if="missingProbeCoefficients" type="warning" text>
          Scanning probe coefficients are not available.
        </v-alert>
      </v-col>
    </v-row>
    <v-col>
      <div class="text--primary">
        Select the scanning probes to display on the chart.
      </div>
    </v-col>

    <v-row class="mb-3">
      <v-col cols="9">
        <v-select
          v-model="selectedProbes"
          :items="scanningProbes"
          multiple
          :item-text="(item) => `Probe ${item.id}`"
          :item-value="(item) => item.id"
          label="Scanning Probes"
          hide-details
          solo
        />
      </v-col>

      <v-col cols="3" class="d-flex justify-end">
        <v-switch v-model="filterValues" label="Filter out 999999" />
      </v-col>
    </v-row>

    <v-row>
      <v-col>
        <v-card class="d-flex flex-column flex-grow-1">
          <canvas ref="chart"></canvas>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts">
import Chart, {
  ChartDataSets,
  MajorTickOptions,
  NestedTickOptions,
} from "chart.js";
import dateFnsLocale from "date-fns/locale/en-US";
import { Probe } from "@duet3d/objectmodel";
import Vue from "vue";

import i18n from "@/i18n";
import store from "@/store";
import Events from "@/utils/events";
import { int } from "@babylonjs/core";

const probeColors = [
  "primary",
  "red",
  "green",
  "orange",
  "grey",
  "lime",
  "black",
  "purple",
  "yellow",
  "teal",
  "brown",
  "deep-orange",
  "pink",
  "blue-grey",
];

const sampleInterval = 1000; // Interval at which probe samples are recorded
const defaultMaxProbevalue = 999999; // Default maximum probe value
const defaultMaxHeightValue = 10; // Default maximum height value
const maxSampleTime = 120000; // Maximum time to save sample data in ms

interface ExtraDatasetValues {
  index: number;
  locale: string;
}

type ProbeChartDataset = ChartDataSets & ExtraDatasetValues;

interface ProbeSampleData {
  times: Array<number>;
  probeValues: ProbeChartDataset[];
}

const probeSampleData: ProbeSampleData = {
  times: [],
  probeValues: [],
};

/**
 * Make a new dataset to render temperature data
 * @param index Sensor index
 * @param numSamples Number of current samples to generate for the resulting datset
 */
function makeDataset(index: int, numSamples: int): ProbeChartDataset {
  const color = probeColors[index],
    dataset = {
      index,
      fill: false,
      backgroundColor: color,
      borderColor: color,
      borderWidth: 2,
      label: `Probe ${index}`,
      data: new Array<number>(numSamples).fill(NaN),
      locale: i18n.locale,
      pointRadius: 0,
      pointHitRadius: 0,
      showLine: true,
    };
  return dataset;
}

/**
 * Push sensor data of a given machine to the dataset
 * @param index Index of the sensor
 * @param sensor Sensor item
 */
function pushSeriesData(index: number, sensor: Probe) {
  let machineData = probeSampleData;
  let dataset = machineData.probeValues.find((item) => item.index === index);

  if (!dataset) {
    const newDataset = makeDataset(index, probeSampleData.times.length);
    probeSampleData.probeValues.push(newDataset);
    newDataset.label = index.toString();
    newDataset.locale = i18n.locale;
  }

  const probeValue = sensor.value ? sensor.value[0] : NaN;
  const heightParams = getHeightParams(sensor);
  const heightValue = calculateHeight(probeValue, heightParams);

  if (dataset && dataset.data) {
    dataset.data.push(probeValue);
  }

  if (heightValue !== null) {
    const heightDataset = probeSampleData.probeValues.find(
      (item) => item.index === index && item.yAxisID === "y1"
    );
    if (!heightDataset) {
      const newHeightDataset = {
        ...makeDataset(index, probeSampleData.times.length),
        yAxisID: "y1",
        label: `Height for Probe ${index}`,
        borderColor: "rgba(255, 162, 235, 0.6)",
        backgroundColor: "rgba(255, 162, 235, 0.1)",
      };
      probeSampleData.probeValues.push(newHeightDataset);
    }

    if (heightDataset && heightDataset.data) {
      heightDataset.data.push(heightValue);
    }
  }
}

interface HeightCalculationParams {
  trigger_height: number;
  A: number | null;
  B: number | null;
  C: number | null;
  probe_threshold: number | null;
  temp_coefficients: number[] | null;
}

function calculateHeight(
  probe_reading: number,
  params: HeightCalculationParams
): number | null {
  const { trigger_height, A, B, C, probe_threshold } = params;

  if (probe_threshold === null) {
    // Threshold is null, do not plot
    return null;
  }

  // Use 0 as the default value for A, B, and C if they are null
  const a = A ?? 0;
  const b = B ?? 0;
  const c = C ?? 0;

  if (a == 0 && b == 0 && c == 0) {
    return null;
  }

  const delta = probe_reading - probe_threshold;
  const height =
    trigger_height +
    a * delta +
    b * Math.pow(delta, 2) +
    c * Math.pow(delta, 3);
  return isNaN(height) ? NaN : height;
}

function getHeightParams(sensor: Probe): HeightCalculationParams {
  const heightParams: HeightCalculationParams = {
    trigger_height: sensor.triggerHeight,
    A: sensor.scanCoefficients ? sensor.scanCoefficients[0] : 0,
    B: sensor.scanCoefficients ? sensor.scanCoefficients[1] : 0,
    C: sensor.scanCoefficients ? sensor.scanCoefficients[2] : 0,
    probe_threshold: sensor.threshold,
    temp_coefficients: sensor.temperatureCoefficients,
  };
  return heightParams;
}

interface ProbeWithId extends Probe {
  id: number;
}

let storeSubscribed = false,
  instances: Array<{ update: () => void }> = [];

export default Vue.extend({
  computed: {
    scanningProbes(): ProbeWithId[] {
      const machine = store.state.machine.model;
      if (!machine) return [];

      return machine.sensors.probes
        .map(
          (probe, index) =>
            ({
              ...probe,
              id: index,
            } as ProbeWithId)
        )
        .filter(
          (probe): probe is ProbeWithId => probe !== null && probe.type === 11
        ); // Scanning Probes
    },
    probeIds(): number[] {
      return this.scanningProbes.map((probe) => probe.id);
    },
    darkTheme(): boolean {
      return store.state.settings.darkTheme;
    },
    selectedMachine(): string {
      return store.state.selectedMachine;
    },
    chartOptions() {
      return {
        responsive: true,
        maintainAspectRatio: false,
        scales: {
          x: {
            type: "time",
            time: { parser: "HH:mm:ss", tooltipFormat: "ll HH:mm:ss" },
            title: { display: true, text: "Time" },
          },
          y: {
            beginAtZero: true,
            title: { display: true, text: "Probe Value" },
            ticks: {
              stepSize: 50,
            },
          },
          y1: {
            beginAtZero: true,
            position: "right",
            title: { display: true, text: "Height (mm)" },
            ticks: {
              stepSize: 0.5,
            },
          },
        },
      };
    },
  },
  data() {
    return {
      chart: {} as Chart,
      lastUpdate: 0,
      maxProbeValue: defaultMaxProbevalue,
      minProbeValue: 0,
      minHeightValue: 0,
      maxHeightValue: 10,
      selectedProbes: [] as number[],
      heightParams: {} as HeightCalculationParams,
      filterValues: false,
      missingProbeCoefficients: false,
    };
  },
  methods: {
    initializeDefaultSelectedProbe() {
      if (this.scanningProbes.length > 0) {
        this.selectedProbes = [this.scanningProbes[0].id];
      }
    },
    initChart() {
      this.chart = new Chart(this.$refs.chart as HTMLCanvasElement, {
        type: "line",
        options: {
          // Disable animation
          animation: {
            duration: 0,
          },
          // Disable line smoothing
          elements: {
            line: {
              tension: 0,
            },
          },
          legend: {
            labels: {
              filter: (legendItem, data) =>
                data.datasets![legendItem.datasetIndex!].showLine,
              fontFamily: "Roboto,sans-serif",
            },
          },
          maintainAspectRatio: false,
          responsive: true,
          responsiveAnimationDuration: 0,
          scales: {
            xAxes: [
              {
                adapters: {
                  date: {
                    locale: dateFnsLocale,
                  },
                },
                gridLines: {
                  display: true,
                },
                ticks: {
                  min: new Date().getTime() - maxSampleTime,
                  max: new Date().getTime(),
                  minor: {
                    fontFamily: "Roboto,sans-serif",
                  },
                  major: {
                    fontFamily: "Roboto,sans-serif",
                  },
                },
                time: {
                  unit: "minute",
                  displayFormats: {
                    minute: "HH:mm",
                  },
                },
                type: "time",
              },
            ],
            yAxes: [
              {
                id: "y0",
                position: "left",
                gridLines: {
                  display: true,
                },
                ticks: {
                  minor: {
                    fontFamily: "Roboto,sans-serif",
                  },
                  major: {
                    fontFamily: "Roboto,sans-serif",
                  },
                  min: this.minProbeValue,
                  max: this.maxProbeValue,
                  stepSize: 5,
                },
              },
              {
                id: "y1",
                position: "right",
                scaleLabel: {
                  display: true,
                  labelString: "Height (mm)",
                },
                ticks: {
                  minor: {
                    fontFamily: "Roboto,sans-serif",
                  },
                  major: {
                    fontFamily: "Roboto,sans-serif",
                  },
                  min: this.minHeightValue,
                  max: this.maxHeightValue,
                  stepSize: 0.1,
                },
              },
            ],
          },
        },
        data: {
          labels: probeSampleData.times,
          datasets: probeSampleData.probeValues,
        },
      });
    },
    checkMissingProbeCoefficients() {
      this.scanningProbes.forEach((probe) => {
        if (
          probe.scanCoefficients === null ||
          // All scan coefficients are 0
          probe.scanCoefficients.every((coefficient) => coefficient === 0) ||
          probe.threshold === null
        ) {
          this.missingProbeCoefficients = true;
        }
      });
    },
    calculateMinMaxValues() {
      let probeMin = Infinity;
      let probeMax = -Infinity;
      let heightMin = Infinity;
      let heightMax = -Infinity;
      const datasets = this.chart.data.datasets;
      if (!datasets) return; // Check if datasets is defined
      const dataset = datasets[0];
      const data = dataset.data as number[]; // Assert that data is an array of numbers
      data.forEach((value) => {
        if (typeof value === "number") {
          if (value < probeMin) probeMin = value;
          if (value > probeMax) probeMax = value;
        }
        // Assuming calculateHeight can return a height for each probe value
        const heightValue = calculateHeight(value, this.heightParams);
        console.log("calculated height value", heightValue);
        if (heightValue !== null && !isNaN(heightValue)) {
          if (heightValue < heightMin) heightMin = heightValue;
          if (heightValue > heightMax) heightMax = heightValue;
        }
      });

      this.minProbeValue = isFinite(probeMin) ? probeMin : 0;
      this.maxProbeValue = isFinite(probeMax) ? probeMax : defaultMaxProbevalue;

      // Update the height min/max values
      this.minHeightValue = isFinite(heightMin) ? heightMin : 0;
      this.maxHeightValue = isFinite(heightMax)
        ? heightMax
        : defaultMaxHeightValue;
    },
    update() {
      this.updateChartTick();
      this.chart.update();
    },
    updateChartTick() {
      this.calculateMinMaxValues();
      const now = new Date().getTime();
      if (now - this.lastUpdate >= 1000) {
        this.chart.config.options!.scales!.yAxes![0].ticks!.min = this
          .minProbeValue
          ? this.minProbeValue
          : 0;
        this.chart.config.options!.scales!.yAxes![0].ticks!.max = this
          .maxProbeValue
          ? this.maxProbeValue
          : defaultMaxProbevalue;
        this.chart.config.options!.scales!.yAxes![1].ticks!.min = this
          .minHeightValue
          ? this.minHeightValue
          : 0;
        this.chart.config.options!.scales!.yAxes![1].ticks!.max = this
          .maxHeightValue
          ? this.maxHeightValue
          : defaultMaxHeightValue;
        this.chart.config.options!.scales!.xAxes![0].ticks!.min =
          new Date().getTime() - maxSampleTime;
        this.chart.config.options!.scales!.xAxes![0].ticks!.max =
          new Date().getTime();

        this.chart.update();
        this.lastUpdate = now;
      }
    },
    applyDarkTheme(active: boolean) {
      const ticksColor = active ? "#FFF" : "#666";
      this.chart.config.options!.legend!.labels!.fontColor = ticksColor;
      (
        this.chart.config.options!.scales!.xAxes![0].ticks!
          .minor as NestedTickOptions
      ).fontColor = ticksColor;
      (
        this.chart.config.options!.scales!.xAxes![0].ticks!
          .major as MajorTickOptions
      ).fontColor = ticksColor;
      (
        this.chart.config.options!.scales!.yAxes![0].ticks!
          .minor as NestedTickOptions
      ).fontColor = ticksColor;
      (
        this.chart.config.options!.scales!.yAxes![0].ticks!
          .major as MajorTickOptions
      ).fontColor = ticksColor;

      const gridLineColor = active
        ? "rgba(255,255,255,0.15)"
        : "rgba(0,0,0,0.15)";
      this.chart.config.options!.scales!.xAxes![0].gridLines!.color =
        gridLineColor;
      this.chart.config.options!.scales!.yAxes![0].gridLines!.color =
        gridLineColor;
      this.chart.config.options!.scales!.yAxes![0].gridLines!.zeroLineColor =
        gridLineColor;

      this.chart.update();
    },
  },
  created() {
    this.initializeDefaultSelectedProbe();
  },
  mounted() {
    this.initChart();
    this.checkMissingProbeCoefficients();
    this.initializeDefaultSelectedProbe();

    instances.push(this);
    this.$root.$on(Events.machineAdded, () => {
      probeSampleData.times = [];
      probeSampleData.probeValues = [];
    });

    this.$root.$on(Events.machineModelUpdated, (hostname: string) => {
      const now = new Date().getTime();

      if (
        probeSampleData.times.length === 0 ||
        now - probeSampleData.times[probeSampleData.times.length - 1] >
          sampleInterval
      ) {
        probeSampleData.times.push(now);

        store.state.machine.model.sensors.probes.forEach(
          (probe, probeIndex) => {
            if (probe && probe.type === 11 /* Scanning Probe */) {
              pushSeriesData(probeIndex, probe);
            }
          }
        );

        // Remove expired temperature samples
        while (
          probeSampleData.times.length &&
          now - probeSampleData.times[0] > maxSampleTime
        ) {
          probeSampleData.times.shift();
          probeSampleData.probeValues.forEach((data) => data.data!.shift());
        }
        probeSampleData.times.push(now);
      }
      instances.forEach((instance) => instance.update());
    });
    storeSubscribed = true;
  },
  beforeDestroy() {
    instances = instances.filter((instance) => instance !== this, this);
  },
  watch: {
    darkTheme(to: boolean) {
      this.applyDarkTheme(to);
    },
    filterValues(newVal) {
      this.filterValues = newVal;
    },
    selectedMachine() {
      this.chart.config.data = {
        labels: probeSampleData.times,
        datasets: probeSampleData.probeValues,
      };
      this.update();
    },
  },
});
</script>
