<style scoped>
.content {
  position: relative;
  min-height: 480px;
}

.content canvas {
  position: absolute;
}

th {
  white-space: nowrap;
}
</style>

<template>
  <v-row>
    <v-col>
      <v-card>
        <v-tabs v-model="tab">
          <!-- Tabs -->
          <v-tab href="#calibration">
            <v-icon class="mr-1">mdi-information</v-icon>
            Calibration
          </v-tab>
          <v-tab href="#probechart">
            <v-icon class="mr-1">mdi-file</v-icon>
            Probe Chart
          </v-tab>

          <v-btn
            color="success"
            class="align-self-center ml-auto mr-2 hidden-sm-and-down"
            :disabled="uiFrozen"
            @click="showCalibrationDialog = true"
          >
            <v-icon class="mr-1">mdi-record</v-icon>
            Calibrate Scanning Probe
          </v-btn>
        </v-tabs>

        <v-tabs-items v-model="tab">
          <!-- Current Settings -->
          <v-tab-item value="calibration">
            <div class="d-flex flex-column">
              <v-alert
                :value="!isScanningProbePresent"
                type="info"
                class="mb-0"
              >
                No scanning probe is conencted, please attach a scanning probe
                for calibration.
              </v-alert>
              <div
                v-show="isScanningProbePresent"
                class="content flex-grow-1 pa-2"
              >
                Placeholder for
              </div>
            </div>
          </v-tab-item>
          <v-tab-item value="probechart">
            <div v-if="isScanningProbePresent" class="d-flex flex-column">
              <probevalues-chart />
            </div>
            <v-alert v-else :value="true" type="info">
              No scanning probe connected. Please attach a scanning probe.
            </v-alert>
          </v-tab-item>
        </v-tabs-items>
      </v-card>
    </v-col>
    <calibrate-scanning-probe-dialog
      :shown.sync="showCalibrationDialog"
      @finished="recordingFinished"
    />
  </v-row>
</template>

<script>
import ProbeValuesChart from "./ProbeValuesChart.vue";
import { mapGetters } from "vuex";
import CalibrateScanningProbeDialog from "./CalibrateScanningProbeDialog.vue";
import store from "@/store";

function checkScanningProbePresent() {
  return (
    store.state.machine.model.sensors.probes.filter(
      (probe) => probe.type === 11
    ).length > 0
  );
}

export default {
  name: "ScanningProbe",
  components: {
    "probevalues-chart": ProbeValuesChart,
    "calibrate-scanning-probe-dialog": CalibrateScanningProbeDialog,
  },
  data() {
    return {
      tab: null,
      showCalibrationDialog: false,

      files: [],
      filesLastModified: [],
      loadingFiles: false,
      filesError: null,
    };
  },
  computed: {
    ...mapGetters(["isConnected", "uiFrozen"]),
    isScanningProbePresent() {
      return true;
      // return checkScanningProbePresent();
    },
  },
  methods: {
    updateChartData() {},
  },
  mounted() {
    this.updateChartData();
  },
};
</script>
