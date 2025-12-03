<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue';

// CONFIGURACIÓN
const WEBSOCKET_ENDPOINT = 'wss://eszrnakah2.execute-api.eu-south-2.amazonaws.com/production';
const ROWS = 6;
const COLS = 7;

// ESTADO REACTIVO
const socket = ref(null);
const board = ref(Array(ROWS * COLS).fill(null));
const currentTurn = ref('');
const gameStatus = ref('WAITING');
const gameMessage = ref('Esperando conexión...');
const isMyTurn = ref(false);
const playerMarker = ref(null);
const currentGameId = ref(null);
const gameIdInput = ref('');

// Sistema de puntuación
const score = ref({ RED: 0, YELLOW: 0, DRAW: 0 });

// Indicador de última jugada
const lastMove = ref(null);

// Animación de victoria
const winningPositions = ref(null);

// Temporizador de turno (40 segundos)
const turnTimeLeft = ref(40);
const turnTimer = ref(null);
const turnStartTime = ref(null);

// Estado de votación
const myVote = ref(null);
const votingInProgress = ref(false);

// Estado de revancha
const rematchCountdown = ref(0);
const rematchTimer = ref(null);
const waitingForOpponent = ref(false);
const iRequestedRematch = ref(false);

// Hover state para columnas
const hoveredColumn = ref(null);

// Computed: Convertir board plano a matriz 2D
const boardGrid = computed(() => {
  const grid = [];
  for (let row = 0; row < ROWS; row++) {
    grid.push(board.value.slice(row * COLS, (row + 1) * COLS));
  }
  return grid;
});

// Computed: Obtener color CSS según marcador
const getMarkerColor = (marker) => {
  if (marker === 'RED') return '#e53e3e';
  if (marker === 'YELLOW') return '#ecc94b';
  return 'transparent';
};

// COMUNICACIÓN

const sendAction = (action, payload = {}) => {
  if (socket.value && socket.value.readyState === WebSocket.OPEN) {
    const message = JSON.stringify({ action, ...payload });
    socket.value.send(message);
    console.log(`Mensaje enviado: ${action}`, payload);
  } else {
    console.error('WebSocket no está conectado.');
  }
};

const handleMessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Mensaje recibido:', data);

  if (data.type === 'gameUpdate') {
    board.value = data.board;
    currentTurn.value = data.turn;
    gameStatus.value = data.status;

    if (data.youAre) {
      playerMarker.value = data.youAre;
    }
    
    // Actualizar puntuación
    if (data.score) {
      score.value = data.score;
    }
    
    // Actualizar última jugada
    lastMove.value = data.lastMove;
    
    // Actualizar posiciones ganadoras
    if (data.winningPositions) {
      winningPositions.value = data.winningPositions;
    } else {
      winningPositions.value = null;
    }
    
    isMyTurn.value = currentTurn.value === playerMarker.value;
    
    // Iniciar temporizador si es mi turno y el juego está activo
    if (data.status === 'PLAYING' && data.turnStartTime) {
      startTurnTimer(data.turnStartTime);
    } else {
      stopTurnTimer();
    }
    
    if (data.status === 'VOTING') {
      votingInProgress.value = true;
      myVote.value = null;
      gameMessage.value = data.message || '¿Quién debe comenzar?';
      stopTurnTimer();
    } else if (data.winner) {
      votingInProgress.value = false;
      stopTurnTimer();
      if (data.winner === 'DRAW') {
        gameMessage.value = '¡Empate!';
      } else {
        gameMessage.value = `¡${data.winner} ha ganado!`;
      }
      startRematchCountdown();
    } else if (data.status === 'PLAYING') {
       votingInProgress.value = false;
       gameMessage.value = data.message || (isMyTurn.value ? '¡Es tu turno!' : `Turno de ${currentTurn.value}`);
       resetRematchState();
    } else if (data.status === 'WAITING') {
       votingInProgress.value = false;
       stopTurnTimer();
       gameMessage.value = 'Esperando a otro jugador...';
    }
  } else if (data.type === 'voteRegistered') {
    gameMessage.value = data.message;
  } else if (data.type === 'waitingVote') {
    gameMessage.value = data.message;
  } else if (data.type === 'rematchWaiting') {
    waitingForOpponent.value = true;
    gameMessage.value = data.message;
  } else if (data.type === 'closeSession') {
    gameMessage.value = data.message;
    stopRematchCountdown();
    stopTurnTimer();
    setTimeout(() => {
      if (socket.value) {
        socket.value.close();
      }
      leaveGame();
    }, 2000);
  } else if (data.type === 'error') {
    gameMessage.value = `Error: ${data.message || data.error}`;
  }
};

// LÓGICA DE TEMPORIZADOR DE TURNO

const startTurnTimer = (serverStartTime) => {
  stopTurnTimer();
  turnStartTime.value = serverStartTime;
  
  const updateTimer = () => {
    const now = Date.now() / 1000;
    const elapsed = now - turnStartTime.value;
    const remaining = Math.max(0, 40 - Math.floor(elapsed));
    turnTimeLeft.value = remaining;
    
    if (remaining <= 0) {
      stopTurnTimer();
      // El servidor manejará el timeout
    }
  };
  
  updateTimer();
  turnTimer.value = setInterval(updateTimer, 100);
};

const stopTurnTimer = () => {
  if (turnTimer.value) {
    clearInterval(turnTimer.value);
    turnTimer.value = null;
  }
  turnTimeLeft.value = 40;
};

// LÓGICA DE REVANCHA

const startRematchCountdown = () => {
  rematchCountdown.value = 10;
  iRequestedRematch.value = false;
  waitingForOpponent.value = false;
  
  rematchTimer.value = setInterval(() => {
    rematchCountdown.value--;
    
    if (rematchCountdown.value <= 0) {
      stopRematchCountdown();
      sendAction('declineRematch', { gameId: currentGameId.value });
    }
  }, 1000);
};

const stopRematchCountdown = () => {
  if (rematchTimer.value) {
    clearInterval(rematchTimer.value);
    rematchTimer.value = null;
  }
  rematchCountdown.value = 0;
};

const resetRematchState = () => {
  stopRematchCountdown();
  iRequestedRematch.value = false;
  waitingForOpponent.value = false;
};

const requestRematch = () => {
  iRequestedRematch.value = true;
  sendAction('requestRematch', { gameId: currentGameId.value });
};

const declineRematch = () => {
  stopRematchCountdown();
  sendAction('declineRematch', { gameId: currentGameId.value });
};

// LÓGICA DEL JUEGO

const voteForTurn = (vote) => {
  if (votingInProgress.value && !myVote.value) {
    myVote.value = vote;
    sendAction('voteTurn', { vote: vote, gameId: currentGameId.value });
  }
};

const makeMove = (col) => {
  if (gameStatus.value === 'PLAYING' && isMyTurn.value) {
    sendAction('makeMove', { column: col, gameId: currentGameId.value });
  }
};

const handleColumnHover = (col) => {
  if (gameStatus.value === 'PLAYING' && isMyTurn.value) {
    hoveredColumn.value = col;
  }
};

const handleColumnLeave = () => {
  hoveredColumn.value = null;
};

const isColumnFull = (col) => {
  // Verificar si la fila superior de esta columna está ocupada
  return board.value[col] !== null;
};

const joinGame = (gameId) => {
  if (!gameId || gameId.trim() === '') {
    gameMessage.value = 'Error: Ingresa un ID de partida';
    return;
  }
  
  currentGameId.value = gameId;
  playerMarker.value = null;
  board.value = Array(ROWS * COLS).fill(null);
  gameMessage.value = 'Uniéndose a la partida...';
  resetRematchState();
  score.value = { RED: 0, YELLOW: 0, DRAW: 0 };
  lastMove.value = null;
  stopTurnTimer();
  
  sendAction('joinGame', { gameId: gameId });
};

const createNewGame = () => {
  const newGameId = `game_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  gameIdInput.value = newGameId;
  joinGame(newGameId);
};

const copyGameId = () => {
  if (navigator.clipboard && navigator.clipboard.writeText) {
    navigator.clipboard.writeText(currentGameId.value).then(() => {
      console.log('ID copiado al portapapeles');
    }).catch(err => {
      console.error('Error al copiar:', err);
    });
  } else {
    const textArea = document.createElement('textarea');
    textArea.value = currentGameId.value;
    document.body.appendChild(textArea);
    textArea.select();
    try {
      document.execCommand('copy');
      console.log('ID copiado al portapapeles (fallback)');
    } catch (err) {
      console.error('Error al copiar:', err);
    }
    document.body.removeChild(textArea);
  }
};

const leaveGame = () => {
  resetRematchState();
  stopTurnTimer();
  
  if (currentGameId.value && socket.value && socket.value.readyState === WebSocket.OPEN) {
    sendAction('leaveGame', { gameId: currentGameId.value });
  }
  
  if (socket.value && socket.value.readyState === WebSocket.OPEN) {
    socket.value.close();
  }
  
  currentGameId.value = null;
  playerMarker.value = null;
  board.value = Array(ROWS * COLS).fill(null);
  gameStatus.value = 'WAITING';
  gameMessage.value = 'Desconectado. Recarga la página para volver a jugar.';
  gameIdInput.value = '';
  votingInProgress.value = false;
  myVote.value = null;
  score.value = { RED: 0, YELLOW: 0, DRAW: 0 };
  lastMove.value = null;
};

// INICIALIZACIÓN

onMounted(() => {
  socket.value = new WebSocket(WEBSOCKET_ENDPOINT);

  socket.value.onopen = () => {
    gameMessage.value = 'Conectado. Crea o únete a una partida.';
  };

  socket.value.onmessage = handleMessage;

  socket.value.onclose = () => {
    gameMessage.value = 'Desconectado del servidor.';
    resetRematchState();
  };

  socket.value.onerror = (error) => {
    console.error('Error de WebSocket:', error);
    gameMessage.value = 'Error de conexión.';
  };
});

onUnmounted(() => {
  resetRematchState();
  stopTurnTimer();
  if (socket.value) {
    socket.value.close();
  }
});
</script>

<template>
  <div class="connect4-container">
    <h1>🔴 Cuatro en Raya 🟡</h1>
    
    <!-- Panel de lobby -->
    <div v-if="!currentGameId" class="game-lobby">
      <div class="lobby-section">
        <h3>Crear nueva partida</h3>
        <button @click="createNewGame" class="btn-primary">🎮 Crear Partida Nueva</button>
        <p class="help-text">Crea una partida y comparte el ID con tu oponente</p>
      </div>
      
      <div class="lobby-divider">O</div>
      
      <div class="lobby-section">
        <h3>Unirse a partida existente</h3>
        <input 
          v-model="gameIdInput" 
          type="text" 
          placeholder="Ingresa el ID de la partida"
          class="game-id-input"
          @keyup.enter="joinGame(gameIdInput)"
        />
        <button @click="joinGame(gameIdInput)" class="btn-secondary">🚀 Unirse</button>
        <p class="help-text">Pide el ID a quien creó la partida</p>
      </div>
    </div>

    <!-- Panel de juego -->
    <div v-else class="game-panel">
      <div class="game-info">
        <div class="game-id-display">
          <strong>ID de Partida:</strong> 
          <code>{{ currentGameId }}</code>
          <button @click="copyGameId" class="btn-copy" title="Copiar ID">
            📋
          </button>
        </div>
        
        <!-- Scoreboard -->
        <div class="scoreboard">
          <div class="score-item">
            <div class="score-disc red"></div>
            <span class="score-value">{{ score.RED }}</span>
          </div>
          <div class="score-item draw">
            <span class="score-label">Empates</span>
            <span class="score-value">{{ score.DRAW }}</span>
          </div>
          <div class="score-item">
            <div class="score-disc yellow"></div>
            <span class="score-value">{{ score.YELLOW }}</span>
          </div>
        </div>
        
        <div class="player-info">
          <div class="player-marker">
            <span>Mi color:</span>
            <div 
              class="color-circle" 
              :style="{ backgroundColor: getMarkerColor(playerMarker) }"
            ></div>
          </div>
          
          <!-- Temporizador de turno -->
          <div v-if="gameStatus === 'PLAYING'" class="turn-timer">
            <div class="timer-bar-container">
              <div 
                class="timer-bar" 
                :style="{ width: `${(turnTimeLeft / 40) * 100}%` }"
                :class="{ 'timer-warning': turnTimeLeft <= 10 }"
              ></div>
            </div>
            <span class="timer-text" :class="{ 'timer-warning-text': turnTimeLeft <= 10 }">
              ⏱️ {{ turnTimeLeft }}s
            </span>
          </div>
        </div>
        
        <p class="status">{{ gameMessage }}</p>
      </div>
      
      <!-- Panel de votación -->
      <div v-if="votingInProgress" class="voting-panel">
        <h3>¿Quién debe comenzar?</h3>
        <p class="voting-description">
          Si ambos eligen el mismo, ese jugador comienza.<br>
          Si no coinciden, se decidirá aleatoriamente.
        </p>
        <div class="vote-buttons">
          <button 
            @click="voteForTurn('RED')" 
            class="btn-vote btn-vote-red"
            :class="{ 'voted': myVote === 'RED' }"
            :disabled="myVote !== null"
          >
            <div class="vote-disc red"></div>
            ROJO comienza
          </button>
          <button 
            @click="voteForTurn('YELLOW')" 
            class="btn-vote btn-vote-yellow"
            :class="{ 'voted': myVote === 'YELLOW' }"
            :disabled="myVote !== null"
          >
            <div class="vote-disc yellow"></div>
            AMARILLO comienza
          </button>
        </div>
      </div>
      
      <!-- Tablero de juego -->
      <div class="board-wrapper">
        <div class="board">
          <!-- Columnas clicables -->
          <div 
            v-for="col in COLS" 
            :key="`col-${col}`"
            class="column"
            :class="{ 
              'column-hover': hoveredColumn === col - 1 && gameStatus === 'PLAYING' && isMyTurn,
              'column-full': isColumnFull(col - 1),
              'column-disabled': gameStatus !== 'PLAYING' || !isMyTurn
            }"
            @click="makeMove(col - 1)"
            @mouseenter="handleColumnHover(col - 1)"
            @mouseleave="handleColumnLeave"
          >
            <!-- Celdas de la columna -->
            <div 
              v-for="row in ROWS" 
              :key="`cell-${row}-${col}`"
              class="cell"
            >
              <div 
                class="disc"
                :class="{ 
                  'disc-red': boardGrid[row - 1][col - 1] === 'RED',
                  'disc-yellow': boardGrid[row - 1][col - 1] === 'YELLOW',
                  'disc-last-move': lastMove === ((row - 1) * COLS + (col - 1)),
                  'disc-winning': winningPositions && winningPositions.includes((row - 1) * COLS + (col - 1))
                }"
              ></div>
            </div>
          </div>
        </div>
      </div>
      
      <!-- Panel de revancha -->
      <div v-if="gameStatus === 'FINISHED' && rematchCountdown > 0" class="rematch-panel">
        <div class="countdown">{{ rematchCountdown }}s</div>
        <p v-if="waitingForOpponent" class="rematch-waiting">Esperando al otro jugador...</p>
        <div v-if="!iRequestedRematch" class="rematch-buttons">
          <button @click="requestRematch" class="btn-rematch">
            🔄 Revancha
          </button>
          <button @click="declineRematch" class="btn-decline">
            ❌ Rechazar
          </button>
        </div>
        <p v-else class="rematch-requested">Has aceptado la revancha. Esperando al oponente...</p>
      </div>
      
      <button @click="leaveGame" class="btn-leave">
        🚪 Salir de la partida
      </button>
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

.connect4-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  padding: 20px;
  min-height: 100vh;
  background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
  overflow-y: auto;
}

@media (prefers-color-scheme: dark) {
  .connect4-container {
    background: linear-gradient(135deg, #0f1419 0%, #1a2332 100%);
  }
}

h1 {
  color: white;
  margin-bottom: 20px;
  font-size: clamp(1.8em, 4vw, 2.5em);
  text-shadow: 3px 3px 6px rgba(0,0,0,0.4);
  text-align: center;
}

/* LOBBY STYLES */
.game-lobby {
  background: white;
  padding: 40px;
  border-radius: 20px;
  box-shadow: 0 15px 40px rgba(0,0,0,0.3);
  max-width: 600px;
  width: 100%;
}

@media (prefers-color-scheme: dark) {
  .game-lobby {
    background: #1e1e1e;
  }
}

.lobby-section {
  margin: 25px 0;
  text-align: center;
}

.lobby-section h3 {
  color: #333;
  margin-bottom: 20px;
  font-size: 1.5em;
}

@media (prefers-color-scheme: dark) {
  .lobby-section h3 {
    color: #e0e0e0;
  }
}

.help-text {
  font-size: 0.9em;
  color: #666;
  margin-top: 12px;
}

@media (prefers-color-scheme: dark) {
  .help-text {
    color: #a0a0a0;
  }
}

.lobby-divider {
  text-align: center;
  color: #999;
  font-weight: bold;
  margin: 35px 0;
  font-size: 1.3em;
  position: relative;
}

.lobby-divider::before,
.lobby-divider::after {
  content: '';
  position: absolute;
  top: 50%;
  width: 40%;
  height: 2px;
  background: #ddd;
}

@media (prefers-color-scheme: dark) {
  .lobby-divider {
    color: #888;
  }
  
  .lobby-divider::before,
  .lobby-divider::after {
    background: #444;
  }
}

.lobby-divider::before {
  left: 0;
}

.lobby-divider::after {
  right: 0;
}

.game-id-input {
  width: 100%;
  padding: 15px;
  font-size: 1.1em;
  border: 2px solid #ddd;
  border-radius: 10px;
  margin-bottom: 12px;
  transition: border-color 0.3s;
}

.game-id-input:focus {
  outline: none;
  border-color: #2a5298;
}

@media (prefers-color-scheme: dark) {
  .game-id-input {
    background: #2a2a2a;
    color: #e0e0e0;
    border-color: #444;
  }
  
  .game-id-input:focus {
    border-color: #4a7bc8;
  }
}

/* BUTTON STYLES */
button {
  padding: 15px 30px;
  font-size: 1.1em;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.3s;
  font-weight: bold;
}

.btn-primary {
  background: linear-gradient(135deg, #e53e3e 0%, #dd6b20 100%);
  color: white;
  width: 100%;
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(229, 62, 62, 0.4);
}

.btn-secondary {
  background: linear-gradient(135deg, #38a169 0%, #2f855a 100%);
  color: white;
  width: 100%;
}

.btn-secondary:hover {
  background: linear-gradient(135deg, #2f855a 0%, #276749 100%);
  transform: translateY(-2px);
}

.btn-leave {
  background: #718096;
  color: white;
  margin-top: 25px;
  width: 100%;
}

.btn-leave:hover {
  background: #4a5568;
}

.btn-copy {
  padding: 6px 12px;
  font-size: 1em;
  margin-left: 10px;
  background: #e2e8f0;
}

.btn-copy:hover {
  background: #cbd5e0;
}

@media (prefers-color-scheme: dark) {
  .btn-copy {
    background: #3a3a3a;
  }
  
  .btn-copy:hover {
    background: #4a4a4a;
  }
}

/* GAME PANEL */
.game-panel {
  background: white;
  padding: 25px;
  border-radius: 20px;
  box-shadow: 0 15px 40px rgba(0,0,0,0.3);
  max-width: 850px;
  width: 100%;
}

@media (prefers-color-scheme: dark) {
  .game-panel {
    background: #1e1e1e;
  }
}

.game-info {
  margin-bottom: 20px;
  text-align: center;
}

.game-id-display {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 12px;
  flex-wrap: wrap;
  font-size: 1em;
}

.game-id-display code {
  background: #f7fafc;
  padding: 6px 12px;
  border-radius: 8px;
  font-family: 'Courier New', monospace;
  font-size: 0.9em;
  margin: 0 8px;
}

@media (prefers-color-scheme: dark) {
  .game-id-display code {
    background: #2a2a2a;
    color: #a0d8f1;
  }
}

/* SCOREBOARD */
.scoreboard {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 25px;
  margin: 15px 0;
  padding: 15px;
  background: linear-gradient(135deg, #f7fafc 0%, #e2e8f0 100%);
  border-radius: 12px;
  box-shadow: 0 3px 10px rgba(0,0,0,0.1);
}

@media (prefers-color-scheme: dark) {
  .scoreboard {
    background: linear-gradient(135deg, #2a2a2a 0%, #1a1a1a 100%);
  }
}

.score-item {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 1.3em;
  font-weight: bold;
  color: #2c5282;
}

@media (prefers-color-scheme: dark) {
  .score-item {
    color: #4a7bc8;
  }
}

.score-item.draw {
  flex-direction: column;
  gap: 2px;
}

.score-label {
  font-size: 0.7em;
  color: #718096;
}

@media (prefers-color-scheme: dark) {
  .score-label {
    color: #a0a0a0;
  }
}

.score-value {
  font-size: 1.4em;
  color: #1a365d;
}

@media (prefers-color-scheme: dark) {
  .score-value {
    color: #90cdf4;
  }
}

.score-disc {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  border: 2px solid rgba(0,0,0,0.2);
  box-shadow: 0 2px 6px rgba(0,0,0,0.2);
}

.score-disc.red {
  background: radial-gradient(circle at 30% 30%, #fc8181, #e53e3e);
}

.score-disc.yellow {
  background: radial-gradient(circle at 30% 30%, #faf089, #ecc94b);
}

.player-info {
  margin: 12px 0;
}

.player-marker {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  font-size: 1.1em;
  font-weight: bold;
  color: #333;
  margin-bottom: 10px;
}

@media (prefers-color-scheme: dark) {
  .player-marker {
    color: #e0e0e0;
  }
}

.color-circle {
  width: 35px;
  height: 35px;
  border-radius: 50%;
  border: 3px solid #333;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
}

@media (prefers-color-scheme: dark) {
  .color-circle {
    border-color: #666;
  }
}

/* TEMPORIZADOR DE TURNO */
.turn-timer {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  margin-top: 10px;
}

.timer-bar-container {
  width: 200px;
  height: 12px;
  background: #e2e8f0;
  border-radius: 10px;
  overflow: hidden;
  box-shadow: inset 0 2px 4px rgba(0,0,0,0.1);
}

@media (prefers-color-scheme: dark) {
  .timer-bar-container {
    background: #2a2a2a;
  }
}

.timer-bar {
  height: 100%;
  background: linear-gradient(90deg, #48bb78 0%, #38a169 100%);
  transition: width 0.1s linear;
  border-radius: 10px;
}

.timer-bar.timer-warning {
  background: linear-gradient(90deg, #fc8181 0%, #e53e3e 100%);
  animation: pulse-timer 0.5s infinite;
}

@keyframes pulse-timer {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

.timer-text {
  font-size: 1.1em;
  font-weight: bold;
  color: #38a169;
}

.timer-text.timer-warning-text {
  color: #e53e3e;
  animation: pulse-text 0.5s infinite;
}

@keyframes pulse-text {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.status {
  font-weight: bold;
  font-size: 1.2em;
  color: #2a5298;
  margin: 12px 0;
}

@media (prefers-color-scheme: dark) {
  .status {
    color: #4a7bc8;
  }
}

/* BOARD STYLES */
.board-wrapper {
  display: flex;
  justify-content: center;
  margin: 20px 0;
}

.board {
  display: inline-grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 4px;
  background: #2c5282;
  padding: 12px;
  border-radius: 15px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
}

@media (prefers-color-scheme: dark) {
  .board {
    background: #1a365d;
  }
}

.column {
  display: flex;
  flex-direction: column;
  gap: 4px;
  cursor: pointer;
  transition: transform 0.2s;
  border-radius: 8px;
  padding: 4px;
}

.column:not(.column-disabled):not(.column-full):hover {
  transform: scale(1.05);
  background: rgba(255, 255, 255, 0.1);
}

.column-full {
  cursor: not-allowed;
  opacity: 0.7;
}

.column-disabled {
  cursor: default;
}

.cell {
  width: clamp(45px, 7vw, 70px);
  height: clamp(45px, 7vw, 70px);
  background: #1a365d;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: inset 0 2px 8px rgba(0,0,0,0.3);
}

@media (prefers-color-scheme: dark) {
  .cell {
    background: #0f1e33;
  }
}

.disc {
  width: 85%;
  height: 85%;
  border-radius: 50%;
  background: transparent;
  transition: all 0.3s;
  box-shadow: 0 4px 8px rgba(0,0,0,0.2);
}

.disc-red {
  background: radial-gradient(circle at 30% 30%, #fc8181, #e53e3e);
  animation: drop 0.5s ease-out;
}

.disc-yellow {
  background: radial-gradient(circle at 30% 30%, #faf089, #ecc94b);
  animation: drop 0.5s ease-out;
}

/* INDICADOR DE ÚLTIMA JUGADA */
.disc-last-move {
  box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.8), 0 4px 12px rgba(0,0,0,0.3);
  animation: pulse-last-move 1.5s infinite;
}

@keyframes pulse-last-move {
  0%, 100% { 
    box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.8), 0 4px 12px rgba(0,0,0,0.3);
  }
  50% { 
    box-shadow: 0 0 0 5px rgba(255, 255, 255, 1), 0 4px 16px rgba(0,0,0,0.4);
  }
}

/* ANIMACIÓN DE VICTORIA */
.disc-winning {
  animation: celebrate 0.8s ease-in-out infinite;
  box-shadow: 0 0 20px rgba(255, 215, 0, 0.8), 0 4px 12px rgba(0,0,0,0.3);
  transform: scale(1.1);
}

@keyframes celebrate {
  0%, 100% { 
    transform: scale(1.1) rotate(0deg);
    box-shadow: 0 0 20px rgba(255, 215, 0, 0.8), 0 4px 12px rgba(0,0,0,0.3);
  }
  25% { 
    transform: scale(1.15) rotate(-5deg);
    box-shadow: 0 0 30px rgba(255, 215, 0, 1), 0 4px 16px rgba(0,0,0,0.4);
  }
  75% { 
    transform: scale(1.15) rotate(5deg);
    box-shadow: 0 0 30px rgba(255, 215, 0, 1), 0 4px 16px rgba(0,0,0,0.4);
  }
}

@keyframes drop {
  0% {
    transform: translateY(-300px) scale(0);
  }
  60% {
    transform: translateY(10px) scale(1.1);
  }
  100% {
    transform: translateY(0) scale(1);
  }
}

/* REMATCH PANEL */
.rematch-panel {
  margin-top: 25px;
  padding: 25px;
  background: #f7fafc;
  border-radius: 15px;
  text-align: center;
}

@media (prefers-color-scheme: dark) {
  .rematch-panel {
    background: #2a2a2a;
  }
}

/* VOTING PANEL */
.voting-panel {
  margin: 25px 0;
  padding: 30px;
  background: linear-gradient(135deg, #f7fafc 0%, #e2e8f0 100%);
  border-radius: 15px;
  text-align: center;
  box-shadow: 0 5px 20px rgba(0,0,0,0.1);
}

@media (prefers-color-scheme: dark) {
  .voting-panel {
    background: linear-gradient(135deg, #2a2a2a 0%, #1a1a1a 100%);
  }
}

.voting-panel h3 {
  color: #2c5282;
  font-size: 1.8em;
  margin-bottom: 15px;
}

@media (prefers-color-scheme: dark) {
  .voting-panel h3 {
    color: #4a7bc8;
  }
}

.voting-description {
  color: #4a5568;
  font-size: 1em;
  margin-bottom: 25px;
  line-height: 1.6;
}

@media (prefers-color-scheme: dark) {
  .voting-description {
    color: #a0a0a0;
  }
}

.vote-buttons {
  display: flex;
  gap: 20px;
  justify-content: center;
  flex-wrap: wrap;
}

.btn-vote {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 18px 30px;
  font-size: 1.2em;
  font-weight: bold;
  border: 3px solid transparent;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s;
  flex: 1;
  min-width: 200px;
  max-width: 280px;
}

.btn-vote:disabled {
  cursor: default;
  opacity: 0.6;
}

.btn-vote-red {
  background: linear-gradient(135deg, #fc8181 0%, #e53e3e 100%);
  color: white;
}

.btn-vote-red:hover:not(:disabled) {
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(229, 62, 62, 0.4);
  border-color: #c53030;
}

.btn-vote-red.voted {
  border-color: #c53030;
  box-shadow: 0 0 20px rgba(229, 62, 62, 0.6);
  animation: pulse-vote 1.5s infinite;
}

.btn-vote-yellow {
  background: linear-gradient(135deg, #faf089 0%, #ecc94b 100%);
  color: #744210;
}

.btn-vote-yellow:hover:not(:disabled) {
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(236, 201, 75, 0.4);
  border-color: #d69e2e;
}

.btn-vote-yellow.voted {
  border-color: #d69e2e;
  box-shadow: 0 0 20px rgba(236, 201, 75, 0.6);
  animation: pulse-vote 1.5s infinite;
}

.vote-disc {
  width: 35px;
  height: 35px;
  border-radius: 50%;
  border: 2px solid rgba(0,0,0,0.2);
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
}

.vote-disc.red {
  background: radial-gradient(circle at 30% 30%, #fc8181, #e53e3e);
}

.vote-disc.yellow {
  background: radial-gradient(circle at 30% 30%, #faf089, #ecc94b);
}

@keyframes pulse-vote {
  0%, 100% { 
    transform: scale(1); 
  }
  50% { 
    transform: scale(1.05); 
  }
}

.countdown {
  font-size: 3em;
  font-weight: bold;
  color: #e53e3e;
  margin-bottom: 20px;
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.15); }
}

.rematch-buttons {
  display: flex;
  gap: 15px;
  justify-content: center;
}

.btn-rematch {
  background: #38a169;
  color: white;
  flex: 1;
  max-width: 200px;
}

.btn-rematch:hover {
  background: #2f855a;
}

.btn-decline {
  background: #e53e3e;
  color: white;
  flex: 1;
  max-width: 200px;
}

.btn-decline:hover {
  background: #c53030;
}

.rematch-requested,
.rematch-waiting {
  color: #2a5298;
  font-weight: bold;
  margin-top: 15px;
  font-size: 1.1em;
}

@media (prefers-color-scheme: dark) {
  .rematch-requested,
  .rematch-waiting {
    color: #4a7bc8;
  }
}

/* RESPONSIVE */
@media (max-width: 768px) {
  .board {
    padding: 10px;
  }
  
  .cell {
    width: clamp(40px, 9vw, 55px);
    height: clamp(40px, 9vw, 55px);
  }
  
  .game-lobby, .game-panel {
    padding: 20px;
  }
  
  h1 {
    font-size: 1.8em;
    margin-bottom: 15px;
  }
  
  .scoreboard {
    gap: 15px;
    padding: 12px;
  }
  
  .score-item {
    font-size: 1.1em;
  }
}

@media (max-width: 480px) {
  .cell {
    width: clamp(35px, 11vw, 50px);
    height: clamp(35px, 11vw, 50px);
  }
  
  .board {
    gap: 3px;
    padding: 8px;
  }
  
  .column {
    gap: 3px;
  }
  
  .scoreboard {
    flex-direction: column;
    gap: 10px;
  }
  
  .timer-bar-container {
    width: 150px;
  }
}
</style>