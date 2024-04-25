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
            This wizard lets you calibrate your Duet scanning probe by using the
            thermistor attached to the coil.<br />

            <!-- List available scanning probes and their coefficients -->
            <div v-if="probes.length > 0">
              <ul class="mt-3 mb-4">
                <li v-for="probe in probes" :key="probe.id">
                  <strong>Scanning Probe: Index #{{ probe.id }}</strong>
                  <ul>
                    <li>
                      Current temp coefficients:
                      {{ probe.temperatureCoefficients }}
                    </li>
                    <li>
                      Current calibration temp:
                      {{ probe.calibrationTemperature }}
                    </li>
                  </ul>
                </li>
              </ul>
            </div>

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
              This machine has multiple scanning probes. If you are operating a
              tool changer, you will need to calibrate each scanning probe
              separately.
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

            <!-- Instruction to continue -->
            <span v-show="probes.length > 0"> Press Next to continue. </span>
          </v-window-item>

          <!-- Configuration -->
          <v-window-item value="config">
            <div class="d-flex flex-column">
              Set up parameters for the calibration process.
              <br />
              <div>
                <div class="text-subtitle-1 mt-3">
                  Select the scanning probe to calibrate
                </div>
                <v-row>
                  <v-col>
                    <v-select
                      v-model="calibrationParams.selectedScanningProbeId"
                      :items="
                        probes.map((probe) => ({
                          text: `Probe #${probe.id}`,
                          value: probe.id,
                        }))
                      "
                      item-text="text"
                      item-value="value"
                      label="Select Scanning Probe Index"
                      class="mb-3"
                      hide-details
                      :rules="probeIsRequired"
                    ></v-select>
                  </v-col>
                  <v-col>
                    <v-select
                      v-model="calibrationParams.selectedThermistor"
                      :items="thermistors"
                      item-text="name"
                      item-value="id"
                      label="Select Scanning Probe Thermistor"
                      class="mb-1"
                      hide-details
                      :rules="thermistorIsRequired"
                    ></v-select>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col>
                    <div class="text-subtitle-1">
                      Select the tool to which the scanning probe is attached
                    </div>
                    <v-select
                      v-model="calibrationParams.selectedTool"
                      :items="toolList"
                      item-text="text"
                      item-value="value"
                      label="Select ScanningProbeTool"
                      class="mb-3"
                      hide-details
                      :rules="toolIsRequired"
                    ></v-select>
                  </v-col>
                </v-row>
              </div>
              <div>
                <v-simple-table class="mt-3">
                  <thead>
                    <tr>
                      <th class="pl-0">Heater</th>
                      <th style="width: 20%">Start (°C)</th>
                      <th style="width: 20%">Stop (°C)</th>
                      <th style="width: 20%" class="pr-0">Step (°C)</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr
                      v-for="(heater, index) in calibrationParams.bedHeater"
                      :key="`heater-${index}`"
                    >
                      <td class="px-0">
                        <v-select
                          v-model="heater.id"
                          :items="bedHeaters"
                          item-text="name"
                          item-value="id"
                          label="Bed Heater"
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
                    </tr>
                  </tbody>
                </v-simple-table>
              </div>
              <v-divider class="mt-3" />
              <v-checkbox
                v-model="showChamberHeatersConfig"
                label="Add a Chamber Heater to the calibration process"
                :disabled="
                  !showChamberHeatersConfig && chamberHeaterIds.length === 0
                "
              ></v-checkbox>
              <div v-if="showChamberHeatersConfig">
                <v-row>
                  <v-col>
                    <v-text-field
                      v-model="calibrationParams.chamberHeaterStart"
                      type="number"
                      label="Start Temperature (°C)"
                      hide-details
                      :min="0"
                      :max="getChamberHeaterMaxTemp()"
                      :rules="startTempRules(getChamberHeaterMaxTemp())"
                    />
                  </v-col>

                  <v-col>
                    <v-text-field
                      v-model="calibrationParams.chamberHeaterStop"
                      type="number"
                      label="Stop Temperature (°C)"
                      hide-details
                      :min="calibrationParams.chamberHeaterStart"
                      :max="getChamberHeaterMaxTemp()"
                      :rules="
                        stopTempRules(
                          calibrationParams.chamberHeaterStart,
                          getChamberHeaterMaxTemp()
                        )
                      "
                    />
                  </v-col>
                </v-row>
              </div>
              <div>
                <v-alert type="warning" dense>
                  You should enclose your printer if possible to help simulate
                  higher probe temperatures.
                </v-alert>
                <!-- Fan calbiration params -->
              </div>
              <v-divider class="mx-auto" />
              <v-checkbox
                v-model="showFanConfig"
                label="Add a Fan to the calibration process"
                :disabled="!showFanConfig && availableFans().length === 0"
              ></v-checkbox>
              <div v-if="showFanConfig">
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
                  class="px-3 mt-3 mb-3"
                  outlined
                  text
                  @click="addFan"
                >
                  <v-icon class="mr-1">mdi-plus</v-icon>
                  Add Fan
                </v-btn>
              </div>
              <v-divider class="mx-auto" />
              <v-alert dense text>
                The machine will calculate new temperature coefficients as soon
                as Next is clicked.
              </v-alert>
            </div>
          </v-window-item>

          <!-- Data Collection -->
          <v-window-item value="calibration">
            <v-container>
              <v-alert
                dense
                text
                :type="allAxesHomed || isBusy ? 'success' : 'warning'"
                class="mb-3"
              >
                <v-row align="center">
                  <v-col>
                    {{
                      allAxesHomed
                        ? "Machine is homed and ready for calibration!"
                        : "Machine is not homed! Home it before starting calibration."
                    }}
                  </v-col>

                  <v-col class="shrink">
                    <v-btn small v-if="!allAxesHomed" @click="homeMachine">
                      <v-icon left>mdi-home</v-icon> Home Machine
                    </v-btn>
                  </v-col>

                  <v-col class="shrink">
                    <v-progress-circular
                      v-if="isBusy"
                      indeterminate
                      color="blue"
                      size="24"
                    >
                    </v-progress-circular>
                  </v-col>
                </v-row>
              </v-alert>
            </v-container>

            <v-container>
              <div v-if="!finished">
                <h2>Calibration in Progress</h2>
                <v-data-table
                  :headers="calibrationTableHeaders"
                  :items="calibrationProgress"
                  hide-default-footer
                  dense
                >
                  <template v-slot:item="{ item }">
                    <tr>
                      <td class="pa-3">
                        <v-icon>
                          {{ getStateIcon(item.status) }}
                        </v-icon>
                      </td>
                      <td class="pa-3 text-left">{{ item.index }}</td>
                      <td
                        class="pa-3 text-left"
                        v-if="item.currentTemp == null"
                      ></td>
                      <td class="pa-3 text-left" v-else>
                        Current Temp: {{ item.currentTemp }}°C
                      </td>
                      <td class="pa-3 text-left">
                        Target Temp: {{ item.targetTemp }}°C
                      </td>
                      <td
                        class="pa-3 text-left"
                        v-if="item.timeElapsed == null"
                      ></td>
                      <td class="pa-3 text-left" v-else>
                        Time Elapsed: {{ item.timeElapsed }}
                      </td>
                    </tr>
                  </template>
                </v-data-table>
                <div v-if="!calibrationStarted">
                  <v-alert type="info" dense>
                    Press the button below to start the calibration process.
                  </v-alert>
                  <v-btn color="blue darken-1" @click="startCalibration"
                    >Start Calibration</v-btn
                  >
                  <v-btn color="gray" @click="goBack"
                    >Go Back to Configuration</v-btn
                  >
                </div>
                <div v-else>
                  <v-alert type="info" dense>
                    Calibration in progress. Please wait...
                  </v-alert>
                  <v-btn color="red" v-if="!cancelled" @click="cancel"
                    >Cancel Calibration</v-btn
                  >
                </div>
              </div>
              <div v-else>
                <h2>Calibration Finished</h2>
                <!-- Display calibration results or next steps -->
              </div>
            </v-container>
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
    // Internal computed properties
    ...mapState("machine/model", [
      "sensors",
      "move",
      "tools",
      "fans",
      "heat",
      "state",
    ]),
    shownInternal: {
      get() {
        return this.shown;
      },
      set(value) {
        this.$emit("update:shown", value);
      },
    },
    //
    // Getters
    //
    bedHeaterIds() {
      return Object.keys(this.heat.bedHeaters)
        .filter((key) => this.heat.bedHeaters[key] !== -1)
        .map((key) => Number(key));
    },
    chamberHeaterIds() {
      return Object.keys(this.heat.chamberHeaters)
        .filter((key) => this.heat.chamberHeaters[key] !== -1)
        .map((key) => Number(key));
    },
    bedHeaters() {
      return this.heaters.filter((heater) =>
        this.bedHeaterIds.includes(heater.id)
      );
    },
    chamberHeaters() {
      return this.heaters.filter((heater) =>
        this.chamberHeaterIds.includes(heater.id)
      );
    },
    heaters() {
      return this.heat.heaters.map((heater, index) => ({
        ...heater,
        name: `Heater ${index}`,
        id: index,
      }));
    },
    getChamberHeaterMaxTemp() {
      return Math.max(...this.chamberHeaters().map((heater) => heater.max));
    },
    getChamberHeaterMinTemp() {
      return Math.min(...this.chamberHeaters().map((heater) => heater.min));
    },
    probes() {
      const annotatedSensors = this.sensors.probes.map((sensor, index) => ({
        ...sensor,
        id: index,
      }));
      return annotatedSensors.filter((sensor) => sensor.type == 11);
    },
    thermistors() {
      return this.sensors.analog.map((sensor, index) => ({
        ...sensor,
        id: index,
      }));
    },
    toolList() {
      return this.tools
        .filter((tool) => tool)
        .map((tool) => ({
          text: tool.name || tool.number.toString(),
          value: tool,
        }));
    },
    getHeaterMaxTemp() {
      return (heaterId) => {
        if (heaterId === null || heaterId === undefined) {
          return 0;
        }
        const heaters = this.heat.heaters;
        return heaters[heaterId].max;
      };
    },
    //
    // Booleans
    //
    isBusy() {
      return this.state.status === MachineStatus.busy;
    },
    hasMultipleScanningProbes() {
      return this.probes.length > 1;
    },
    allAxesHomed() {
      return !this.move.axes.some((axis) => axis.visible && !axis.homed);
    },
    //
    // Page navigation
    //
    canGoBack() {
      return this.currentPage === "config";
    },
    canGoNext() {
      switch (this.currentPage) {
        case "start":
          return this.probes.length > 0;
        case "config":
          return this.isConfigPageValid;
      }
      return false;
    },
    isConfigPageValid() {
      const isScanningProbeSelected =
        this.calibrationParams.selectedScanningProbeId !== null;
      const isToolSelected = this.calibrationParams.selectedTool !== null;
      const isThermistorSelected = !!this.calibrationParams.selectedThermistor;
      const heatersValid = this.calibrationParams.bedHeater.every(
        (heater) =>
          heater.id !== null &&
          heater.start < heater.stop &&
          heater.start >= 0 &&
          heater.stop <= this.getHeaterMaxTemp(heater.id) &&
          heater.step > 0
      );
      let chamberHeatersValid = true;
      if (this.showChamberHeatersConfig) {
        chamberHeatersValid = this.calibrationParams.chamberHeaters.every(
          (heater) =>
            heater.id !== null &&
            heater.start < heater.stop &&
            heater.start >= 0 &&
            heater.stop <= this.getHeaterMaxTemp(heater.id)
        );
      }
      let fansValid = true;
      if (this.showFanConfig) {
        fansValid = this.calibrationParams.fans.every(
          (fan) => fan.id !== null && fan.speed >= 0 && fan.speed <= 255
        );
      }

      return (
        isScanningProbeSelected &&
        isToolSelected &&
        isThermistorSelected &&
        heatersValid &&
        chamberHeatersValid &&
        fansValid
      );
    },
    //
    // Rules
    //
    toolIsRequired() {
      return [(v) => v !== null || "Tool selection is required"];
    },
    probeIsRequired() {
      return [(v) => v !== null || "Probe index is required"];
    },
    thermistorIsRequired() {
      return [(v) => v !== null || "A thermistor must be selected."];
    },
    startTempRules() {
      return (maxTemp) => [
        (v) => (v !== null && v !== "") || "Start temperature is required",
        (v) => Number(v) >= 0 || "Start temperature must be >= 0",
        (v) =>
          Number(v) <= maxTemp || `Start temperature must be <= ${maxTemp}`,
      ];
    },
    stopTempRules() {
      return (start, maxTemp) => [
        (v) => (v !== null && v !== "") || "Stop temperature is required",
        (v) =>
          Number(v) >= start || "Stop temperature must be >= start temperature",
        (v) => Number(v) <= maxTemp || `Stop temperature must be <= ${maxTemp}`,
      ];
    },
  },
  data() {
    return {
      currentPage: "start",
      showFanConfig: false,
      showChamberHeatersConfig: false,
      calibrationParams: {
        selectedTool: null,
        selectedThermistor: null,
        selectedScanningProbeId: null,
        bedHeater: [{ id: null, start: null, stop: null, step: null }],
        chamberHeaters: [{ id: null, start: null, stop: null }],
        fans: [{ id: null, speed: 0 }],
      },
      run: 0,

      calibrationStarted: false,
      finished: false,
      cancelled: false,

      calibrationValues: [],
      startTime: 0,
      calibrationProgress: [],
      calibrationTableHeaders: [
        { text: "Status", align: "start", value: "status" },
        { text: "Index", sortable: false, value: "index" },
        { text: "Current Temp", value: "currentTemp" },
        { text: "Target Temp", value: "targetTemp" },
        { text: "Time Elapsed", value: "timeElapsed" },
      ],
    };
  },
  methods: {
    ...mapActions("machine", ["sendCode"]),
    //
    // Initialization methods
    //
    initializeCalibrationParams() {
      if (this.probes.length === 1) {
        this.calibrationParams.selectedScanningProbeId = this.probes[0].id;
      }
      if (this.thermistors.length === 1) {
        this.calibrationParams.selectedThermistor = this.thermistors[0].id;
      }
      if (this.toolList.length === 1) {
        this.calibrationParams.selectedTool = this.toolList[0].value;
      }
      if (this.bedHeaters.length === 1) {
        this.calibrationParams.bedHeater[0].id = this.bedHeaters[0].id;
      }
    },
    //
    // Getters
    //
    getAvailableItems(allItems, selectedIds) {
      return allItems
        .map((item, index) => ({
          ...item,
          id: index,
        }))
        .filter((item) => !selectedIds.includes(item.id));
    },
    availableFans() {
      const selectedFanIds = this.calibrationParams.fans.map((fan) => fan.id);
      const allFans = this.fans;
      return this.getAvailableItems(allFans, selectedFanIds);
    },
    availableHeaters() {
      const selectedHeaters = this.calibrationParams.bedHeater.map(
        (heater) => heater.id
      );
      const selectedChamberHeaters = this.calibrationParams.chamberHeaters.map(
        (heater) => heater.id
      );
      const selectedHeaterIds = selectedHeaters.concat(selectedChamberHeaters);
      const allHeaters = this.heat.heaters;
      return this.getAvailableItems(allHeaters, selectedHeaterIds);
    },
    availableFansFor(fanId) {
      const selectedFanIds = this.calibrationParams.fans
        .map((fan) => fan.id)
        .filter((id) => id != fanId);
      const allFans = this.fans;
      return this.getAvailableItems(allFans, selectedFanIds);
    },
    getHeaterLabel(heaterId) {
      const maxTemp = this.getHeaterMaxTemp(heaterId);
      return `0-${maxTemp}`;
    },
    getStateIcon(completed) {
      if (completed === null) {
        return null;
      }
      return completed ? "mdi-check" : "mdi-clock";
    },
    //
    // Heater and Fan Operations
    //
    async doCode(code) {
      const reply = await this.sendCode(code);
      if (reply.indexOf("Error") === 0) {
        throw new Error(`Code ${code} failed: ${reply}`);
      }
    },
    async homeMachine() {
      // Need to include the toolId mapping to the homing probe
      await this.doCode("G28");
    },
    async enableBedHeater(temp) {
      await this.doCode(`M140 S${temp} G4 P1000`);
    },
    async disableBedHeater() {
      await this.doCode("M140 S-273.1 G4 P1000");
    },
    async enableChamberHeater(currentBedTemp) {
      const enableChamberHeaters =
        this.showChamberHeatersConfig &&
        this.calibrationParams.chamberHeaters.length > 0;
      if (enableChamberHeaters) {
        const heaters = this.calibrationParams.chamberHeaters;
        const chamberTemp = Number.min(heater.stop, currentBedTemp);
        await this.doCode(`M141 S${currentBedTemp} G4 P1000`);
      }
    },
    async disableChamberHeaters() {
      const heaters = this.calibrationParams.chamberHeaters;
      for (const heater of heaters) {
        await this.doCode(`M141 S-273.1 G4 P1000`);
      }
    },
    async enableFan(fanId, speed) {
      await this.doCode(`M106 P${fanId} S${speed} G4 P1000`);
    },
    async disableFan(fanId) {
      await this.enableFan(fanId, 0);
    },
    async enableFans() {
      const enableFan =
        this.showFanConfig && this.calibrationParams.fans.length > 0;
      if (enableFan) {
        const fans = this.calibrationParams.fans;
        for (const fan of fans) {
          await this.enableFan(fan.id, fan.speed);
        }
      }
    },
    async disableFans() {
      const fans = this.calibrationParams.fans;
      for (const fan of fans) {
        await disableFan(fan.id);
      }
    },
    addChamberHeater() {
      this.calibrationParams.chamberHeaters.push({
        id: null,
        start: null,
        stop: null,
      });
    },
    addFan() {
      this.calibrationParams.fans.push({ id: null, speed: 0 });
    },
    removeChamberHeater(heaterId) {
      this.calibrationParams.chamberHeaters.splice(heaterId, 1);
    },
    removeFan(fanId) {
      this.calibrationParams.fans.splice(fanId, 1);
    },
    //
    // Pre Calibration
    //
    calculateTemps() {
      // Based on the steps, calculate the temperatures to be used
      const heater = this.calibrationParams.bedHeater[0]; // Bed heater
      const n = Math.ceil(
        (Number(heater.stop) - Number(heater.start)) / Number(heater.step)
      );
      const temps = Array.from({ length: n }, (_, i) =>
        Math.min(
          Number(heater.start) + i * Number(heater.step),
          Number(heater.stop)
        )
      );
      if (Number(heater.stop) !== temps[temps.length - 1]) {
        temps.push(Number(heater.stop));
      }
      return temps;
    },
    loadCalibrationPage() {
      this.calibrationValues = [];
      this.finished = false;
      this.cancelled = false;
      this.startTime = Date.now();
      this.populateCalibrationProgress();
    },
    populateCalibrationProgress() {
      const temps = this.calculateTemps();
      this.calibrationProgress = temps.map((targetTemp, index) => ({
        status: null,
        index,
        currentTemp: null,
        targetTemp,
        timeElapsed: null,
      }));
    },
    //
    // Calibration Process
    //
    async startCalibration() {
      this.calibrationStarted = true;
      await this.startMeasurement();
    },
    async startMeasurement() {
      const probeId = this.calibrationParams.selectedScanningProbeId;
      const bedHeaterId = this.calibrationParams.bedHeater[0].id;
      const temps = this.calibrationProgress.map((item) => item.targetTemp);

      await this.enableFans();
      for (let i = 0; i < temps.length; i++) {
        const temp = temps[i];
        this.startTime = Date.now();
        this.setCalibrationStatus(i, false);
        await this.waitForStabilization(bedHeaterId, temp, probeId, i);
        await this.soakAtTargetTemperature(bedHeaterId, probeId, i);
        this.setCalibrationStatus(i, true);
      }
      await this.finishMeasurement();
    },
    setCalibrationStatus(index, status) {
      this.calibrationProgress[index].status = status;
    },
    async waitForStabilization(bedHeaterId, targetTemp, probeId, i) {
      let isStable = false;
      await this.enableBedHeater(targetTemp);
      // Wait until the bed heater reaches target temp
      while (!isStable) {
        isStable = this.isTemperatureStable(bedHeaterId);

        if (!isStable) {
          await this.delay(9000);
          await this.recordProbeValue(probeId);
          this.updateCalibrationProgress(i, this.getHeaterTemp(bedHeaterId));
        }
      }
    },
    async soakAtTargetTemperature(bedHeaterId, probeId, i) {
      const soakStartTime = Date.now();
      const totalSoakTime = 300000; // 5 minutes in milliseconds

      while (totalSoakTime < Date.now() - soakStartTime) {
        await this.delay(9000);
        await this.recordProbeValue(probeId);
        this.updateCalibrationProgress(i, this.getHeaterTemp(bedHeaterId));
      }
    },
    updateCalibrationProgress(index, currentTemp) {
      this.calibrationProgress[index].currentTemp = currentTemp;
      this.calibrationProgress[index].timeElapsed = this.formatTime(
        Date.now() - this.startTime
      );
    },
    async finishMeasurement() {
      await this.disableBedHeater();
      await this.disableChamberHeaters();
      await this.disableFans();

      const filename = this.getSaveFilename();
      this.saveMeasurement(filename);
      this.downloadCSV(this.calibrationValues, filename);

      this.finished = true;
    },

    saveMeasurement(filename) {
      const calibrationKey = `scanningProbeCalibration_${filename}`;
      this.saveToLocalStorage(calibrationKey, this.calibrationValues);
    },

    getSaveFilename() {
      const time = new Date().toISOString().replace(/:/g, "-");
      const probeId = this.calibrationParams.selectedScanningProbeId;
      return `scanning-probe-index-${probeIndex}-calibration-${time}.csv`;
    },
    async recordProbeValue(probeId) {
      const heaterId = this.calibrationParams.bedHeater[0].id;
      const currentTargetBedTemp = this.heat.heaters[heaterId].active;

      await this.disableBedHeater();

      const currentBedTemp = this.getHeaterTemp(heaterId);
      const currentProbeValue = this.getScanningProbeValue(probeId);
      const currentProbeTemp = this.getScanningProbeTemp();
      await this.delay(1000);
      await this.enableBedHeater(currentTargetBedTemp);
      this.calibrationValues.push([
        currentBedTemp,
        currentProbeTemp,
        currentProbeValue,
      ]);
    },
    delay(ms) {
      return new Promise((resolve) => setTimeout(resolve, ms));
    },
    isTemperatureStable(heaterId) {
      const currentTemp = this.getHeaterTemp(heaterId);
      const targetTemp = this.calibrationParams.bedHeater[0].id;
      return Math.abs(currentTemp - targetTemp) < 1;
    },
    formatTime(ms) {
      const minutes = Math.floor(ms / (1000 * 60));
      const seconds = Math.floor((ms % (1000 * 60)) / 1000);
      return `${minutes}m ${seconds}s`;
    },
    //
    // Getters
    //
    getScanningProbeTriggerHeight(probeId) {
      return this.probes[probeId].getScanningProbeTriggerHeight;
    },
    getScanningProbeTemp() {
      return this.getThermistorReading(
        this.calibrationParams.selectedThermistor
      );
    },
    getThermistorReading(thermistorId) {
      return this.thermistors[thermistorId].lastReading;
    },
    getScanningProbeValue(probeId) {
      const probe = this.probes.find((p) => p.id === probeId);
      return probe ? probe.value[0] : null;
    },
    getHeaterTemp(heaterId) {
      const heater = this.heaters.find((h) => h.id === heaterId);
      return heater ? heater.current : null;
    },
    //
    // Page operations
    //
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
        case "calibration":
          this.currentPage = "config";
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
          this.loadCalibrationPage();
          break;
      }
    },
    //
    // File operations
    //
    formatCalibrationValuesToCSV(calibrationValues) {
      const headers = ["Bed Temp (°C)", "Probe Temp (°C)", "Probe Value"];
      const rows = calibrationValues.map((value) => {
        const [bedTemp, probeTemp, probeValue] = value;
        return `${bedTemp},${probeTemp},${probeValue}`;
      });
      return [headers.join(","), ...rows].join("\n");
    },

    saveToLocalStorage(key, calibrationValues) {
      const csvContent = this.formatCalibrationValuesToCSV(calibrationValues);
      localStorage.setItem(key, csvContent);
    },

    downloadCSV(calibrationValues, filename) {
      const csvContent = this.formatCalibrationValuesToCSV(calibrationValues);

      const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });

      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = filename;

      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);

      URL.revokeObjectURL(link.href);
    },
  },
  created() {
    this.initializeCalibrationParams();
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
    probes(newProbes) {
      if (newProbes.length === 1) {
        this.calibrationParams.selectedScanningProbeId = newProbes[0].id;
      }
    },
    tools(newTools) {
      if (newTools.length === 1) {
        this.calibrationParams.selectedTool = newTools[0];
      }
    },
    bedHeaters(newHeaters) {
      if (newHeaters.length === 1) {
        this.calibrationParams.bedHeater[0].id = newHeaters[0].id;
      }
  },
};
</script>
