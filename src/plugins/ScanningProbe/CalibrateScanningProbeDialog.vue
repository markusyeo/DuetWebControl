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
                <li v-for="probe in probes" :key="probe.index">
                  <strong>Scanning Probe: Index #{{ probe.index }}</strong>
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
                      v-model="calibrationParams.selectedScanningProbeIndex"
                      :items="
                        probes.map((probe) => ({
                          text: `Probe #${probe.index}`,
                          value: probe.index,
                        }))
                      "
                      item-text="text"
                      item-value="value"
                      label="Select Scanning Probe Index"
                      class="mb-3"
                      hide-details
                      :rules="probeIndexIsRequired"
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
                      <th style="width: 20%">Step (°C)</th>
                      <th style="width: 5%" class="pr-0"></th>
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
                      <td style="width: 5%" class="pr-0">
                        <v-btn
                          color="warning"
                          :disabled="calibrationParams.bedHeater.length <= 1"
                          icon
                          @click="removeHeater(index)"
                        >
                          <v-icon>mdi-close</v-icon>
                        </v-btn>
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
                  !showChamberHeatersConfig && availableHeaters().length === 0
                "
              ></v-checkbox>
              <div v-if="showChamberHeatersConfig">
                <v-simple-table class="mt-1">
                  <thead>
                    <tr>
                      <th class="pl-0">Heater</th>
                      <th style="width: 20%">Start (°C)</th>
                      <th style="width: 20%">Stop (°C)</th>
                      <th style="width: 10%" class="pr-0"></th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr
                      v-for="(
                        heater, index
                      ) in calibrationParams.chamberHeaters"
                      :key="`heater-${index}`"
                    >
                      <td class="px-0">
                        <v-select
                          v-model="heater.id"
                          :items="availableHeatersFor(heater.id)"
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
                      <td style="width: 10%" class="pr-0">
                        <v-btn
                          color="warning"
                          :disabled="calibrationParams.bedHeater.length <= 1"
                          icon
                          @click="removeChamberHeater(index)"
                        >
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
                  @click="addChamberHeater"
                  :disabled="
                    heaters.length >=
                    calibrationParams.bedHeater.length +
                      calibrationParams.chamberHeaters.length
                  "
                >
                  <v-icon class="mr-1">mdi-plus</v-icon>
                  Add Chamber Heater
                </v-btn>
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
    isBusy() {
      return this.state.status === MachineStatus.busy;
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
      return [].concat(
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
          const isThermistorSelected =
            !!this.calibrationParams.selectedThermistor;
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
            isThermistorSelected &&
            heatersValid &&
            chamberHeatersValid &&
            fansValid
          );
      }
      return false;
    },
    toolIsRequired() {
      return [(v) => !!v || "Tool selection is required"];
    },
    probeIndexIsRequired() {
      return [(v) => !!v || "Probe index is required"];
    },
    thermistorIsRequired() {
      return [(v) => !!v || "A thermistor must be selected."];
    },
  },
  data() {
    return {
      currentPage: "start",
      showChamberHeatersConfig: false,
      showFanConfig: false,
      calibrationParams: {
        selectedTool: null,
        selectedThermistor: null,
        selectedScanningProbeIndex: null,
        bedHeater: [{ id: null, start: null, stop: null, step: null }],
        chamberHeaters: [{ id: null, start: null, stop: null }],
        fans: [{ id: null, speed: 0 }],
      },
      run: 0,

      calibrationStarted: false,
      finished: false,
      cancelled: false,

      calibrationValues: [],
      elapsedTime: 0, // in milliseconds

      calibrationTableHeaders: [
        { text: "Index", align: "start", sortable: false, value: "index" },
        { text: "Current Temp", value: "currentTemp" },
        { text: "Target Temp", value: "targetTemp" },
        { text: "Time Elapsed", value: "timeElapsed" },
      ],
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
    homeMachine() {
      this.doCode("G28");
    },
    populateCalibrationProgress() {
      const temps = this.calculateTemps();
      this.calibrationProgress = temps.map((targetTemp, index) => ({
        index,
        currentTemp: null,
        targetTemp,
        timeElapsed: null,
      }));
      console.log(this.calibrationProgress);
    },
    async startCalibration() {
      this.calibrationStarted = true; // Mark calibration as started
      this.startMeasurement(); // Initiate the calibration process
    },
    async startMeasurement() {
      const temps = this.calibrationProgress.map((item) => item.targetTemp);
      // Form the data required in the table
      for (let i = 0; i < temps.length; i++) {
        const targetTemp = temps[i];
        await this.enableHeater(
          this.calibrationParams.bedHeater[0].id,
          targetTemp
        );

        let isStable = false;
        while (!isStable) {
          isStable = this.isTemperatureStable(
            this.calibrationParams.bedHeater[0].id
          );
          if (!isStable) {
            await this.delay(1000); // Check every second
            this.calibrationProgress[i].currentTemp = this.getHeaterTemp(
              this.calibrationParams.bedHeater[0].id
            );
          }
        }

        // Soak for 5 minutes
        const soakTime = 300000; // 5 minutes
        this.elapsedTime = 0;
        while (this.elapsedTime < soakTime) {
          await this.delay(1000);
          this.elapsedTime += 1000;
          this.recordProbeValue();
        }
        // Record the current temperature and other values
        this.calibrationProgress[i].currentTemp = this.getHeaterTemp(
          this.calibrationParams.bedHeater[0].id
        );
        this.calibrationProgress[i].timeElapsed = this.formatTime(
          elapsedTime + soakTime
        );
      }

      this.finished = true;
    },
    delay(ms) {
      return new Promise((resolve) => setTimeout(resolve, ms));
    },
    isTemperatureStable(heaterId) {
      const currentTemp = this.getHeaterTemp(heaterId);
      const targetTemp = this.calibrationParams.bedHeater[0].id;
      return Math.abs(currentTemp - targetTemp) < 1;
    },
    calibrationtemps() {},
    calibrationProgress() {
      const temps = this.calculateTemps();
    },
    getTimeElapsed(index) {
      // Calculate the time elapsed for the current target temperature
      if (index === 0) {
        return this.formatTime(Date.now() - this.startTime);
      } else {
        const prevTimeElapsed = this.calibrationProgress[index - 1].timeElapsed;
        const currentTime = Date.now();
        const prevTime = this.startTime + prevTimeElapsed;
        return this.formatTime(currentTime - prevTime);
      }
    },
    formatTime(ms) {
      const minutes = Math.floor(ms / (1000 * 60));
      const seconds = Math.floor((ms % (1000 * 60)) / 1000);
      return `${minutes}m ${seconds}s`;
    },
    record() {
      this.calibrationValues = [];
      this.finished = false;
      this.cancelled = false;
      this.run++;
    },
    home() {
      this.doCode("G28");
    },
    getScanningProbeTriggerHeight(probeIndex) {
      return this.probes[probeIndex].getScanningProbeTriggerHeight;
    },
    getScanningProbeTemp() {
      return this.thermistors[this.thermistor.index].value;
    },
    getScanningProbeValue(probeIndex) {
      return this.probes[probeIndex].value;
    },
    recordProbeValue(probeIndex) {
      const heaterId = this.calibrationParams.bedHeater[0].id;
      const currentTargetBedTemp = heaters[heaterId].active;

      this.disableBedHeater();

      const currentBedTemp = getHeaterTemp(heaterId);
      const currentProbeValue = getScanningProbeValue(probeIndex);
      const currentProbeTemp = getScanningProbeTemp();

      this.enableHeater(heaterId, currentTargetBedTemp);

      this.calibrationValues.push([
        currentBedTemp,
        currentProbeTemp,
        currentProbeValue,
      ]);
    },
    getHeaterTemp(heaterId) {
      const heater = this.heaters.find((h) => h.id === heaterId);
      return heater ? heater.current : null;
    },
    enableHeater(heaterId, temp) {
      sendCode(`M104 H${heaterId} S${temp}`);
    },
    disableBedHeater() {
      const heaters = this.calibrationParams.bedHeater;
      for (const heater of heaters) {
        sendCode(`M104 H${heater.id} S0`);
      }
    },
    enableFan(fanId, speed) {
      sendCode(`M106 P${fanId} S${speed}`);
    },
    disableFan(fanId) {
      this.enableFan(fanId, 0);
    },
    checkHeaterTemp(heaterId) {
      this.heaters.forEach((heater) => {
        if (heater.id === heaterId) {
          return heater.current;
        }
      });
    },
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
    // This is for deciding the values to be used fo the probe
    // Bassed on the recorded values
    plotProbeValues() {},
    calculateTemperatureCoefficients() {
      // Calculate the temperature coefficients using the recorded values
    },
    saveProbeCalibration() {
      // Save the calculated temperature coefficients to the scanning probe
    },
    enableFans() {
      const fans = this.calibrationParams.fans;
      for (const fan of fans) {
        sendCode(`M106 P${fan.id} S${fan.speed}`);
      }
    },
    disableFans() {
      const fans = this.calibrationParams.fans;
      for (const fan of fans) {
        sendCode(`M106 P${fan.id} S0`);
      }
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
    availableHeatersFor(heaterId) {
      const selectedHeaters = this.calibrationParams.bedHeater
        .map((heater) => heater.id)
        .filter((id) => id != heaterId);
      const selectedChamberHeaters = this.calibrationParams.chamberHeaters
        .map((heater) => heater.id)
        .filter((id) => id != heaterId);
      const selectedHeaterIds = selectedHeaters.concat(selectedChamberHeaters);
      const remainingHeaters = this.heat.heaters
        .map((heater, index) => ({
          ...heater,
          name: `Heater ${index}`,
          id: index,
        }))
        .filter((heater) => !selectedHeaterIds.includes(heater.id));
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
          this.record();
          this.populateCalibrationProgress();
          break;
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
    removeChamberHeater(index) {
      this.calibrationParams.chamberHeaters.splice(index, 1);
    },
    removeFan(index) {
      this.calibrationParams.fans.splice(index, 1);
    },
  },
  created() {
    if (this.probes.length === 1) {
      this.calibrationParams.selectedScanningProbeIndex = this.probes[0].index;
    }
    if (this.thermistors.length === 1) {
      this.calibrationParams.selectedThermistor = this.thermistors[0].id;
    }
    if (this.toolList.length === 1) {
      this.calibrationParams.selectedTool = this.toolList[0].value;
    }
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
        this.calibrationParams.selectedScanningProbeIndex = newProbes[0].index;
      }
    },
    tools(newTools) {
      if (newTools.length === 1) {
        this.calibrationParams.selectedTool = newTools[0];
      }
    },
  },
};
</script>
