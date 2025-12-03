<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue';

// 1. **CONFIGURACIÓN DEL ENDPOINT**
const WEBSOCKET_ENDPOINT = 'wss://eszrnakah2.execute-api.eu-south-2.amazonaws.com/production';

// --- ESTADO REACTIVO DEL JUEGO ---
const socket = ref(null);
const board = ref(Array(9).fill(null));
const currentTurn = ref('');
const gameStatus = ref('WAITING'); // Estado real: WAITING, PLAYING, FINISHED
const gameMessage = ref('Esperando conexión...'); // Mensaje para mostrar al usuario
const isMyTurn = ref(false);
const playerMarker = ref(null);
const currentGameId = ref(null);
const gameIdInput = ref('');

// Estado de revancha
const rematchCountdown = ref(0);
const rematchTimer = ref(null);
const waitingForOpponent = ref(false);
const iRequestedRematch = ref(false);

// --- FUNCIONES DE COMUNICACIÓN ---

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

    if (data.youAre && !playerMarker.value) {
      playerMarker.value = data.youAre;
    }
    
    isMyTurn.value = currentTurn.value === playerMarker.value;
    
    // Actualizar el MENSAJE basado en el estado
    if (data.winner) {
      if (data.winner === 'DRAW') {
        gameMessage.value = '¡Empate!';
      } else {
        gameMessage.value = `¡${data.winner} ha ganado!`;
      }
      // Iniciar contador de revancha
      startRematchCountdown();
    } else if (data.status === 'PLAYING') {
       gameMessage.value = isMyTurn.value ? '¡Es tu turno!' : `Turno de ${currentTurn.value}`;
       // Si estaba en revancha y ahora está jugando, limpiar estado
       resetRematchState();
    } else if (data.status === 'WAITING') {
       gameMessage.value = 'Esperando a otro jugador...';
    }
  } else if (data.type === 'rematchWaiting') {
    waitingForOpponent.value = true;
    gameMessage.value = data.message;
  } else if (data.type === 'closeSession') {
    gameMessage.value = data.message;
    stopRematchCountdown();
    // Cerrar el WebSocket después de un breve delay
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

// --- LÓGICA DE REVANCHA ---

const startRematchCountdown = () => {
  rematchCountdown.value = 10;
  iRequestedRematch.value = false;
  waitingForOpponent.value = false;
  
  rematchTimer.value = setInterval(() => {
    rematchCountdown.value--;
    
    if (rematchCountdown.value <= 0) {
      stopRematchCountdown();
      // Tiempo agotado, enviar rechazo
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

// --- LÓGICA DEL JUEGO EN EL FRONTEND ---

const makeMove = (index) => {
  if (gameStatus.value === 'PLAYING' && isMyTurn.value && board.value[index] === null) {
    sendAction('makeMove', { position: index, gameId: currentGameId.value });
  }
};

const joinGame = (gameId) => {
  if (!gameId || gameId.trim() === '') {
    gameMessage.value = 'Error: Ingresa un ID de partida';
    return;
  }
  
  currentGameId.value = gameId;
  playerMarker.value = null;
  board.value = Array(9).fill(null);
  gameMessage.value = 'Uniéndose a la partida...';
  resetRematchState();
  
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
  
  // Cerrar WebSocket si está abierto
  if (socket.value && socket.value.readyState === WebSocket.OPEN) {
    socket.value.close();
  }
  
  currentGameId.value = null;
  playerMarker.value = null;
  board.value = Array(9).fill(null);
  gameStatus.value = 'WAITING';
  gameMessage.value = 'Desconectado. Recarga la página para volver a jugar.';
  gameIdInput.value = '';
};

// --- INICIALIZACIÓN ---

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
  if (socket.value) {
    socket.value.close();
  }
});
</script>

<template>
  <div class="tic-tac-toe-container">
    <h2>Tres en Raya - WebSockets</h2>
    
    <!-- Panel de gestión de partidas -->
    <div v-if="!currentGameId" class="game-lobby">
      <div class="lobby-section">
        <h3>Crear nueva partida</h3>
        <button @click="createNewGame" class="btn-primary">Crear Partida Nueva</button>
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
        <button @click="joinGame(gameIdInput)" class="btn-secondary">Unirse</button>
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
        <p><strong>Mi marcador:</strong> <span class="marker">{{ playerMarker || '...' }}</span></p>
        <p class="status">{{ gameMessage }}</p>
      </div>
      
      <div class="board">
        <div 
          v-for="(cell, index) in board" 
          :key="index" 
          :class="['cell', { 'clickable': gameStatus === 'PLAYING' && isMyTurn && cell === null }]"
          @click="makeMove(index)"
        >
          {{ cell }}
        </div>
      </div>
      
      <!-- Panel de revancha (solo se muestra cuando termina la partida) -->
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
        Salir de la partida
      </button>
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

.tic-tac-toe-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  padding: 20px;
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

@media (prefers-color-scheme: dark) {
  .tic-tac-toe-container {
    background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
  }
}

h2 {
  color: white;
  margin-bottom: 30px;
  font-size: 2.5em;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
}

/* Lobby styles */
.game-lobby {
  background: white;
  padding: 40px;
  border-radius: 15px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
  max-width: 500px;
  width: 100%;
}

@media (prefers-color-scheme: dark) {
  .game-lobby {
    background: #1e1e1e;
    box-shadow: 0 10px 30px rgba(0,0,0,0.6);
  }
}

.lobby-section {
  margin: 20px 0;
  text-align: center;
}

.lobby-section h3 {
  color: #333;
  margin-bottom: 15px;
  font-size: 1.3em;
}

@media (prefers-color-scheme: dark) {
  .lobby-section h3 {
    color: #e0e0e0;
  }
}

.help-text {
  font-size: 0.85em;
  color: #666;
  margin-top: 10px;
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
  margin: 30px 0;
  font-size: 1.2em;
  position: relative;
}

.lobby-divider::before,
.lobby-divider::after {
  content: '';
  position: absolute;
  top: 50%;
  width: 40%;
  height: 1px;
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
  padding: 12px;
  font-size: 1em;
  border: 2px solid #ddd;
  border-radius: 8px;
  margin-bottom: 10px;
}

.game-id-input:focus {
  outline: none;
  border-color: #667eea;
}

@media (prefers-color-scheme: dark) {
  .game-id-input {
    background: #2a2a2a;
    color: #e0e0e0;
    border-color: #444;
  }
  
  .game-id-input:focus {
    border-color: #667eea;
  }
  
  .game-id-input::placeholder {
    color: #888;
  }
}

/* Button styles */
button {
  padding: 12px 24px;
  font-size: 1em;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s;
  font-weight: bold;
}

.btn-primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  width: 100%;
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(102, 126, 234, 0.4);
}

.btn-secondary {
  background: #48bb78;
  color: white;
  width: 100%;
}

.btn-secondary:hover {
  background: #38a169;
  transform: translateY(-2px);
}

.btn-leave {
  background: #f56565;
  color: white;
  margin-top: 20px;
  width: 100%;
}

.btn-leave:hover {
  background: #e53e3e;
}

.btn-copy {
  padding: 4px 8px;
  font-size: 0.9em;
  margin-left: 10px;
  background: #e2e8f0;
  cursor: pointer;
}

.btn-copy:hover {
  background: #cbd5e0;
}

@media (prefers-color-scheme: dark) {
  .btn-copy {
    background: #3a3a3a;
    color: #e0e0e0;
  }
  
  .btn-copy:hover {
    background: #4a4a4a;
  }
}

/* Rematch panel */
.rematch-panel {
  margin-top: 20px;
  padding: 20px;
  background: #f7fafc;
  border-radius: 10px;
  text-align: center;
}

@media (prefers-color-scheme: dark) {
  .rematch-panel {
    background: #2a2a2a;
  }
}

.countdown {
  font-size: 2.5em;
  font-weight: bold;
  color: #667eea;
  margin-bottom: 15px;
  animation: pulse 1s infinite;
}

@media (prefers-color-scheme: dark) {
  .countdown {
    color: #a0d8f1;
  }
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.rematch-buttons {
  display: flex;
  gap: 10px;
  justify-content: center;
}

.btn-rematch {
  background: #48bb78;
  color: white;
  flex: 1;
}

.btn-rematch:hover {
  background: #38a169;
}

.btn-decline {
  background: #f56565;
  color: white;
  flex: 1;
}

.btn-decline:hover {
  background: #e53e3e;
}

.rematch-requested,
.rematch-waiting {
  color: #667eea;
  font-weight: bold;
  margin-top: 10px;
}

@media (prefers-color-scheme: dark) {
  .rematch-requested,
  .rematch-waiting {
    color: #a0d8f1;
  }
}

/* Game panel styles */
.game-panel {
  background: white;
  padding: 30px;
  border-radius: 15px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
  max-width: 400px;
  width: 100%;
}

@media (prefers-color-scheme: dark) {
  .game-panel {
    background: #1e1e1e;
    box-shadow: 0 10px 30px rgba(0,0,0,0.6);
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
  margin-bottom: 10px;
  flex-wrap: wrap;
}

.game-id-display code {
  background: #f7fafc;
  padding: 5px 10px;
  border-radius: 5px;
  font-family: 'Courier New', monospace;
  font-size: 0.9em;
  margin: 0 5px;
}

@media (prefers-color-scheme: dark) {
  .game-id-display code {
    background: #2a2a2a;
    color: #a0d8f1;
  }
}

.game-info p {
  margin: 10px 0;
  color: #333;
}

@media (prefers-color-scheme: dark) {
  .game-info p {
    color: #e0e0e0;
  }
}

.marker {
  font-size: 1.5em;
  font-weight: bold;
  color: #667eea;
}

@media (prefers-color-scheme: dark) {
  .marker {
    color: #a0d8f1;
  }
}

.status {
  font-weight: bold;
  font-size: 1.2em;
  color: #667eea;
  margin: 15px 0;
}

@media (prefers-color-scheme: dark) {
  .status {
    color: #a0d8f1;
  }
}

/* Board styles */
.board {
  display: grid;
  grid-template-columns: repeat(3, 100px);
  grid-template-rows: repeat(3, 100px);
  gap: 5px;
  border: 3px solid #333;
  margin: 20px auto 0 auto;
  border-radius: 5px;
  overflow: hidden;
  justify-self: center;
}

@media (prefers-color-scheme: dark) {
  .board {
    border-color: #555;
  }
}

.cell {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 3em;
  font-weight: bold;
  background-color: #f7fafc;
  cursor: default;
  user-select: none;
  transition: all 0.2s;
}

.clickable {
  cursor: pointer;
  background-color: #e2e8f0;
}

.clickable:hover {
  background-color: #cbd5e0;
  transform: scale(1.05);
}

@media (prefers-color-scheme: dark) {
  .cell {
    background-color: #2a2a2a;
    color: #e0e0e0;
  }
  
  .clickable {
    background-color: #3a3a3a;
  }
  
  .clickable:hover {
    background-color: #4a4a4a;
  }
}

@media (max-width: 600px) {
  .board {
    grid-template-columns: repeat(3, 80px);
    grid-template-rows: repeat(3, 80px);
  }
  
  .cell {
    font-size: 2.5em;
  }
  
  h2 {
    font-size: 2em;
  }
  
  .game-lobby, .game-panel {
    padding: 20px;
  }
}
</style>