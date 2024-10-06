<template>
  <div v-if="!store.isSolverFinished">
    <div class="flex w-full max-w-screen-xl mx-auto px-4 py-6 items-center">
      <span
        v-if="store.isSolverRunning || store.isFinalizing"
        class="spinner inline-block mr-3"
      ></span>
      {{
        !store.hasSolverRun
          ? "Solver has not run."
          : store.isSolverRunning
          ? "Solver running..."
          : store.isFinalizing
          ? "Finalizing..."
          : "Solver paused."
      }}
    </div>
  </div>

  <div v-else class="flex flex-col h-full">
    <GameNav
      :cards="cards"
      :dealt-card="dealtCard"
      :is-handler-updated="isHandlerUpdated"
      :is-locked="isLocked"
      :player-position="playerPosition"
      @update:is-handler-updated="(value) => (isHandlerUpdated = value)"
      @update:is-locked="(value) => (isLocked = value)"
      @trigger-update="onUpdateSpot"
    />
    <ResultMiddle
      :auto-player-basics="autoPlayerBasics"
      :auto-player-chance="autoPlayerChance"
      :chance-mode="chanceMode"
      :copy-success="copySuccess"
      :display-mode="displayMode"
      @update:display-mode="updateDisplayMode"
      @update:display-options="updateDisplayOptions"
      @copy-to-clipboard="copyRangeTextToClipboard"
      @reset-copy-success="resetCopySuccess"
    />

    <div
      v-if="store.navView === 'game' && selectedSpot && results"
      class="flex flex-grow min-h-0"
    >
      <template v-if="displayMode === 'basics'">
        <GameTable
          :cards="cards"
          :display-player="displayPlayerBasics"
          :hover-content="basicsHoverContent"
          :results="results"
          :selected-spot="selectedSpot"
          style="flex: 3"
          table-mode="basics"
        />
      </template>

      <template v-else-if="displayMode === 'graphs'">
        <ResultGraphs
          :cards="cards"
          :chance-reports="chanceReports"
          :display-options="displayOptions"
          :display-player="displayPlayerBasics"
          :results="results"
          :selected-chance="selectedChance"
          :selected-spot="selectedSpot"
        />
      </template>

      <template v-else-if="displayMode === 'compare'">
        <ResultBasics
          :cards="cards"
          :current-board="currentBoard"
          :display-options="displayOptions"
          :is-compare-mode="true"
          :results="results"
          :selected-chance="selectedChance"
          :selected-spot="selectedSpot"
          :total-bet-amount="totalBetAmount"
          display-player="oop"
          style="flex: 5"
        />

        <ResultCompare
          :results="results"
          :selected-chance="selectedChance"
          :selected-spot="selectedSpot"
          style="flex: 2"
        />

        <ResultBasics
          :cards="cards"
          :current-board="currentBoard"
          :display-options="displayOptions"
          :is-compare-mode="true"
          :results="results"
          :selected-chance="selectedChance"
          :selected-spot="selectedSpot"
          :total-bet-amount="totalBetAmount"
          display-player="ip"
          style="flex: 5"
        />
      </template>

      <template v-else-if="displayMode === 'chance' && selectedChance">
        <ResultChance
          :chance-reports="chanceReports"
          :display-options="displayOptions"
          :display-player="displayPlayerChance"
          :selected-chance="selectedChance"
          :selected-spot="selectedSpot"
          @deal-card="onDealCard"
        />
      </template>
    </div>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, ref } from "vue";
import { useGameStore, useStore } from "../store";
import { handler } from "../global-worker";

import {
  ChanceReports,
  DisplayMode,
  DisplayOptions,
  HoverContent,
  PlayInfo,
  Results,
  Spot,
  SpotChance,
  SpotPlayer,
} from "../result-types";
import ResultMiddle from "./ResultMiddle.vue";
import ResultBasics from "./ResultBasics.vue";
import ResultCompare from "./ResultCompare.vue";
import ResultGraphs from "./ResultGraphs.vue";
import ResultChance from "./ResultChance.vue";
import GameNav from "./GameNav.vue";
import { cardText, getRandomItemByWeight } from "../utils";
import GameTable from "./GameTable.vue";

export default defineComponent({
  components: {
    GameTable,
    GameNav,
    ResultMiddle,
    ResultBasics,
    ResultCompare,
    ResultGraphs,
    ResultChance,
  },

  setup() {
    const store = useStore();

    /* Navigation */

    const isHandlerUpdated = ref(false);
    const isLocked = ref(false);
    // const playersInfo = ref<PlayInfo[]>([]);
    // const playerPositionInt = ref(0);
    const gameStore = useGameStore();

    const cards = ref<number[][]>([[], []]);
    const dealtCard = ref(-1);

    const selectedSpot = ref<Spot | null>(null);
    const selectedChance = ref<SpotChance | null>(null);
    const currentBoard = ref<number[]>([]);
    const results = ref<Results | null>(null);
    const chanceReports = ref<ChanceReports | null>(null);
    const totalBetAmount = ref([0, 0]);

    const isSolverFinished = ref(false);
    store.$subscribe(async (_, store) => {
      if (isSolverFinished.value !== store.isSolverFinished) {
        if ((isSolverFinished.value = store.isSolverFinished)) {
          await init();
        } else {
          clear();
        }
      }
    });

    const init = async () => {
      if (!handler) return;

      const cardsBuffer = [
        await handler.privateCards(0),
        await handler.privateCards(1),
      ];

      cards.value = Array.from({ length: 2 }, (_, player) => {
        return Array.from(cardsBuffer[player]);
      });

      isHandlerUpdated.value = true;
    };

    const initPlayInfo = () => {
      if (!results.value) return;
      if (gameStore.playersInfo.length !== 0) return;

      const oopCards = getRandomItemByWeight(
        cards.value[0],
        results.value?.weights[0]
      );
      const ipCards = getRandomItemByWeight(
        cards.value[1],
        results.value?.weights[1]
      );
      gameStore.playersInfo = [];
      gameStore.playersInfo.push({
        cards: oopCards,
        card1: oopCards & 0xff,
        card2: oopCards >>> 8,
      });

      gameStore.playersInfo.push({
        cards: ipCards,
        card1: ipCards & 0xff,
        card2: ipCards >>> 8,
      });
      gameStore.playerPositionInt = Math.random() < 0.5 ? 0 : 1;
      gameStore.playerPosition =
        gameStore.playerPositionInt === 0 ? "oop" : "ip";

      console.log(
        "position",
        gameStore.playerPositionInt,
        pairText(gameStore.playersInfo[0].cards),
        pairText(gameStore.playersInfo[1].cards)
      );
    };

    // results.value?.weights[0].forEach((index) => {
    //   console.log(cards.value[0][index], results.value?.weights[0][index])
    //   console.log(
    //     pairText(cards.value[0][index]),
    //     results.value?.weights[0][index]
    //   );
    // });

    const pairText = (pair: number) => {
      const card1 = pair & 0xff;
      const card2 = pair >>> 8;
      if (card2 !== 0xff) {
        return [cardText(card2), cardText(card1)];
      } else {
        return [cardText(card1)];
      }
    };

    const clear = () => {
      cards.value = [[], []];
      selectedSpot.value = null;
      selectedChance.value = null;
      results.value = null;
      chanceReports.value = null;
      gameStore.playersInfo = [];
    };

    const onUpdateSpot = (
      newSelectedSpot: Spot | null,
      newSelectedChance: SpotChance | null,
      newCurrentBoard: number[],
      newResults: Results,
      newChanceReports: ChanceReports | null,
      newTotalBetAmount: number[]
    ) => {
      dealtCard.value = -1;
      selectedSpot.value = newSelectedSpot;
      selectedChance.value = newSelectedChance;
      currentBoard.value = newCurrentBoard;
      results.value = newResults;
      chanceReports.value = newChanceReports;
      totalBetAmount.value = newTotalBetAmount;
      isLocked.value = false;

      chanceMode.value = newSelectedChance?.player ?? "";
      initPlayInfo();
    };

    /* Middle Bar */

    const displayMode = ref<DisplayMode>("basics");
    const chanceMode = ref("");

    const displayOptions = ref<DisplayOptions>({
      playerBasics: "auto",
      playerChance: "auto",
      barHeight: "normalized",
      suit: "grouped",
      strategy: "show",
      contentBasics: "default",
      contentGraphs: "eq",
      chartChance: "strategy-combos",
    });

    const copySuccess = ref(0);

    const updateDisplayMode = (mode: DisplayMode) => {
      displayMode.value = mode;
    };

    const updateDisplayOptions = (options: DisplayOptions) => {
      displayOptions.value = options;
    };

    const copyRangeTextToClipboard = async () => {
      const text = "Hello World";
      navigator.clipboard
        .writeText(text)
        .then(() => (copySuccess.value = 1))
        .catch(() => (copySuccess.value = -1));
    };

    const resetCopySuccess = () => {
      copySuccess.value = 0;
    };

    /* Computed */

    const autoPlayerBasics = computed(() => {
      const spot = selectedSpot.value;
      const chance = selectedChance.value;
      if (!spot) return "oop";

      if (chance) {
        return chance.prevPlayer;
      } else if (spot.type === "terminal") {
        return spot.prevPlayer;
      } else {
        return (spot as SpotPlayer).player;
      }
    });

    const autoPlayerChance = computed(() => {
      const spot = selectedSpot.value;
      if (!spot) return "oop";
      if (spot.type === "terminal") {
        return spot.prevPlayer;
      } else {
        return (spot as SpotPlayer).player;
      }
    });

    const displayPlayerBasics = computed(() => {
      const optionPlayer = displayOptions.value.playerBasics;
      if (optionPlayer === "auto") {
        return autoPlayerBasics.value;
      } else {
        return optionPlayer;
      }
    });

    const displayPlayerChance = computed(() => {
      const optionPlayer = displayOptions.value.playerChance;
      if (optionPlayer === "auto") {
        return autoPlayerChance.value;
      } else {
        return optionPlayer;
      }
    });

    /* Results */

    const basicsHoverContent = ref<HoverContent | null>(null);

    const onUpdateHoverContent = (content: HoverContent | null) => {
      basicsHoverContent.value = content;
    };

    const onDealCard = (card: number) => {
      dealtCard.value = card;
    };

    return {
      store,
      isHandlerUpdated,
      isLocked,
      cards,
      dealtCard,
      selectedSpot,
      selectedChance,
      currentBoard,
      results,
      chanceReports,
      totalBetAmount,
      onUpdateSpot,
      displayMode,
      chanceMode,
      displayOptions,
      updateDisplayMode,
      updateDisplayOptions,
      copySuccess,
      copyRangeTextToClipboard,
      resetCopySuccess,
      autoPlayerBasics,
      autoPlayerChance,
      displayPlayerBasics,
      displayPlayerChance,
      basicsHoverContent,
      onUpdateHoverContent,
      onDealCard,
    };
  },
});
</script>
