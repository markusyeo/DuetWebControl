<style scoped>
.card {
  display: flex;
  flex-direction: column;
  width: 100%;
}

.content {
  position: relative;
}

.content > canvas {
  position: absolute;
}
</style>

<template>
  <v-col>
    <v-switch v-model="filterValues" label="Filter out 999999"></v-switch>
    <v-card class="d-flex flex-column flex-grow-1">
      <canvas ref="chart"></canvas>
    </v-card>
  </v-col>
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

const sampleInterval = 100; // Interval at which probe samples are recorded
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

  if (!dataset || dataset.locale !== i18n.locale) {
    if (dataset) {
      dataset.label = index.toString();
      dataset.locale = i18n.locale;
    } else {
      dataset = makeDataset(index, probeSampleData.times.length);
      machineData.probeValues.push(dataset);
    }
  }

  const probeValue = sensor.value !== null ? sensor.value[0] : NaN;
  dataset.data!.push(probeValue);

  const heightParams = getHeightParams(sensor);
  const heightValue = calculateHeight(probeValue, heightParams);
  if (heightValue !== null) {
    pushHeightData(index, heightValue);
  }
}

function pushHeightData(index: number, heightValue: number) {
  let machineData = probeSampleData;
  let heightDataset = machineData.probeValues.find(
    (item) => item.index === index && item.yAxisID === "y1"
  );

  if (!heightDataset) {
    heightDataset = {
      ...makeDataset(index, machineData.times.length),
      yAxisID: "y1",
      label: `Height for Probe ${index}`,
      borderColor: "rgba(255, 162, 235, 0.6)",
      backgroundColor: "rgba(255, 162, 235, 0.1)",
    };
    machineData.probeValues.push(heightDataset);
  }

  heightDataset.data!.push(heightValue);
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

let storeSubscribed = false,
  instances: Array<{ update: () => void }> = [];

export default Vue.extend({
  computed: {
    darkTheme(): boolean {
      return store.state.settings.darkTheme;
    },
    selectedMachine(): string {
      return store.state.selectedMachine;
    },
    chartOptions() {
      return {
        scales: {
          x: {
            type: "time",
            time: { parser: "HH:mm:ss", tooltipFormat: "ll HH:mm:ss" },
            title: { display: true, text: "Time" },
          },
          y: {
            beginAtZero: true,
            title: { display: true, text: "Probe Value" },
          },
          y1: {
            beginAtZero: true,
            position: "right",
            title: { display: true, text: "Height (mm)" },
          },
        },
        plugins: { legend: { position: "top" } },
        responsive: true,
        maintainAspectRatio: false,
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
      heightParams: {} as HeightCalculationParams,
      filterValues: false,
    };
  },
  methods: {
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
      this.updateChartData();
    },
    updateHeightParams() {
      const machine = store.state.machine.model;
      if (machine) {
        const probe = machine.sensors.probes[0];
        if (probe) {
          this.heightParams = getHeightParams(probe);
        }
      }
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
    filterProbeValues(): ProbeChartDataset[] {
      return probeSampleData.probeValues.map((dataset) => {
        const filteredData = dataset.data!.map((value) =>
          value == defaultMaxProbevalue ? NaN : value
        );
        return {
          ...dataset,
          data: filteredData,
        } as ProbeChartDataset;
      });
    },
    updateChartData() {
      const filteredProbeValues = this.filterValues
        ? this.filterProbeValues()
        : probeSampleData.probeValues;
      this.chart.data.datasets = filteredProbeValues;
      this.chart.update();
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
  mounted() {
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
    this.applyDarkTheme(this.darkTheme);

    // Keep track of updates
    instances.push(this);
    if (!storeSubscribed) {
      this.$root.$on(Events.machineAdded, () => {
        probeSampleData.times = [];
        probeSampleData.probeValues = [];
      });

      this.$root.$on(Events.machineModelUpdated, (hostname: string) => {
        const dataset = probeSampleData,
          now = new Date().getTime();
        if (
          dataset.times.length === 0 ||
          now - dataset.times[dataset.times.length - 1] > sampleInterval
        ) {
          // Record sensor temperatures
          store.state.machine.model.sensors.probes.forEach(
            (probe, probeIndex) => {
              if (probe !== null) {
                if (probe.type.valueOf() == 11 /* Scanning Probe */) {
                  pushSeriesData(probeIndex, probe);
                }
              }
            }
          );

          // Record time and deal wih expired temperature samples
          while (
            dataset.times.length &&
            now - dataset.times[0] > maxSampleTime
          ) {
            dataset.times.shift();
            dataset.probeValues.forEach((data) => data.data!.shift());
          }
          dataset.times.push(now);
        }

        instances.forEach((instance) => instance.update());
      });

      storeSubscribed = true;
    }
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
