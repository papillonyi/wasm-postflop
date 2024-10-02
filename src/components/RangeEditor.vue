<template>
  <div class="flex mt-1">
    <div class="shrink-0 ml-1">
      <table class="shadow-md select-none snug" @mouseleave="dragEnd">
        <tr v-for="row in 13" :key="row" class="h-9">
          <td
            v-for="col in 13"
            :key="col"
            class="relative w-[2.625rem] border border-black"
            @mousedown="dragStart(row, col)"
            @mouseenter="mouseEnter(row, col)"
            @mouseup="dragEnd"
          >
            <div
              :class="
                'absolute w-full h-full left-0 top-0 ' +
                (row === col ? 'bg-neutral-700' : 'bg-neutral-800')
              "
            >
              <div
                :style="{
                  'background-image': `linear-gradient(${yellow500} 0% 100%)`,
                  'background-size': `100% ${cellValue(row, col)}%`,
                }"
                class="absolute w-full h-full left-0 top-0 bg-bottom bg-no-repeat"
              ></div>
            </div>
            <div
              :class="
                'absolute -top-px left-[0.1875rem] z-10 text-shadow ' +
                (cellValue(row, col) > 0 ? 'text-white' : 'text-neutral-500')
              "
            >
              {{ cellText(row, col) }}
            </div>
            <div
              class="absolute bottom-px right-1 z-10 text-sm text-shadow text-white"
            >
              {{
                cellValue(row, col) > 0 && cellValue(row, col) < 100
                  ? cellValue(row, col).toFixed(1)
                  : ""
              }}
            </div>
          </td>
        </tr>
      </table>

      <div class="mt-5">
        <div class="flex">
          <input
            v-model="rangeText"
            :class="
              'flex-grow mr-6 px-2 py-1 rounded-lg text-sm ' +
              (rangeTextError ? 'input-error' : '')
            "
            type="text"
            @change="onRangeTextChange"
            @focus="($event.target as HTMLInputElement).select()"
          />

          <button class="button-base button-blue" @click="clearRange">
            Clear
          </button>
        </div>

        <div v-if="rangeTextError" class="mt-1 text-red-500">
          Error: {{ rangeTextError }}
        </div>
      </div>

      <div class="flex mt-3.5 items-center">
        <div>
          Weight:
          <input
            v-model="weight"
            class="ml-3 w-40 align-middle"
            max="100"
            min="0"
            step="5"
            type="range"
            @change="onWeightChange"
          />
          <input
            v-model="weight"
            :class="
              'w-20 ml-4 px-2 py-1 rounded-lg text-sm text-center ' +
              (weight < 0 || weight > 100 ? 'input-error' : '')
            "
            max="100"
            min="0"
            step="5"
            type="number"
            @change="onWeightChange"
          />
          %
        </div>

        <span class="inline-block ml-auto">
          {{ numCombos.toFixed(1) }} combos ({{
            ((numCombos * 100) / ((52 * 51) / 2)).toFixed(1)
          }}%)
        </span>
      </div>
    </div>

    <div class="flex-grow max-w-[18rem] ml-6">
      <DbItemPicker
        :allow-save="rangeText !== '' && rangeTextError === ''"
        :index="player"
        :value="rangeText"
        store-name="ranges"
        @load-item="loadRange"
      />
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from "vue";
import { useConfigStore } from "../store";
import { rankPat, ranks } from "../utils";
import { RangeManager } from "../../pkg/range/range";

import DbItemPicker from "./DbItemPicker.vue";

const yellow500 = "#eab308";

const comboPat = `(?:(?:${rankPat}{2}[os]?)|(?:(?:${rankPat}[cdhs]){2}))`;
const weightPat = "(?:(?:[01](\\.\\d*)?)|(?:\\.\\d+))";
const trimRegex = /\s*([-:,])\s*/g;
const rangeRegex = new RegExp(
  `^(?<range>${comboPat}(?:\\+|(?:-${comboPat}))?)(?::(?<weight>${weightPat}))?$`
);

type DraggingMode = "none" | "enabling" | "disabling";

export default defineComponent({
  components: {
    DbItemPicker,
  },

  props: {
    player: {
      type: Number,
      required: true,
    },
  },

  setup(props) {
    const config = useConfigStore();

    const range = RangeManager.new();
    const rangeStore = config.range[props.player];
    const rangeStoreRaw = config.rangeRaw[props.player];
    const rangeText = ref("");
    const rangeTextError = ref("");
    const weight = ref(100);
    const numCombos = ref(0);

    let draggingMode: DraggingMode = "none";

    const cellText = (row: number, col: number) => {
      const r1 = 13 - Math.min(row, col);
      const r2 = 13 - Math.max(row, col);
      return ranks[r1] + ranks[r2] + ["s", "", "o"][Math.sign(row - col) + 1];
    };

    const cellIndex = (row: number, col: number) => {
      return 13 * (row - 1) + col - 1;
    };

    const cellValue = (row: number, col: number) => {
      return rangeStore[cellIndex(row, col)];
    };

    const onUpdate = () => {
      rangeStoreRaw.set(range.raw_data());
      rangeText.value = range.to_string();
      rangeTextError.value = "";
      numCombos.value = rangeStoreRaw.reduce((acc, cur) => acc + cur, 0);
    };

    const update = (row: number, col: number, weight: number) => {
      const idx = 13 * (row - 1) + col - 1;
      range.update(row, col, weight / 100);
      rangeStore[idx] = weight;
      onUpdate();
    };

    const onRangeTextChange = () => {
      const trimmed = rangeText.value.replace(trimRegex, "$1").trim();
      const ranges = trimmed.split(",");

      if (ranges[ranges.length - 1] === "") {
        ranges.pop();
      }

      for (const range of ranges) {
        if (!rangeRegex.test(range)) {
          rangeTextError.value = `Failed to parse range: ${
            range || "(empty string)"
          }`;
          return;
        }
      }

      const errorString = range.from_string(trimmed);

      if (errorString) {
        rangeTextError.value = errorString;
      } else {
        const weights = range.get_weights();
        for (let i = 0; i < 13 * 13; ++i) {
          rangeStore[i] = weights[i] * 100;
        }
        onUpdate();
      }
    };

    const dragStart = (row: number, col: number) => {
      const idx = 13 * (row - 1) + col - 1;

      if (rangeStore[idx] !== weight.value) {
        draggingMode = "enabling";
        update(row, col, weight.value);
      } else {
        draggingMode = "disabling";
        update(row, col, 0);
      }
    };

    const dragEnd = () => {
      draggingMode = "none";
    };

    const mouseEnter = (row: number, col: number) => {
      if (draggingMode === "enabling") {
        update(row, col, weight.value);
      } else if (draggingMode === "disabling") {
        update(row, col, 0);
      }
    };

    const onWeightChange = () => {
      weight.value = Math.round(Math.max(0, Math.min(100, weight.value)));
    };

    const clearRange = () => {
      range.clear();
      rangeStore.fill(0);
      rangeStoreRaw.fill(0);
      rangeText.value = "";
      rangeTextError.value = "";
      weight.value = 100;
      numCombos.value = 0;
    };

    const loadRange = (rangeStr: unknown) => {
      rangeText.value = String(rangeStr);
      onRangeTextChange();
    };

    return {
      yellow500,
      cellText,
      cellValue,
      rangeStore,
      rangeText,
      rangeTextError,
      weight,
      numCombos,
      onRangeTextChange,
      dragStart,
      dragEnd,
      mouseEnter,
      onWeightChange,
      clearRange,
      loadRange,
    };
  },
});
</script>
