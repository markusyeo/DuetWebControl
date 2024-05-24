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
        <span class="headline">Calibrate Scanning Probe</span>
      </v-card-title>

      <v-card-text class="pb-0">
        <v-window v-model="currentPage">
          <!-- Start -->
          <v-window-item value="start">
            This wizard lets you calibrate your Duet scanning probe by using the
            thermistor attached to the coil.<br />
            <!-- List available scanning probes and their coefficients -->
            <div v-if="scanningProbes.length > 0">
              <ul class="mt-3 mb-4">
                <li v-for="probe in scanningProbes" :key="probe.id">
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
              :value="scanningProbes.length === 0"
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
                >Help</a
              >
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
              :value="scanningProbes.length > 0"
              dense
              text
              type="success"
              class="my-3"
            >
              Ready to calibrate scanning probe!
            </v-alert>
            <span v-show="scanningProbes.length > 0"
              >Press Next to continue.</span
            >
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
                      v-model="calibrationParams.selectedScanningProbe"
                      :items="
                        scanningProbes.map((probe) => ({
                          text: `Probe #${probe.id}`,
                          probe: probe,
                        }))
                      "
                      item-text="text"
                      item-value="probe"
                      label="Select Scanning Probe"
                      class="mb-3"
                      hide-details
                      :rules="scanningProbeIsRequired"
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
                      label="Select Scanning Probe Tool"
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
                    <tr index="bedHeater">
                      <td class="px-0">
                        <v-select
                          v-model="calibrationParams.selectedBedHeater"
                          :items="
                            bedHeaters.map((heater) => ({
                              text: heater.name,
                              heater: heater,
                            }))
                          "
                          item-text="text"
                          item-value="heater"
                          label="Bed Heater"
                          class="pt-0"
                          hide-details
                        ></v-select>
                      </td>
                      <td style="width: 20%">
                        <v-text-field
                          v-model="calibrationParams.bedHeaterStart"
                          type="number"
                          :min="0"
                          :max="
                            getHeaterMaxTemp(
                              calibrationParams.selectedBedHeater
                            )
                          "
                          :label="getBedHeaterStartLabel()"
                          class="pt-0"
                          hide-details
                          :rules="
                            startTempRules(
                              getHeaterMaxTemp(
                                calibrationParams.selectedBedHeater
                              )
                            )
                          "
                        ></v-text-field>
                      </td>
                      <td style="width: 20%">
                        <v-text-field
                          v-model="calibrationParams.bedHeaterStop"
                          type="number"
                          :min="calibrationParams.bedHeaterStart"
                          :max="
                            getHeaterMaxTemp(
                              calibrationParams.selectedBedHeater
                            )
                          "
                          :label="getBedHeaterStopLabel()"
                          class="pt-0"
                          hide-details
                          :rules="
                            stopTempRules(
                              calibrationParams.bedHeaterStart,
                              getHeaterMaxTemp(
                                calibrationParams.selectedBedHeater
                              )
                            )
                          "
                        ></v-text-field>
                      </td>
                      <td style="width: 20%">
                        <v-text-field
                          v-model="calibrationParams.bedHeaterStep"
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
              <div v-if="!calibrationFinished">
                <v-alert
                  v-if="!calibrationStarted"
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
                        v-if="isBusy && !calibrationStarted"
                        indeterminate
                        color="blue"
                        size="24"
                      >
                      </v-progress-circular>
                    </v-col>
                  </v-row>
                </v-alert>
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
                        <v-icon>{{ getStateIcon(item.status) }}</v-icon>
                      </td>
                      <td class="pa-3 text-center">{{ item.index }}</td>
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
                  <v-alert type="warning" dense v-if="checkStartTemperature()">
                    The current bed temperature is higher than the start
                    temperature. If you continue, the calibration process will
                    start at the next temperature step above
                    {{ getBedHeaterCurrentTemp }}°C.
                  </v-alert>
                  <v-alert type="info" dense v-if="m558_1Executed">
                    Press the button below to start the calibration process.<br />
                    M558.1 K{{ calibrationParams.selectedScanningProbe.id }}
                    S0.5 command will be executed when you click it.
                  </v-alert>
                  <v-btn
                    :disabled="!allAxesHomed"
                    color="blue darken-1"
                    @click="startCalibration"
                    >Start Calibration</v-btn
                  >
                </div>
                <div v-else>
                  <v-alert type="info" dense>
                    Calibration in progress. Please wait...
                  </v-alert>
                  <v-btn
                    color="red"
                    v-if="!calibrationCancelled"
                    @click="cancel"
                    >Cancel Calibration</v-btn
                  >
                </div>
              </div>
              <div v-else>
                <h2>Calibration Finished</h2>
                <v-btn color="blue darken-1" @click="downloadCalibrationResults"
                  >Download Calibration Results</v-btn
                >
              </div>
            </v-container>
            <v-divider />
            <v-alert
              :value="calibrationCancelled"
              dense
              text
              type="error"
              class="mt-3"
            >
              Scanning Probe Calibration cancelled!
            </v-alert>
            <v-alert
              :value="calibrationFinished"
              dense
              text
              type="success"
              class="mt-3"
            >
              Scanning Probe Calibration completed!
            </v-alert>
          </v-window-item>
        </v-window>
      </v-card-text>

      <v-card-actions>
        <v-btn
          v-show="!calibrationCancelled && !calibrationFinished"
          color="blue darken-1"
          text
          @click="cancel"
          >Cancel</v-btn
        >
        <v-spacer />
        <v-btn v-show="canGoBack" color="blue darken-1" text @click="goBack"
          >Back</v-btn
        >
        <v-btn
          v-show="currentPage !== 'calibration'"
          color="blue darken-1"
          text
          :disabled="!canGoNext"
          @click="goNext"
          >Next</v-btn
        >
        <v-btn
          :disabled="!calibrationCancelled || !calibrationFinished"
          v-show="currentPage === 'calibration'"
          color="blue darken-1"
          text
          @click="shownInternal = false"
          >Finish</v-btn
        >
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script>
"use strict";

import { MachineStatus } from "@duet3d/objectmodel";
import { mapActions, mapState } from "vuex";

export default {
  props: {
    shown: {
      required: true,
      type: Boolean,
    },
  },
  data() {
    return {
      currentPage: "start",
      // Configuration
      showFanConfig: false,
      showChamberHeatersConfig: false,
      calibrationParams: {
        selectedTool: null,
        selectedThermistor: null,
        selectedScanningProbe: null,
        selectedBedHeater: null,
        bedHeaterStart: null,
        bedHeaterStop: null,
        bedHeaterStep: null,
        chamberHeaterStart: null,
        chamberHeaterStop: null,
        fans: [{ id: null, speed: 0 }],
      },
      // Calibration process
      calibrationStarted: false,
      calibrationFinished: false,
      calibrationCancelled: false,
      calibrationResults: { calibrationValues: [], coefficients: {} },
      startTime: 0,
      calibrationProgress: [],
      m558_1Executed: false,
      calibrationTableHeaders: [
        { text: "Status", align: "start", value: "status" },
        { text: "Index", sortable: false, value: "index" },
        { text: "Current Temp", value: "currentTemp" },
        { text: "Target Temp", value: "targetTemp" },
        { text: "Time Elapsed", value: "timeElapsed" },
      ],
    };
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
    heaters() {
      return this.heat.heaters.map((heater, index) => ({
        ...heater,
        name: `Heater ${index}`,
        id: index,
      }));
    },
    bedHeaterIds() {
      return Object.keys(this.heat.bedHeaters)
        .filter((key) => this.heat.bedHeaters[key] !== -1)
        .map((key) => Number(this.heat.bedHeaters[key]));
    },
    chamberHeaterIds() {
      return Object.keys(this.heat.chamberHeaters)
        .filter((key) => this.heat.chamberHeaters[key] !== -1)
        .map((key) => Number(this.heat.chamberHeaters[key]));
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
    getHeaterMaxTemp() {
      return (heater) => (heater ? heater.max : 0);
    },
    getHeaterMinTemp() {
      return (heater) => (heater ? heater.min : 0);
    },
    getChamberHeaterMaxTemp() {
      return Math.max(...this.chamberHeaters().map((heater) => heater.max));
    },
    getChamberHeaterMinTemp() {
      return Math.min(...this.chamberHeaters().map((heater) => heater.min));
    },
    scanningProbes() {
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
      return this.tools.map((tool, index) => ({
        text: tool.name || index.toString(),
        value: { ...tool, id: index },
      }));
    },
    //
    // Calibration Params Getters
    //
    selectedTool() {
      return this.calibrationParams.selectedTool;
    },
    selectedScanningProbe() {
      return this.calibrationParams.selectedScanningProbe;
    },
    selectedBedHeater() {
      return this.calibrationParams.selectedBedHeater;
    },
    getBedHeaterActiveTemp() {
      return this.selectedBedHeater.active;
    },
    getBedHeaterCurrentTemp() {
      return this.selectedBedHeater.current;
    },
    getThermistorReading() {
      return (thermistorId) => this.thermistors[thermistorId].lastReading;
    },
    getScanningProbeTemp() {
      return this.getThermistorReading(
        this.calibrationParams.selectedThermistor
      );
    },
    //
    // Booleans
    //
    isBusy() {
      return this.state.status === MachineStatus.busy;
    },
    hasMultipleScanningProbes() {
      return this.scanningProbes.length > 1;
    },
    allAxesHomed() {
      return !this.move.axes.some((axis) => axis.visible && !axis.homed);
    },
    //
    // Page navigation
    //
    canGoBack() {
      if (this.currentPage === "config") {
        return true;
      }
      if (this.currentPage === "calibration") {
        return !this.calibrationStarted;
      }
    },
    canGoNext() {
      switch (this.currentPage) {
        case "start":
          return this.scanningProbes.length > 0;
        case "config":
          return this.isConfigPageValid;
      }
      return false;
    },
    isConfigPageValid() {
      const isScanningProbeSelected =
        this.calibrationParams.selectedScanningProbe !== null;
      const isToolSelected = this.calibrationParams.selectedTool !== null;
      const isThermistorSelected = !!this.calibrationParams.selectedThermistor;
      const bedHeater = this.selectedBedHeater;
      const bedHeaterValid =
        bedHeater !== null &&
        this.calibrationParams.bedHeaterStart <
          this.calibrationParams.bedHeaterStop &&
        this.calibrationParams.bedHeaterStart >= 0 &&
        this.calibrationParams.bedHeaterStop <= bedHeater.max &&
        this.calibrationParams.bedHeaterStep > 0;
      const chamberHeatersValid =
        !this.showChamberHeatersConfig ||
        (this.calibrationParams.chamberHeaterStart !== null &&
          this.calibrationParams.chamberHeaterStop !== null);
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
        bedHeaterValid &&
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
    scanningProbeIsRequired() {
      return [(v) => v !== null || "Probe index is required"];
    },
    thermistorIsRequired() {
      return [(v) => v !== null || "A thermistor must be selected."];
    },
    startTempRules() {
      return (maxTemp) => [
        (v) => (v !== null && v !== "") || "Start temperature is required",
        (v) => Number(v) >= 0 || "Start temperature must be >= 0",
        (v) => Number(v) < maxTemp || `Start temperature must be <= ${maxTemp}`,
      ];
    },
    stopTempRules() {
      return (start, maxTemp) => [
        (v) => (v !== null && v !== "") || "Stop temperature is required",
        (v) =>
          Number(v) > start || "Stop temperature must be >= start temperature",
        (v) => Number(v) <= maxTemp || `Stop temperature must be <= ${maxTemp}`,
      ];
    },
  },
  methods: {
    ...mapActions("machine", ["sendCode"]),
    async doCode(code) {
      const reply = await this.sendCode(code);
      if (reply.indexOf("Error") === 0) {
        throw new Error(`Code ${code} failed: ${reply}`);
      }
    },
    //
    // Initialization methods
    //
    initializeCalibrationParams() {
      if (this.scanningProbes.length === 1) {
        this.calibrationParams.selectedScanningProbe = this.scanningProbes[0];
      }
      if (this.thermistors.length === 1) {
        this.calibrationParams.selectedThermistor = this.thermistors[0].id;
      }
      if (this.toolList.length === 1) {
        this.calibrationParams.selectedTool = this.toolList[0].value;
      }
      if (this.bedHeaters.length === 1) {
        this.calibrationParams.selectedBedHeater = this.bedHeaters[0];
      }
    },
    //
    // Configuration methods
    //
    addFan() {
      this.calibrationParams.fans.push({ id: null, speed: 0 });
    },
    removeFan(index) {
      this.calibrationParams.fans.splice(index, 1);
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
      return this.getAvailableItems(this.fans, selectedFanIds);
    },
    availableFansFor(fanId) {
      const selectedFanIds = this.availableFans().filter((id) => id != fanId);
      return this.getAvailableItems(this.fans, selectedFanIds);
    },
    getBedHeaterStartLabel() {
      let minTemp = 0;
      let maxTemp = 0;
      const bedHeater = this.selectedBedHeater;
      if (bedHeater !== null) {
        minTemp = Math.max(0, bedHeater.min);
        maxTemp = bedHeater.max - 1;
      }
      if (this.calibrationParams.bedHeaterStop !== null) {
        maxTemp = Math.min(maxTemp, this.calibrationParams.bedHeaterStop) - 1;
      }
      return `[${minTemp}-${maxTemp}]`;
    },
    getBedHeaterStopLabel() {
      let minTemp = 0;
      let maxTemp = 0;
      const bedHeater = this.selectedBedHeater;
      if (bedHeater !== null) {
        minTemp = Math.max(minTemp, bedHeater.min);
        maxTemp = bedHeater.max;
      }
      if (this.calibrationParams.bedHeaterStart !== null) {
        minTemp = Math.max(minTemp, this.calibrationParams.bedHeaterStart) + 1;
      }
      return `[${minTemp}-${maxTemp}]`;
    },
    getStateIcon(completed) {
      if (completed === null) {
        return;
      }
      return completed ? "mdi-check" : "mdi-clock";
    },
    //
    // Machine Operations
    //
    async homeMachine() {
      await this.doCode("G28");
    },
    async setBedHeater(temp) {
      await this.doCode(`M140 S${temp}`);
    },
    async disableBedHeater() {
      await this.setBedHeater(-273.1);
    },
    async setChamberHeater(temp) {
      await this.doCode(`M141 S${temp}`);
    },
    async disableChamberHeater() {
      await this.setChamberHeater(-273.1);
    },
    async enableChamberHeater(bedTemp) {
      if (
        this.showChamberHeatersConfig === false ||
        this.showChamberHeatersConfig === null
      ) {
        return;
      }
      const heaterStart = this.calibrationParams.chamberHeaterStart;
      const heaterStop = this.calibrationParams.chamberHeaterStop;
      const targetChamberTemp = Number.min(
        heaterStop,
        Number.max(heaterStart, bedTemp)
      );
      await this.setChamberHeater(targetChamberTemp);
    },
    async setFan(fanId, speed) {
      await this.doCode(`M106 P${fanId} S${speed}`);
    },
    async disableFan(fanId) {
      await this.setFan(fanId, 0);
    },
    async enableFans() {
      const enableFan =
        this.showFanConfig && this.calibrationParams.fans.length > 0;
      if (!enableFan) {
        return;
      }
      const fans = this.calibrationParams.fans;
      for (const fan of fans) {
        await this.setFan(fan.id, fan.speed);
      }
    },
    async disableFans() {
      if (!this.showFanConfig) {
        return;
      }
      const fans = this.calibrationParams.fans;
      for (const fan of fans) {
        await this.disableFan(fan.id);
      }
    },
    //
    // Pre Calibration
    //
    loadCalibrationPage() {
      this.resetCalibrationResults();
      this.calibrationStarted = false;
      this.calibrationFinished = false;
      this.calibrationCancelled = false;
      this.m558_1Executed = false;
      this.populateCalibrationProgress();
    },
    calculateTemps() {
      const start = this.calibrationParams.bedHeaterStart;
      const stop = this.calibrationParams.bedHeaterStop;
      const step = this.calibrationParams.bedHeaterStep;
      const n = Math.ceil((Number(stop) - Number(start)) / Number(step));
      const temps = Array.from({ length: n }, (_, i) =>
        Math.min(Number(start) + i * Number(step), Number(stop))
      );
      if (Number(stop) !== temps[temps.length - 1]) {
        temps.push(Number(stop));
      }
      return temps;
    },
    resetCalibrationResults() {
      this.calibrationResults = { calibrationValues: [], coefficients: {} };
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
      await this.doM558_1();
      this.recordScanCoefficients();
      await this.setToolToTriggerHeight();
      await this.startMeasurement();
    },
    async awaitBusy() {
      while (this.isBusy) {
        await this.delay(1000);
      }
    },
    async doM558_1() {
      const probeId = this.calibrationParams.selectedScanningProbe.id;
      const m558_1_S_Value = 0.5;
      const m558_1Command = `M558.1 K${probeId} S${m558_1_S_Value}`;
      await this.doCode(m558_1Command);
      await this.awaitBusy();
      this.m558_1Executed = true;
    },
    recordScanCoefficients(probeId) {
      const scanningProbe = this.calibrationParams.selectedScanningProbe;
      const coefficients = {
        A: scanningProbe.scanCoefficients[0],
        B: scanningProbe.scanCoefficients[1],
        C: scanningProbe.scanCoefficients[2],
        triggerOffset: scanningProbe.scanCoefficients[3],
      };
      this.calibrationResults.coefficients = coefficients;
    },
    async setToolToTriggerHeight() {
      const toolId = this.calibrationParams.selectedTool.id;
      const probeTriggerHeight =
        this.calibrationParams.selectedScanningProbe.triggerHeight;
      const g1Command = `G1 P${toolId} Z${probeTriggerHeight}`;
      await this.doCode(g1Command);
      await this.awaitBusy();
    },
    async startMeasurement() {
      const temps = this.calibrationProgress.map((item) => item.targetTemp);
      await this.enableFans();

      for (let i = 0; i < temps.length; i++) {
        if (this.calibrationCancelled) {
          this.cleanUp();
          break;
        }
        const temp = temps[i];
        if (this.getBedHeaterCurrentTemp > temp + 1) {
          this.setCalibrationStatus(i, true);
          continue;
        }
        this.startTime = Date.now();
        this.setCalibrationStatus(i, false);
        console.log("Setting bed heater to", temp);
        await this.setBedHeater(temp);
        console.log("Waiting for stabilization at", temp);
        await this.waitForStabilization(temp, i);
        console.log("Soaking at", temp);
        await this.soakAtTemp(i);
        console.log("soak completed");
        this.setCalibrationStatus(i, true);
      }
      console.log("calibration finishing...");
      await this.finishMeasurement();
    },
    setCalibrationStatus(index, status) {
      this.calibrationProgress[index].status = status;
    },
    async updateProgressLoop(i) {
      await this.delay(1000);
      this.updateCalibrationProgressTable(i);
      await this.recordProbeValue();
    },
    checkStartTemperature() {
      const currentTemp = this.getBedHeaterCurrentTemp;
      const startTemp = Number(this.calibrationParams.bedHeaterStart) + 1;
      return currentTemp > startTemp;
    },
    isTemperatureStable(targetTemp) {
      const currentTemp = this.getBedHeaterCurrentTemp;
      return Math.abs(currentTemp - targetTemp) < 1;
    },
    async waitForStabilization(targetTemp, i) {
      while (!this.isTemperatureStable(targetTemp)) {
        if (this.calibrationCancelled) {
          this.cleanUp();
          break;
        }
        await this.updateProgressLoop(i);
      }
    },
    async soakAtTemp(i, duration = 300000) {
      // Default soak duration is 5 minutes
      const soakStartTime = Date.now();
      await this.delay(1000);
      while (duration > Date.now() - soakStartTime) {
        if (this.calibrationCancelled) {
          this.cleanUp();
          break;
        }
        await this.updateProgressLoop(i);
      }
    },
    updateCalibrationProgressTable(index) {
      this.calibrationProgress[index].currentTemp =
        this.getBedHeaterCurrentTemp;
      this.calibrationProgress[index].timeElapsed = this.formatTime(
        Date.now() - this.startTime
      );
    },
    async cleanUp() {
      await this.disableBedHeater();
      await this.disableChamberHeater();
      await this.disableFans();
    },
    async finishMeasurement() {
      await this.cleanUp();
      this.calibrationFinished = true;
    },
    downloadCalibrationResults() {
      const filename = this.getSaveFilename() + ".json";
      this.downloadJSON(this.calibrationResults, filename);
    },
    getSaveFilename() {
      const time = new Date().toISOString().replace(/:/g, "-");
      const probeId = this.calibrationParams.selectedScanningProbe.id;
      return `scanning-probe-index-${probeId}-calibration-${time}`;
    },
    async recordProbeValue() {
      const currentTargetBedTemp = this.getBedHeaterActiveTemp;

      await this.disableBedHeater();

      const currentBedTemp = this.getBedHeaterCurrentTemp;
      const currentProbeValue =
        this.calibrationParams.selectedScanningProbe.value[0];
      const currentProbeTemp = this.getScanningProbeTemp;
      await this.setBedHeater(currentTargetBedTemp);
      const res = [currentBedTemp, currentProbeTemp, currentProbeValue];
      this.calibrationResults.calibrationValues.push(res);
    },
    delay(ms) {
      return new Promise((resolve) => setTimeout(resolve, ms));
    },
    formatTime(ms) {
      const minutes = Math.floor(ms / (1000 * 60));
      const seconds = Math.floor((ms % (1000 * 60)) / 1000);
      return `${minutes}m ${seconds}s`;
    },
    //
    // Page operations
    //
    cancel() {
      if (this.currentPage === "calibration" && !this.calibrationCancelled) {
        this.calibrationCancelled = true;
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
          this.loadCalibrationPage();
          break;
      }
    },
    //
    // File operations
    //
    downloadJSON(calibrationResults, filename) {
      const jsonString = JSON.stringify(calibrationResults, null, 2);
      const blob = new Blob([jsonString], {
        type: "application/json;charset=utf-8;",
      });

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
        this.initializeCalibrationParams();
      } else {
        if (this.currentPage === "calibration") {
          this.calibrationCancelled = true;
        }
        this.currentPage = "start";
        this.calibrationCancelled = this.calibrationFinished = false;
      }
    },
    "state.status"(to) {
      if (
        (to === MachineStatus.disconnected || to === MachineStatus.off) &&
        this.currentPage === "calibration"
      ) {
        this.calibrationCancelled = true;
      }
    },
    scanningProbes(newProbes) {
      if (newProbes.length === 1) {
        this.calibrationParams.selectedScanningProbe = newProbes[0];
      }
    },
    tools(newTools) {
      if (newTools.length === 1) {
        this.calibrationParams.selectedTool = newTools[0];
      }
    },
    bedHeaters(newHeaters) {
      if (newHeaters.length === 1) {
        this.calibrationParams.selectedBedHeater = newHeaters[0];
      }
    },
  },
};
</script>
