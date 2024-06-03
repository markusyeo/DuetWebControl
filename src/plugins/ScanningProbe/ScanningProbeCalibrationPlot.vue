<style scoped>
.calibration-plot {
  height: 100%;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  background-color: transparent;
}

.canvas-wrapper {
  height: 100%;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
}

canvas {
  height: 100%;
  width: 100%;
  position: relative;
  background: transparent;
}

.v-file-input {
  margin-top: 16px;
  width: 100%;
}
</style>

<template>
  <div class="calibration-plot">
    <v-alert v-if="containsInvalidValues" type="warning" text>
      JSON contains invalid values with probeValue set to 999999.
    </v-alert>

    <div class="canvas-wrapper">
      <canvas ref="scatterChart"></canvas>

      <v-file-input
        label="Upload Calibration JSON"
        @change="onFileChange"
        accept=".json"
        outlined
        full-width
      />

      <v-text-field
        v-if="jsonLoaded"
        v-model="calibrationTemp"
        label="Calibration Temperature (°C)"
        type="number"
        outlined
        full-width
      />
    </div>
  </div>
</template>

<script>
import { Chart } from "chart.js";

export default {
  data() {
    return {
      scatterChart: null,
      calibrationTemp: 25,
      probeData: {
        scanCoefficients: { probeValueDelta: null, A: null, B: null, C: null },
        probeThreshold: null,
      },
      chartData: {
        datasets: [
          {
            label: "Probe Temp vs. Computed Height",
            data: [],
            backgroundColor: "rgba(54, 162, 235, 0.5)",
            type: "line",
            fill: false,
          },
          {
            label: "Best Fit Curve",
            data: [],
            borderColor: "rgba(255, 99, 132, 1)",
            backgroundColor: "rgba(255, 99, 132, 0.5)",
            type: "line",
            fill: false,
          },
        ],
      },
      containsInvalidValues: false,
      jsonLoaded: false,
    };
  },
  watch: {
    calibrationTemp: "computeTemperatureCoefficients",
  },
  mounted() {
    this.initChart();
  },
  methods: {
    initChart() {
      const ctx = this.$refs.scatterChart.getContext("2d");
      this.scatterChart = new Chart(ctx, {
        type: "scatter",
        data: this.chartData,
        options: {
          responsive: true,
          scales: {
            x: {
              type: "linear",
              position: "bottom",
              title: {
                display: true,
                text: "Probe Temp (°C)",
              },
            },
            y: {
              title: {
                display: true,
                text: "Height",
              },
            },
          },
        },
      });
    },
    onFileChange(file) {
      if (file) {
        const reader = new FileReader();
        reader.onload = (event) => {
          const csvContent = event.target.result;
          this.resetScanCoefficients();
          this.parseJsonData(csvContent);
          this.jsonLoaded = true;
        };
      }
    },
    resetData() {
      this.resetProbeData();
      this.chartData.datasets[0].data = [];
      this.chartData.datasets[1].data = [];
      this.containsInvalidValues = false;
      this.jsonLoaded = false;
      this.calibrationTemp = 25;
    },
    resetProbeData() {
      const nullScanCoefficients = {
        probeValueDelta: null,
        A: null,
        B: null,
        C: null,
      };
      this.probeData = {
        scanCoefficients: nullScanCoefficients,
        probeThreshold: null,
      };
    },
    parseJsonData(jsonContent) {
      try {
        const jsonData = JSON.parse(jsonContent);
        const validData = [];

        this.resetData();
        this.containsInvalidValues = false;
        this.probeData.scanCoefficients = jsonData.coefficients;
        this.probeData.probeThreshold = jsonData.probeThreshold;

        jsonData.calibrationValues.forEach((dataPoint) => {
          const [bedTemp, probeTemp, probeValue] = dataPoint;

          if (probeValue === 999999) {
            this.containsInvalidValues = true;
          } else {
            const height = this.computeHeight(probeValue);
            validData.push({ x: probeTemp, y: height });
          }
        });

        this.chartData.datasets[0].data = validData;

        this.computeTemperatureCoefficients();

        this.updateScatterChart();
      } catch (error) {
        console.error("Error parsing JSON:", error);
      }
    },
    computeHeight(probeValue) {
      const probeDelta = probeValue - this.probeData.probeThreshold;
      const scanCoefficients = this.probeData.scanCoefficients;
      return (
        scanCoefficients.A * probeDelta +
        scanCoefficients.B * probeDelta ** 2 +
        scanCoefficients.C * probeDelta ** 3
      );
    },
    computeBestFitCurve(data, xDelta = 0) {
      const n = data.length;
      let sumX = 0;
      let sumY = 0;
      let sumXY = 0;
      let sumXX = 0;

      data.forEach((dataPoint) => {
        const x = dataPoint.x - xDelta;
        sumX += x;
        sumY += dataPoint.y;
        sumXY += x * dataPoint.y;
        sumXX += x * x;
      });

      const A = (n * sumXY - sumX * sumY) / (n * sumXX - sumX * sumX);
      const B = (sumY - A * sumX) / n;

      return [A, B];
    },
    computeTemperatureCoefficients() {
      const data = this.chartData.datasets[0].data;
      const calibrationTemp = this.calibrationTemp;

      const [A, B] = computeBestFitCurve(data, calibrationTemp);

      const bestFitData = data.map((dataPoint) => {
        const deltaTemp = dataPoint.x - calibrationTemp;
        const bestFitHeight = A * deltaTemp + B * deltaTemp ** 2;
        return { x: dataPoint.x, y: bestFitHeight };
      });

      this.chartData.datasets[1].data = bestFitData;

      this.updateScatterChart();
    },
    updateScatterChart() {
      if (this.scatterChart) {
        this.scatterChart.update();
      }
    },
  },
};
</script>
