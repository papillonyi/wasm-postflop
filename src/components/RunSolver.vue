<template>
  <div class="flex mt-1">
    <div class="shrink-0 ml-1">
      <div class="flex my-1 items-center">
        Number of threads:
        <input
          v-model="numThreads"
          :class="
            'w-20 ml-2 px-2 py-1 rounded-lg text-sm text-center ' +
            (numThreads < 1 ||
            numThreads > (isSafari ? 1 : 64) ||
            numThreads % 1 !== 0
              ? 'input-error'
              : '')
          "
          max="64"
          min="1"
          type="number"
        />
        <button
          :disabled="
            isTreeBuilding ||
            store.isSolverRunning ||
            store.isFinalizing ||
            numThreads < 1 ||
            numThreads > (isSafari ? 1 : 64) ||
            numThreads % 1 !== 0
          "
          class="ml-3 button-base button-blue"
          @click="buildTree"
        >
          Build New Treea
        </button>
        <button class="ml-3 button-base button-blue">Load</button>
      </div>

      <div class="my-1">Status: {{ treeStatus }}</div>

      <div v-if="isTreeBuilt" class="mt-3">
        <div>
          Precision mode:
          <Tippy
            :delay="[200, 0]"
            :interactive="true"
            class="inline-block cursor-help"
            max-width="500px"
            placement="bottom"
            trigger="mouseenter click"
          >
            <QuestionMarkCircleIcon class="w-5 h-5 text-gray-600" />
            <template #content>
              <div class="px-1 py-0.5 text-justify">
                Precision mode mainly affects memory usage, but it also has
                several other effects.
                <ul class="pl-6 list-disc">
                  <li class="mt-1">
                    32-bit FP (floating-point): This is recommended if the
                    memory usage is below the limit (= 3.9GB). It has about 7
                    significant digits and better performance.
                  </li>
                  <li class="mt-1">
                    16-bit integer: This setting can help to avoid the memory
                    limit if the 32-bit FP mode is not usable. Since the
                    significant digits are about 4 digits, it is not suitable
                    for satisfying an exploitability target below 0.1%.
                    Performance is also worse than in 32-bit FP mode.
                  </li>
                </ul>
              </div>
            </template>
          </Tippy>
        </div>
        <div class="mt-1 ml-2">
          <label :class="{ 'cursor-pointer': !store.hasSolverRun }">
            <input
              v-model="isCompressionEnabled"
              :disabled="store.hasSolverRun"
              :value="false"
              class="mr-2 cursor-pointer disabled:cursor-default"
              name="compression"
              type="radio"
            />
            <span class="inline-block w-[6.75rem] ml-1">32-bit FP:</span>
            needs
            {{
              memoryUsage >= 1023.5 * 1024 * 1024
                ? (memoryUsage / (1024 * 1024 * 1024)).toFixed(2) + "GB"
                : (memoryUsage / (1024 * 1024)).toFixed(0) + "MB"
            }}
            RAM
            {{ memoryUsage > maxMemoryUsage ? "(limit exceeded)" : "" }}
          </label>
        </div>
        <div class="ml-2">
          <label :class="{ 'cursor-pointer': !store.hasSolverRun }">
            <input
              v-model="isCompressionEnabled"
              :disabled="store.hasSolverRun"
              :value="true"
              class="mr-2 cursor-pointer disabled:cursor-default"
              name="compression"
              type="radio"
            />
            <span class="inline-block w-[6.75rem] ml-1">16-bit integer:</span>
            needs
            {{
              memoryUsageCompressed >= 1023.5 * 1024 * 1024
                ? (memoryUsageCompressed / (1024 * 1024 * 1024)).toFixed(2) +
                  "GB"
                : (memoryUsageCompressed / (1024 * 1024)).toFixed(0) + "MB"
            }}
            RAM
            {{
              memoryUsageCompressed > maxMemoryUsage ? "(limit exceeded)" : ""
            }}
          </label>
        </div>
        <div v-if="memoryUsage > maxMemoryUsage" class="mt-1.5">
          RAM limit: 3.9GB (= 4GB Wasm limit - 0.1GB margin)
        </div>

        <div class="mt-4">
          Target exploitability:
          <Tippy
            :delay="[200, 0]"
            :interactive="true"
            class="inline-block cursor-help"
            max-width="500px"
            placement="bottom"
            trigger="mouseenter click"
          >
            <QuestionMarkCircleIcon class="w-5 h-5 text-gray-600" />
            <template #content>
              <div class="px-1 py-0.5 text-justify">
                <div>
                  Specifies the acceptable distance to the Nash equilibrium. A
                  lower value gives more accurate results, but also requires
                  more computation time.
                </div>
                <div class="mt-3">
                  <span class="underline">A more detailed description:</span>
                  When a Nash equilibrium solution is obtained, the strategies
                  of both players become MESs (Maximally Exploitative
                  Strategies) to each other. Using this property, we define the
                  distance to the Nash equilibrium of the obtained strategy as
                  follows:
                </div>
                <div class="my-1 text-center">
                  Distance = (Opponent's MES EV) - (Opponent's obtained EV).
                </div>
                <div>
                  This distance is always non-negative and is zero if and only
                  if the obtained strategy is a part of a particular Nash
                  equilibrium. Exploitability is defined as the average distance
                  of both players.
                </div>
              </div>
            </template>
          </Tippy>
          <input
            v-model="targetExploitability"
            :class="
              'w-20 ml-3 px-2 py-1 rounded-lg text-sm text-center ' +
              (targetExploitability <= 0 ? 'input-error' : '')
            "
            :disabled="store.hasSolverRun && !store.isSolverPaused"
            min="0"
            step="0.05"
            type="number"
          />
          %
        </div>

        <div class="mt-1">
          Maximum number of iterations:
          <input
            v-model="maxIterations"
            :class="
              'w-[5.5rem] ml-2 px-2 py-1 rounded-lg text-sm text-center ' +
              (maxIterations < 0 ||
              maxIterations % 1 !== 0 ||
              maxIterations > 100000
                ? 'input-error'
                : '')
            "
            :disabled="store.hasSolverRun && !store.isSolverPaused"
            max="100000"
            min="0"
            type="number"
          />
        </div>

        <div class="flex mt-6 gap-3">
          <button
            :disabled="
              store.hasSolverRun ||
              memoryUsageSelected > maxMemoryUsage ||
              targetExploitability <= 0 ||
              maxIterations < 0 ||
              maxIterations % 1 !== 0 ||
              maxIterations > 100000
            "
            class="button-base button-blue"
            @click="runSolver"
          >
            Run Solver
          </button>
          <button
            :disabled="!store.isSolverRunning"
            class="button-base button-red"
            @click="() => (terminateFlag = true)"
          >
            Stop
          </button>
          <button
            v-if="!store.isSolverPaused"
            :disabled="!store.isSolverRunning"
            class="button-base button-green"
            @click="() => (pauseFlag = true)"
          >
            Pause
          </button>
          <button
            v-else
            :disabled="
              targetExploitability <= 0 ||
              maxIterations < 0 ||
              maxIterations % 1 !== 0 ||
              maxIterations > 100000
            "
            class="button-base button-green"
            @click="resumeSolver"
          >
            Resume
          </button>
          <button
            :disabled="!store.isSolverFinished"
            class="button-base button-green"
            @click="downloadGame"
          >
            Download
          </button>
          <button
            :disabled="!store.isSolverFinished"
            class="button-base button-green"
            @click="loadGame"
          >
            Load
          </button>
          <div>
            <input type="file" @change="loadGameFromFile" />
          </div>
        </div>

        <div v-if="store.hasSolverRun" class="mt-6">
          <div class="flex items-center">
            <span
              v-if="store.isSolverRunning || store.isFinalizing"
              class="spinner inline-block mr-3"
            ></span>
            {{
              store.isSolverRunning
                ? "Solver running..."
                : store.isFinalizing
                ? "Finalizing..."
                : store.isSolverPaused
                ? "Solver paused."
                : "Solver finished."
            }}
          </div>
          {{ iterationText }}
          <br />
          {{ exploitabilityText }}
          <br />
          {{ timeText }}
        </div>
      </div>
    </div>
    <!--    <div class="flex-grow max-w-[18rem] ml-6">-->
    <!--      <DbItemPicker-->
    <!--        :allow-save="gameUintArray.length !== 0"-->
    <!--        :value="gameUintArray"-->
    <!--        store-name="solvers"-->
    <!--        @load-item="loadGame"-->
    <!--      />-->
    <!--    </div>-->
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, ref } from "vue";
import { handler, init } from "../global-worker";
import {
  saveConfig,
  saveConfigTmp,
  useConfigStore,
  useSavedConfigStore,
  useStore,
  useTmpConfigStore,
} from "../store";
import {
  convertBetString,
  INVALID_LINE_STRING,
  MAX_AMOUNT,
  readableLineString,
  ROOT_LINE_STRING,
} from "../utils";
import { detect } from "detect-browser";

import { Tippy } from "vue-tippy";
import { QuestionMarkCircleIcon } from "@heroicons/vue/20/solid";

const maxMemoryUsage = 3.9 * 1024 * 1024 * 1024; // 3.9 GB
const browser = detect();
const isSafari = browser && (browser.name === "safari" || browser.os === "iOS");

const gameUintArray = ref(new Uint8Array([]));
const fileContent = ref(new Uint8Array([]));

const checkConfig = (
  config: ReturnType<typeof useConfigStore>
): string | null => {
  if (config.board.length < 3) {
    return "Board must consist of at least three cards";
  }

  if (config.startingPot <= 0) {
    return "Starting pot must be positive";
  }

  if (config.startingPot > MAX_AMOUNT) {
    return "Starting pot is too large";
  }

  if (config.startingPot % 1 !== 0) {
    return "Starting pot must be an integer";
  }

  if (config.effectiveStack <= 0) {
    return "Effective stack must be positive";
  }

  if (config.effectiveStack > MAX_AMOUNT) {
    return "Effective stack is too large";
  }

  if (config.effectiveStack % 1 !== 0) {
    return "Effective stack is be an integer";
  }

  const betConfig = [
    { s: config.oopFlopBetSanitized, kind: "OOP flop bet" },
    { s: config.oopFlopRaiseSanitized, kind: "OOP flop raise" },
    { s: config.oopTurnBetSanitized, kind: "OOP turn bet" },
    { s: config.oopTurnRaiseSanitized, kind: "OOP turn raise" },
    { s: config.oopRiverBetSanitized, kind: "OOP river bet" },
    { s: config.oopRiverRaiseSanitized, kind: "OOP river raise" },
    { s: config.ipFlopBetSanitized, kind: "IP flop bet" },
    { s: config.ipFlopRaiseSanitized, kind: "IP flop raise" },
    { s: config.ipTurnBetSanitized, kind: "IP turn bet" },
    { s: config.ipTurnRaiseSanitized, kind: "IP turn raise" },
    { s: config.ipRiverBetSanitized, kind: "IP river bet" },
    { s: config.ipRiverRaiseSanitized, kind: "IP river raise" },
  ];

  for (const { s, kind } of betConfig) {
    if (!s.valid) {
      return `${kind}: ${s.s}`;
    }
  }

  if (config.donkOption) {
    if (!config.oopTurnDonkSanitized.valid) {
      return `OOP turn donk: ${config.oopTurnDonkSanitized.s}`;
    }
    if (!config.oopRiverDonkSanitized.valid) {
      return `OOP river donk: ${config.oopRiverDonkSanitized.s}`;
    }
  }

  if (config.addAllInThreshold < 0) {
    return "Invalid add all-in threshold";
  }

  if (config.forceAllInThreshold < 0) {
    return "Invalid force all-in threshold";
  }

  if (config.mergingThreshold < 0) {
    return "Invalid merging threshold";
  }

  if (
    config.expectedBoardLength > 0 &&
    config.board.length !== config.expectedBoardLength
  ) {
    return `Invalid board (expected ${config.expectedBoardLength} cards)`;
  }

  const addedLinesArray =
    config.addedLines === ""
      ? []
      : config.addedLines.split(",").map(readableLineString);

  const removedLinesArray =
    config.removedLines === ""
      ? []
      : config.removedLines.split(",").map(readableLineString);

  if (
    addedLinesArray.includes(ROOT_LINE_STRING) ||
    addedLinesArray.includes(INVALID_LINE_STRING) ||
    removedLinesArray.includes(ROOT_LINE_STRING) ||
    removedLinesArray.includes(INVALID_LINE_STRING)
  ) {
    return "Invalid line found (loaded broken configurations?)";
  }

  if (
    ![0, 3, 4, 5].includes(config.expectedBoardLength) ||
    (config.expectedBoardLength === 0 &&
      (addedLinesArray.length > 0 || removedLinesArray.length > 0)) ||
    (config.expectedBoardLength > 0 &&
      addedLinesArray.length === 0 &&
      removedLinesArray.length === 0)
  ) {
    return "Invalid configurations (loaded broken configurations?)";
  }

  return null;
};

export default defineComponent({
  components: {
    Tippy,
    QuestionMarkCircleIcon,
  },

  setup() {
    const store = useStore();
    const config = useConfigStore();
    const savedConfig = useSavedConfigStore();
    const tmpConfig = useTmpConfigStore();

    const numThreads = ref((!isSafari && navigator.hardwareConcurrency) || 1);
    const targetExploitability = ref(0.3);
    const maxIterations = ref(1000);

    const isTreeBuilding = ref(false);
    const isTreeBuilt = ref(false);
    const treeStatus = ref("Module not loaded");
    const memoryUsage = ref(0);
    const memoryUsageCompressed = ref(0);
    const isCompressionEnabled = ref(false);
    const terminateFlag = ref(false);
    const pauseFlag = ref(false);
    const currentIteration = ref(-1);
    const exploitability = ref(Number.POSITIVE_INFINITY);
    const elapsedTimeMs = ref(-1);

    let startTime = 0;
    let exploitabilityUpdated = false;

    const memoryUsageSelected = computed(() => {
      if (isCompressionEnabled.value) {
        return memoryUsageCompressed.value;
      } else {
        return memoryUsage.value;
      }
    });

    const iterationText = computed(() => {
      if (currentIteration.value === -1) {
        return "Allocating memory...";
      } else {
        return `Iteration: ${currentIteration.value}`;
      }
    });

    const exploitabilityText = computed(() => {
      if (!Number.isFinite(exploitability.value)) {
        return "";
      } else {
        const valueText = exploitability.value.toFixed(2);
        const percent = (exploitability.value * 100) / config.startingPot;
        const percentText = `${percent.toFixed(2)}%`;
        return `Exploitability: ${valueText} (${percentText})`;
      }
    });

    const timeText = computed(() => {
      if (elapsedTimeMs.value === -1 || !store.isSolverFinished) {
        return "";
      } else {
        return `Time: ${(elapsedTimeMs.value / 1000).toFixed(2)}s`;
      }
    });

    const buildTree = async () => {
      isTreeBuilt.value = false;

      const configError = checkConfig(config);
      if (configError !== null) {
        treeStatus.value = `Error: ${configError}`;
        return;
      }

      saveConfigTmp();
      isTreeBuilding.value = true;
      store.isSolverPaused = false;
      store.isSolverFinished = false;
      treeStatus.value = "Building tree...";

      await init(numThreads.value);
      if (!handler) return;

      const errorString = await handler.init(
        tmpConfig.rangeRaw[0],
        tmpConfig.rangeRaw[1],
        new Uint8Array(tmpConfig.board),
        tmpConfig.startingPot,
        tmpConfig.effectiveStack,
        tmpConfig.rakePercent / 100,
        tmpConfig.rakeCap,
        tmpConfig.donkOption,
        convertBetString(tmpConfig.oopFlopBet),
        convertBetString(tmpConfig.oopFlopRaise),
        convertBetString(tmpConfig.oopTurnBet),
        convertBetString(tmpConfig.oopTurnRaise),
        tmpConfig.donkOption ? convertBetString(tmpConfig.oopTurnDonk) : "",
        convertBetString(tmpConfig.oopRiverBet),
        convertBetString(tmpConfig.oopRiverRaise),
        tmpConfig.donkOption ? convertBetString(tmpConfig.oopRiverDonk) : "",
        convertBetString(tmpConfig.ipFlopBet),
        convertBetString(tmpConfig.ipFlopRaise),
        convertBetString(tmpConfig.ipTurnBet),
        convertBetString(tmpConfig.ipTurnRaise),
        convertBetString(tmpConfig.ipRiverBet),
        convertBetString(tmpConfig.ipRiverRaise),
        tmpConfig.addAllInThreshold / 100,
        tmpConfig.forceAllInThreshold / 100,
        tmpConfig.mergingThreshold / 100,
        tmpConfig.addedLines,
        tmpConfig.removedLines
      );

      if (errorString) {
        isTreeBuilding.value = false;
        treeStatus.value = "Error: " + errorString;
        return;
      }

      saveConfig();

      memoryUsage.value = await handler.memoryUsage(false);
      memoryUsageCompressed.value = await handler.memoryUsage(true);

      if (
        memoryUsage.value > maxMemoryUsage &&
        memoryUsageCompressed.value <= maxMemoryUsage
      ) {
        isCompressionEnabled.value = true;
      }

      const threadText = `${numThreads.value} thread${
        numThreads.value === 1 ? "" : "s"
      }`;

      isTreeBuilding.value = false;
      isTreeBuilt.value = true;
      treeStatus.value = `Successfully built tree (${threadText})`;
    };

    const runSolver = async () => {
      if (!handler) return;

      terminateFlag.value = false;
      pauseFlag.value = false;
      currentIteration.value = -1;
      exploitability.value = Number.POSITIVE_INFINITY;
      elapsedTimeMs.value = -1;

      store.isSolverRunning = true;

      startTime = performance.now();

      await handler.allocateMemory(isCompressionEnabled.value);

      currentIteration.value = 0;
      exploitability.value = Math.max(await handler.exploitability(), 0);
      exploitabilityUpdated = true;

      await resumeSolver();
      await saveGameToBin();
    };

    const resumeSolver = async () => {
      if (!handler) return;

      store.isSolverRunning = true;
      store.isSolverPaused = false;

      if (startTime === 0) {
        startTime = performance.now();
      }

      const target = (config.startingPot * targetExploitability.value) / 100;

      while (
        !terminateFlag.value &&
        currentIteration.value < maxIterations.value &&
        exploitability.value > target
      ) {
        if (pauseFlag.value) {
          const end = performance.now();
          elapsedTimeMs.value += end - startTime;
          startTime = 0;
          pauseFlag.value = false;
          store.isSolverRunning = false;
          store.isSolverPaused = true;
          return;
        }

        await handler.iterate(currentIteration.value);
        ++currentIteration.value;
        exploitabilityUpdated = false;

        if (currentIteration.value % 10 === 0) {
          exploitability.value = Math.max(await handler.exploitability(), 0);
          exploitabilityUpdated = true;
        }
      }

      if (!exploitabilityUpdated) {
        exploitability.value = Math.max(await handler.exploitability(), 0);
      }

      store.isSolverRunning = false;
      store.isFinalizing = true;

      await handler.finalize();

      store.isFinalizing = false;
      store.isSolverFinished = true;

      const end = performance.now();
      elapsedTimeMs.value += end - startTime;
    };

    const saveGameToBin = async () => {
      if (!handler) return;
      const value = await handler.saveGameToBin();
      // console.log(value);
      gameUintArray.value = value;
    };

    const downloadGame = async () => {
      const blob = new Blob([gameUintArray.value], {
        type: "application/octet-stream",
      });

      // Create a URL for the blob
      const url = URL.createObjectURL(blob);

      // Create a link element
      const link = document.createElement("a");
      link.href = url;
      link.download = "filename.bin"; // Set your desired file name

      // Append the link to the body (required for Firefox)
      document.body.appendChild(link);

      // Trigger the download
      link.click();

      // Clean up
      document.body.removeChild(link);
      URL.revokeObjectURL(url);
    };

    const loadGameFromFile = (event: Event) => {
      const target = event.target as HTMLInputElement;
      const files = target.files;
      console.log(files);
      console.log(target);

      if (files && files[0]) {
        const file = files[0];
        const reader = new FileReader();
        console.log(file);

        reader.onload = (e: ProgressEvent<FileReader>) => {
          if (e.target && e.target.result) {
            const arrayBuffer = e.target.result as ArrayBuffer;
            fileContent.value = new Uint8Array(arrayBuffer);
            console.log(
              "File content stored in Uint8Array:",
              fileContent.value
            );
          }
        };

        reader.readAsArrayBuffer(file);
      }
    };

    const loadGame = async () => {
      if (!handler) return;
      store.isFinalizing = false;
      store.isSolverRunning = false;
      store.isSolverPaused = false;
      store.isSolverFinished = false;
      await handler.loadGameFromBin(fileContent.value);
      // await handler.finalize();

      const IpRange = await handler.loadOopRange();
      const OopRange = await handler.loadOopRange();
      const gameBoard = await handler.loadGameBoard();
      const newBoard = Array.from(gameBoard);
      savedConfig.board = newBoard;
      const startingPot = await handler.loadStartingPot();
      const effectiveStack = await handler.loadEffectiveStack();
      const addAllinThreshold = await handler.loadAddAllinThreshold();
      const forceAllinThreshold = await handler.loadForceAllinThreshold();
      const mergingThreshold = await handler.loadMergingThreshold();

      savedConfig.startingPot = startingPot;
      savedConfig.effectiveStack = effectiveStack;

      store.isFinalizing = false;
      store.isSolverRunning = false;
      store.isSolverPaused = false;
      store.isSolverFinished = true;
    };

    return {
      store,
      numThreads,
      isSafari,
      targetExploitability,
      maxIterations,
      isTreeBuilding,
      isTreeBuilt,
      treeStatus,
      maxMemoryUsage,
      memoryUsage,
      memoryUsageCompressed,
      isCompressionEnabled,
      terminateFlag,
      pauseFlag,
      memoryUsageSelected,
      iterationText,
      exploitabilityText,
      gameUintArray,
      timeText,
      buildTree,
      runSolver,
      resumeSolver,
      saveGameToBin,
      loadGame,
      downloadGame,
      loadGameFromFile,
      fileContent,
    };
  },
});
</script>
