<template>
  <div class="lobby-container">
    <div class="lobby-card glass-effect">
      <h1 class="title">Conecta 4 - Sala</h1>

      <!-- Código de sala -->
      <div class="room-info">
        <span class="room-label">Código de la sala:</span>
        <span class="room-code">{{ lobbyGameId }}</span>
      </div>

      <!-- Esperando jugador -->
      <div v-if="!opponentJoined" class="waiting-area">
        <span class="loading-text">Esperando a que otro jugador se una...</span>
        <div class="spinner"></div>
      </div>

      <!-- Juego activo -->
      <div v-else class="game-area">

        <!-- Turno y estado -->
        <div class="turn-info">
          <span v-if="gameStatus === 'PLAYING'">
            Turno de: 
            <strong>{{ currentTurn === playerMarker ? "Tú" : "Oponente" }}</strong>
          </span>

          <span v-else-if="gameStatus === 'FINISHED'">
            <strong v-if="winner === playerMarker">¡Ganaste! 🎉</strong>
            <strong v-else-if="winner === getOpponentMarker(playerMarker)">Perdiste 😢</strong>
            <strong v-else-if="winner === 'EMPATE'">Empate 🤝</strong>
          </span>
        </div>

        <!-- Tablero -->
        <div class="board">
          <div 
            v-for="(column, colIndex) in gameBoard" 
            :key="colIndex" 
            class="column"
            @click="handleColumnClick(colIndex)"
          >
            <div 
              v-for="(cell, rowIndex) in column" 
              :key="rowIndex" 
              class="cell"
              :class="{
                red: cell === 'R',
                yellow: cell === 'Y',
                highlight: isWinningCell(colIndex, rowIndex)
              }"
            ></div>
          </div>
        </div>

        <!-- Botón Reset -->
        <div class="reset-area" v-if="gameStatus === 'FINISHED'">
          <button class="reset-button" @click="resetGame">Jugar de nuevo</button>
        </div>

      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted } from "vue";
import { supabase } from "@/supabase";

export default {
  name: "LobbyGame",

  setup() {
    const lobbyGameId = ref("");
    const gameBoard = ref([]);
    const playerMarker = ref(null);
    const currentTurn = ref(null);
    const opponentJoined = ref(false);
    const gameStatus = ref("WAITING");
    const winner = ref(null);
    const winningCells = ref([]);

    let channel = null;

    const initializeBoard = () => {
      gameBoard.value = Array.from({ length: 7 }, () => Array(6).fill(null));
    };

    const getOpponentMarker = (marker) => (marker === "R" ? "Y" : "R");

    const isWinningCell = (col, row) =>
      winningCells.value.some(c => c.col === col && c.row === row);

    const loadGameState = async () => {
      const { data } = await supabase
        .from("games")
        .select("*")
        .eq("id", lobbyGameId.value)
        .single();

      if (!data) return;

      gameBoard.value = data.board;
      currentTurn.value = data.current_turn;
      playerMarker.value = data.player_marker;
      opponentJoined.value = !!data.player2;
      gameStatus.value = data.status;
      winner.value = data.winner;
      winningCells.value = data.winning_cells || [];
    };

    const handleColumnClick = async (col) => {
      if (gameStatus.value !== "PLAYING") return;
      if (currentTurn.value !== playerMarker.value) return;

      const column = gameBoard.value[col];
      const emptyRow = column.lastIndexOf(null);
      if (emptyRow === -1) return;

      column[emptyRow] = playerMarker.value;

      await supabase
        .from("games")
        .update({
          board: gameBoard.value,
          current_turn: getOpponentMarker(playerMarker.value)
        })
        .eq("id", lobbyGameId.value);
    };

    const resetGame = async () => {
      initializeBoard();

      await supabase
        .from("games")
        .update({
          board: gameBoard.value,
          status: "PLAYING",
          winner: null,
          winning_cells: [],
          current_turn: "R"
        })
        .eq("id", lobbyGameId.value);
    };

    onMounted(async () => {
      lobbyGameId.value = localStorage.getItem("lobbyGameId");
      initializeBoard();
      await loadGameState();

      channel = supabase
        .channel("game-updates-" + lobbyGameId.value)
        .on(
          "postgres_changes",
          { event: "*", schema: "public", table: "games", filter: `id=eq.${lobbyGameId.value}` },
          (payload) => {
            const updated = payload.new;
            gameBoard.value = updated.board;
            currentTurn.value = updated.current_turn;
            opponentJoined.value = !!updated.player2;
            gameStatus.value = updated.status;
            winner.value = updated.winner;
            winningCells.value = updated.winning_cells || [];
          }
        )
        .subscribe();
    });

    onUnmounted(() => {
      if (channel) supabase.removeChannel(channel);
    });

    return {
      lobbyGameId,
      gameBoard,
      playerMarker,
      currentTurn,
      opponentJoined,
      gameStatus,
      winner,
      winningCells,
      isWinningCell,
      getOpponentMarker,
      handleColumnClick,
      resetGame
    };
  },
};
</script>

<style scoped>
.lobby-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

.lobby-card {
  padding: 30px;
  border-radius: 12px;
  width: 420px;
  text-align: center;
}

.title {
  font-size: 26px;
  margin-bottom: 20px;
}

.room-info {
  margin-bottom: 20px;
  font-size: 18px;
}

.room-label {
  font-weight: bold;
}

.room-code {
  font-size: 22px;
  font-weight: bold;
}

.waiting-area {
  margin-top: 20px;
  font-size: 18px;
}

.spinner {
  margin: 20px auto;
  width: 28px;
  height: 28px;
  border: 4px solid #ccc;
  border-top-color: #ffcc00;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.board {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

.column {
  display: flex;
  flex-direction: column-reverse;
  margin: 2px;
  cursor: pointer;
}

.cell {
  width: 55px;
  height: 55px;
  margin: 3px;
  border-radius: 50%;
  background: #e0e0e0;
}

.red {
  background: red;
}

.yellow {
  background: yellow;
}

.highlight {
  box-shadow: 0 0 10px 4px #00ff00;
}

.reset-area {
  margin-top: 20px;
}

.reset-button {
  padding: 10px 18px;
  font-size: 16px;
  border-radius: 8px;
  cursor: pointer;
}
</style>
