# snake-ladder
game


**To implement the features you described, you'll need to add several key components to your game system:

1. **Options to Create or Join a Room**
2. **Waiting Screen Until Two Players Join**
3. **Option to Choose a Game Type After Both Players Have Joined**
4. **Start the Chosen Game**

Here’s how you can achieve this:

### **1. Server-side Updates**

#### **Update `server.js`**

1. **Room Management**

Add the functionality to handle game type selection and start the chosen game.

```javascript
const express = require("express");
const http = require("http");
const socketIo = require("socket.io");
const path = require("path");
const { v4: uuidv4 } = require("uuid");

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

const rooms = {}; // To keep track of rooms and their players

app.use(express.static(path.join(__dirname, "public")));

// Create a new room
app.get("/create-room", (req, res) => {
  const roomId = uuidv4(); // Generate a unique room ID
  rooms[roomId] = { players: [], playerTurn: 1, gameType: null }; // Initialize room
  res.json({ roomId });
});

// Join a room
app.post("/join-room", express.json(), (req, res) => {
  const { roomId, playerId } = req.body;
  if (rooms[roomId] && rooms[roomId].players.length < 2) {
    rooms[roomId].players.push(playerId);
    res.json({ success: true });
  } else {
    res.json({ success: false, message: "Room is full or does not exist." });
  }
});

io.on("connection", (socket) => {
  let roomId;
  let playerId;

  // Handle joining a room
  socket.on("join-room", ({ roomId: id, playerId: id }) => {
    roomId = id;
    playerId = id;

    if (rooms[roomId] && rooms[roomId].players.length < 2) {
      rooms[roomId].players.push(playerId);
      socket.join(roomId);
      socket.emit("player-id", { playerId });
      io.to(roomId).emit("update-players", { players: rooms[roomId].players });

      // Notify both players when room is full
      if (rooms[roomId].players.length === 2) {
        io.to(roomId).emit("room-full", { gameType: rooms[roomId].gameType });
      }
    } else {
      socket.emit("error", { message: "Room is full or does not exist." });
    }
  });

  // Handle game type selection
  socket.on("select-game", ({ roomId, gameType }) => {
    if (rooms[roomId]) {
      rooms[roomId].gameType = gameType;
      io.to(roomId).emit("game-start", { gameType });
    }
  });

  // Handle roll-dice
  socket.on("roll-dice", ({ playerId, diceRoll }) => {
    const newPosition = movePlayer(playerId, diceRoll);
    const room = rooms[roomId];

    // Check if any player has won
    const winner = playerPositions.findIndex(pos => pos >= 100) + 1;
    if (winner) {
      io.to(roomId).emit("game-finished", { winner });
    } else {
      io.to(roomId).emit("update-game", { playerId, newPosition });
      io.to(roomId).emit("dice-result", { diceRoll });
    }
  });

  socket.on("disconnect", () => {
    if (roomId && rooms[roomId]) {
      rooms[roomId].players = rooms[roomId].players.filter(id => id !== playerId);
      io.to(roomId).emit("update-players", { players: rooms[roomId].players });
      if (rooms[roomId].players.length === 0) {
        delete rooms[roomId]; // Clean up room if empty
      }
    }
    console.log(`Player disconnected`);
  });
});

function movePlayer(playerId, diceRoll) {
  // Implement the game logic here
  playerPositions[playerId - 1] += diceRoll;

  const snakesAndLadders = {
    4: 25, 21: 32, 29: 77,
    30: 7, 43: 75, 47: 16,
    55: 12, 63: 71, 80: 89,
    78: 60, 82: 42, 93: 56,
    99: 76
  };

  if (snakesAndLadders[playerPositions[playerId - 1]]) {
    playerPositions[playerId - 1] = snakesAndLadders[playerPositions[playerId - 1]];
  }

  if (playerPositions[playerId - 1] > 100) {
    playerPositions[playerId - 1] = 100; // Ensure the player does not exceed the board
  }

  return playerPositions[playerId - 1];
}

server.listen(8080, () => {
  console.log(`Server is running on port 8080`);
});
```

### **2. Client-side Updates**

#### **Update `script.js`**

Add functionality to handle room creation, joining, and game selection.

1. **Create and Join Room**

```javascript
document.getElementById("create-room").addEventListener("click", async () => {
  const response = await fetch("/create-room");
  const data = await response.json();
  const roomId = data.roomId;
  alert(`Room created! Share this ID: ${roomId}`);
  // Redirect or show the next UI for game type selection
  showWaitingScreen(roomId);
});

document.getElementById("join-room").addEventListener("click", () => {
  const roomId = document.getElementById("join-room-id").value;
  if (roomId) {
    const playerId = Math.floor(Math.random() * 1000) + 1;
    socket.emit("join-room", { roomId, playerId });
  }
});
```

2. **Handle Room Full and Game Type Selection**

```javascript
socket.on("room-full", ({ gameType }) => {
  // Show game type selection UI
  showGameSelectionScreen();
});

socket.on("game-start", ({ gameType }) => {
  // Hide waiting screen and show the selected game
  hideWaitingScreen();
  startGame(gameType);
});
```

3. **Show Waiting Screen**

Implement functions to manage UI changes:

```javascript
function showWaitingScreen(roomId) {
  document.getElementById("waiting-screen").style.display = "block";
  document.getElementById("game-selection").style.display = "none";
  document.getElementById("waiting-room-id").textContent = roomId;
}

function hideWaitingScreen() {
  document.getElementById("waiting-screen").style.display = "none";
}

function showGameSelectionScreen() {
  document.getElementById("game-selection").style.display = "block";
}
```

4. **Handle Game Selection**

```javascript
document.getElementById("select-game").addEventListener("change", (event) => {
  const selectedGame = event.target.value;
  const roomId = document.getElementById("waiting-room-id").textContent;
  socket.emit("select-game", { roomId, gameType: selectedGame });
});
```

5. **Start Game Based on Selection**

```javascript
function startGame(gameType) {
  if (gameType === "snake-ladder") {
    // Initialize Snake and Ladder game
  } else if (gameType === "tic-tac-toe") {
    // Initialize Tic Tac Toe game
  } else if (gameType === "ludo") {
    // Initialize Ludo game
  }
}
```

### **3. HTML Updates**

Add the following elements to your HTML:

```html
<!-- UI for creating and joining rooms -->
<div id="room-management">
  <button id="create-room">Create Room</button>
  <input type="text" id="join-room-id" placeholder="Enter room ID to join" />
  <button id="join-room">Join Room</button>
</div>

<!-- Waiting screen -->
<div id="waiting-screen" style="display: none;">
  <p>Waiting for another player to join...</p>
  <p>Room ID: <span id="waiting-room-id"></span></p>
</div>

<!-- Game selection -->
<div id="game-selection" style="display: none;">
  <select id="select-game">
    <option value="snake-ladder">Snake and Ladder</option>
    <option value="tic-tac-toe">Tic Tac Toe</option>
    <option value="ludo">Ludo</option>
  </select>
</div>
```

### Summary of Changes:

1. **Server-side (`server.js`)**: Added endpoints and logic for room creation, joining, and game type selection.
2. **Client-side (`script.js`)**: Added functions to manage room creation, joining, game type selection, and game start.
3. **HTML**: Added UI elements for creating/joining rooms, waiting for players, and selecting a game.

This setup will create a private game room system where players can create or join a room, wait until both players join, select a game, and start playing.**
