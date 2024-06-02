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
    </div>
  </div>
</template>

<script>
import { Chart } from "chart.js";

export default {
  data() {
    return {
      scatterChart: null,
      chartData: {
        datasets: [
          {
            label: "Probe Temp vs. Probe Value",
            data: [],
            backgroundColor: "rgba(54, 162, 235, 0.5)",
          },
        ],
      },
      scanCoefficients: { A: null, B: null, C: null, triggerHeight: null },
      containsInvalidValues: false,
    };
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
                text: "Probe Temp",
              },
            },
            y: {
              title: {
                display: true,
                text: "Probe Value",
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
        };
      }
    },
    parseJsonData(jsonContent) {
      try {
        const jsonData = JSON.parse(jsonContent);
        const validData = [];

        this.containsInvalidValues = false;

        jsonData.calibrationValues.forEach((dataPoint) => {
          const [bedTemp, probeTemp, probeValue] = dataPoint;

          if (probeValue === 999999) {
            this.containsInvalidValues = true;
          } else {
            validData.push({ x: probeTemp, y: probeValue });
          }
        });

        this.chartData.datasets[0].data = validData;

        this.setScanCoefficients(
          jsonData.scanCoefficients.A,
          jsonData.scanCoefficients.B,
          jsonData.scanCoefficients.C,
          jsonData.scanCoefficients.triggerHeight
        );

        if (this.scatterChart) {
          this.scatterChart.update();
        }
      } catch (error) {
        console.error("Error parsing JSON:", error);
      }
    },
    setScanCoefficients(A, B, C, triggerHeight) {
      this.scanCoefficients = { A, B, C, triggerHeight };
    },
    resetScanCoefficients() {
      this.scanCoefficients = {
        A: null,
        B: null,
        C: null,
        triggerHeight: null,
      };
    },
  },
};
</script>
