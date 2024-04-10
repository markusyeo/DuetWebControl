<style>
.centered-alert > div {
  align-items: center;
}
.centered-alert > div > div {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
</style>

<template>
  <v-dialog v-model="shownInternal" max-width="640px" no-click-animation>
    <v-card>
      <v-card-title>
        <span class="headline"> Calibrate Scanning Probe </span>
      </v-card-title>

      <v-card-text class="pb-0">
        <v-window v-model="currentPage">
          <!-- Start -->
          <v-window-item value="start">
            This wizard lets you to calibrate your Duet scanning probe by using
            the thermistor attached to the coil.<br />

            <ul class="mt-3 mb-4">
              <li>Scanning Probe: Index #{{ probes[0].index }}</li>
              <li>
                Current temp coefficients:
                {{ probes[0].temperatureCoefficients }}
              </li>
              <li>
                Current calibration temp: {{ probes[0].calibrationTemperature }}
              </li>
            </ul>

            <v-alert
              :value="probes.length === 0"
              dense
              text
              type="error"
              class="my-3"
            >
              No scanning probe found!
              <a
                href="https://docs.duet3d.com/en/Duet3D_hardware/Duet_3_family/Duet_3_Scanning_Z_Probe"
                target="_blank"
                class="float-right"
              >
                Help
              </a>
            </v-alert>

            <v-alert
              :value="hasMultipleScanningProbes"
              type="info"
              dense
              class="mt-3"
            >
              This machine appears to have multiple scanning probes. If you are
              operating a tool changer, you will need to calibrate each scanning
              probe separately
            </v-alert>

            <v-alert
              :value="probes.length > 0"
              dense
              text
              type="success"
              class="my-3"
            >
              Ready to calibrate scanning probe!
            </v-alert>

            <span v-show="probes.length > 0"> Press Next to continue. </span>
          </v-window-item>

          <!-- Configuration -->
          <v-window-item value="config">
            <div class="d-flex flex-column">
              Here you can define various parameters to use for calibrating the
              scanning probe.
              <v-divider class="mb-3" />
              <div>
                <div class="text-subtitle-1">
                  Select Scanning Probe Thermistor
                </div>
                <v-select
                  v-model="calibrationParams.selectedThermistor"
                  :items="thermistors"
                  item-text="name"
                  item-value="id"
                  label="Thermistor"
                  class="mt-3"
                  solo
                  hide-details
                  :rules="thermistorRules()"
                ></v-select>
                <v-simple-table class="mt-1">
                  <thead>
                    <tr>
                      <th class="pl-0">Heater</th>
                      <th style="width: 20%">Start (°C)</th>
                      <th style="width: 20%">Stop (°C)</th>
                      <th style="width: 20%">Step (°C)</th>
                      <th style="width: 10%" class="pr-0"></th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr
                      v-for="(heater, index) in calibrationParams.heaters"
                      :key="`heater-${index}`"
                    >
                      <td class="px-0">
                        <v-select
                          v-model="heater.id"
                          :items="availableHeatersFor(heater.id)"
                          item-text="name"
                          item-value="id"
                          label="Heater"
                          class="pt-0"
                          hide-details
                        ></v-select>
                      </td>
                      <td style="width: 20%">
                        <v-text-field
                          v-model="heater.start"
                          type="number"
                          :min="0"
                          :max="getHeaterMaxTemp(heater.id)"
                          class="pt-0"
                          hide-details
                          :rules="startTempRules(getHeaterMaxTemp(heater.id))"
                        ></v-text-field>
                      </td>
                      <td style="width: 20%">
                        <v-text-field
                          v-model="heater.stop"
                          type="number"
                          :min="heater.start"
                          :max="getHeaterMaxTemp(heater.id)"
                          class="pt-0"
                          hide-details
                          :rules="
                            stopTempRules(
                              heater.start,
                              getHeaterMaxTemp(heater.id)
                            )
                          "
                        ></v-text-field>
                      </td>
                      <td style="width: 20%">
                        <v-text-field
                          v-model="heater.step"
                          type="number"
                          class="pt-0"
                          hide-details
                        ></v-text-field>
                      </td>
                      <td style="width: 10%" class="pr-0">
                        <v-btn
                          color="warning"
                          :disabled="calibrationParams.heaters.length <= 1"
                          icon
                          @click="removeHeater(index)"
                        >
                          <v-icon>mdi-close</v-icon>
                        </v-btn>
                      </td>
                    </tr>
                  </tbody>
                </v-simple-table>
                <v-btn
                  color="blue darken-1"
                  class="mx-auto"
                  outlined
                  text
                  @click="addHeater"
                  :disabled="
                    heaters.length === calibrationParams.heaters.length
                  "
                >
                  <v-icon class="mr-1">mdi-plus</v-icon>
                  Add Heater
                </v-btn>
                <v-alert type="warning" dense class="mt-3">
                  You should enclose your printer if possible to help simulate
                  higher probe temperatures.
                </v-alert>
                <v-divider class="mb-3" />
                <!-- Fan calbiration params -->

                <v-simple-table class="mt-1">
                  <thead>
                    <tr>
                      <th class="pl-0">Fan</th>
                      <th style="width: 20%">Speed</th>
                      <th style="width: 10%" class="pr-0"></th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr
                      v-for="(fan, index) in calibrationParams.fans"
                      :key="`fan-${index}`"
                    >
                      <td class="pl-0">
                        <v-select
                          v-model="fan.id"
                          :items="availableFansFor(fan.id)"
                          item-text="name"
                          item-value="id"
                          label="Select Fan"
                          class="pt-0"
                          hide-details
                        />
                      </td>
                      <td>
                        <v-text-field
                          v-model.number="fan.speed"
                          type="number"
                          :min="0"
                          :max="255"
                          label="[0-255]"
                          class="pt-0"
                          hide-details
                        />
                      </td>
                      <td class="pr-0">
                        <v-btn icon color="warning" @click="removeFan(index)">
                          <v-icon>mdi-close</v-icon>
                        </v-btn>
                      </td>
                    </tr>
                  </tbody>
                </v-simple-table>
                <v-btn
                  color="blue darken-1"
                  class="mx-auto"
                  outlined
                  text
                  @click="addFan"
                >
                  <v-icon class="mr-1">mdi-plus</v-icon>
                  Add Fan
                </v-btn>
              </div>
              <v-divider class="mb-3" />

              <v-alert dense text class="my-3">
                The machine will calculate new temperature coefficients as soon
                as Next is clicked.
              </v-alert>
            </div>
          </v-window-item>

          <!-- Data Collection -->
          <v-window-item value="calibration">
            <span v-show="!finished">
              Please wait while the probe is being calibrated...
            </span>
            <!-- 
                Include a timer here to show the time taken to calibrate the scanning probe
                <v-timer />?? maybe
            -->
            <!-- Display the current move being recorded -->
            <v-divider />

            <v-alert :value="cancelled" dense text type="error" class="mt-3">
              Scanning Probe Calibration cancelled!
            </v-alert>
            <v-alert :value="finished" dense text type="success" class="mt-3">
              Scanning Probe Calibration completed!
            </v-alert>
          </v-window-item>
        </v-window>
      </v-card-text>

      <v-card-actions>
        <v-btn
          v-show="!cancelled && !finished"
          color="blue darken-1"
          text
          @click="cancel"
        >
          Cancel
        </v-btn>
        <v-spacer />
        <v-btn v-show="canGoBack" color="blue darken-1" text @click="goBack">
          Back
        </v-btn>
        <v-btn
          v-show="currentPage !== 'collection'"
          color="blue darken-1"
          text
          :disabled="!canGoNext"
          @click="goNext"
        >
          Next
        </v-btn>
        <v-btn
          v-show="cancelled || finished"
          color="blue darken-1"
          text
          @click="shownInternal = false"
        >
          Finish
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script>
"use strict";

import { MachineStatus } from "@duet3d/objectmodel";
import { mapActions, mapState } from "vuex";

import { OperationCancelledError } from "@/utils/errors";

export default {
  props: {
    shown: {
      required: true,
      type: Boolean,
    },
  },
  computed: {
    ...mapState("machine/model", ["sensors", "tools", "fans", "heat"]),
    shownInternal: {
      get() {
        return this.shown;
      },
      set(value) {
        this.$emit("update:shown", value);
      },
    },
    hasMultipleScanningProbes() {
      return this.probes.length > 1;
    },
    allAxesHomed() {
      return !this.move.axes.some((axis) => axis.visible && !axis.homed);
    },
    heaters() {
      return this.heat.heaters.map((heater, index) => ({
        ...heater,
        name: `Heater ${index}`,
        id: index,
      }));
    },
    availableFans() {
      return this.fans;
    },
    probes() {
      const annotatedSensors = this.sensors.probes.map((sensor, index) => ({
        ...sensor,
        index,
      }));
      return annotatedSensors.filter((sensor) => sensor.type == 11);
    },
    thermistors() {
      return this.sensors.analog;
    },
    toolList() {
      return [{ text: "None", value: null }].concat(
        this.tools
          .filter((tool) => !!tool)
          .map((tool) => ({
            text: tool.name || tool.number.toString(),
            value: tool,
          }))
      );
    },
    canGoBack() {
      return this.currentPage === "config";
    },
    canGoNext() {
      switch (this.currentPage) {
        case "start":
          return this.probes.length > 0;
        case "config":
        // Write some test cases to ensure that the min/max temps
        // are within the range of the heater
      }
      return false;
    },
  },
  data() {
    return {
      currentPage: "start",
      calibrationParams: {
        thermistor: null,
        heaters: [{ id: null, start: null, stop: null, step: 10 }],
        fans: [{ id: null, speed: 0 }],
      },
      run: 0,
      finished: false,
      cancelled: false,
    };
  },
  methods: {
    ...mapActions("machine", ["sendCode"]),
    async doCode(code) {
      const reply = await this.sendCode(code);
      if (reply.indexOf("Error") === 0) {
        throw new Error(`Code ${code} failed: ${reply}`);
      }
    },
    thermistorRules() {
      return [(v) => !!v || "A thermistor must be selected."];
    },
    startTempRules(maxTemp) {
      return [
        (v) => (v !== null && v !== "") || "Start temperature is required",
        (v) => v >= 0 || "Start temperature must be >= 0",
        (v) => v < maxTemp || `Start temperature must be < ${maxTemp}`,
      ];
    },
    stopTempRules(startValue, maxTemp) {
      return [
        (v) => (v !== null && v !== "") || "Stop temperature is required",
        (v) =>
          v >= startValue ||
          "Stop temperature must be greater than start temperature.",
        (v) => v <= maxTemp || `Stop temperature must be <= ${maxTemp}`,
      ];
    },
    availableFansFor(fanId) {
      const selectedFanIds = this.calibrationParams.fans
        .map((fan) => fan.id)
        .filter((id) => id != fanId);
      const remainingFans = this.fans
        .map((fan, index) => ({
          ...fan,
          id: index,
        }))
        .filter((fan) => fan.actualValue != null)
        .filter((fan) => !selectedFanIds.includes(fan.id));
      return remainingFans;
    },
    availableHeatersFor(heaterId) {
      const selectedHeaterIds = this.calibrationParams.heaters
        .map((heater) => heater.id)
        .filter((id) => id != heaterId);
      const remainingHeaters = this.heat.heaters
        .map((heater, index) => ({
          ...heater,
          name: `Heater ${index}`,
          id: index,
        }))
        .filter((heater) => !selectedHeaterIds.includes(heater.id));
      console.log("remainingHeaters", remainingHeaters);
      return remainingHeaters;
    },
    getHeaterMaxTemp(heaterId) {
      if (heaterId === null || heaterId === undefined) {
        return 0;
      }
      const heaters = this.heat.heaters;
      return heaters[heaterId].max;
    },
    getHeaterLabel(heaterId) {
      const maxTemp = this.getHeaterMaxTemp(heaterId);
      return `0-${maxTemp}`; // Constructing the label string
    },
    getMoveFilename(move) {
      let filename = this.run.toString();
      if (move.tool) {
        filename += "-T" + move.tool.number;
      }
      filename += `-${move.axis.replace(/\+/g, "")}${move.start}-${move.end}-${
        move.accelerometer
      }-${this.move.shaping.type}`;
      if (
        this.move.shaping.type !== "none" &&
        this.move.shaping.type !== "custom"
      ) {
        filename += `-${this.move.shaping.frequency}Hz-${this.move.shaping.damping}`;
      }
      filename += ".csv";
      return filename;
    },
    getStateIcon(completed) {
      return completed ? "mdi-check" : "mdi-clock";
    },
    cancel() {
      if (this.currentPage === "collection" && !this.cancelled) {
        this.cancelled = this.finished = true;
      } else {
        this.shownInternal = false;
      }
    },
    goBack() {
      switch (this.currentPage) {
        case "config":
          this.currentPage = "start";
          break;
      }
    },
    goNext() {
      switch (this.currentPage) {
        case "start":
          this.currentPage = "config";
          break;
        case "config":
          this.currentPage = "calibration";
          this.cancelled = false;
          this.record();
          break;
      }
    },
    addHeater() {
      this.calibrationParams.heaters.push({
        id: null,
        start: null,
        stop: null,
        step: 10,
      });
    },
    addFan() {
      console.log("Before adding:", this.calibrationParams.fans);
      this.calibrationParams.fans.push({ id: null, speed: 0 });
      console.log("After adding:", this.calibrationParams.fans);
    },
    removeHeater(index) {
      this.calibrationParams.heaters.splice(index, 1);
    },
    removeFan(index) {
      this.calibrationParams.fans.splice(index, 1);
    },
  },
  mounted() {},
  watch: {
    shown(to) {
      if (to) {
      } else {
        if (this.currentPage === "calibration") {
          // This will interrupt the current sampling
          this.cancelled = true;
        }
        this.currentPage = "start";
        this.cancelled = this.finished = false;
      }
    },
    "state.status"(to) {
      if (
        (to === MachineStatus.disconnected || to === MachineStatus.off) &&
        this.currentPage === "calibration"
      ) {
        this.cancelled = true;
      }
    },
  },
};
</script>
