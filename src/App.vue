<script setup>
import { reactive, ref, onMounted, onUnmounted } from "vue";
import { io } from "socket.io-client";
import { Icon } from "@iconify/vue";
import { useToast } from "vue-toastification";

const toast = useToast();
const socket = io("http://localhost:3000", { autoConnect: false });

let content = reactive(["", "", "", "", "", "", "", "", ""]);
const turn = ref(true);
const is_over = ref(false);
const winner = ref(false);
const is_tie = ref(false);
const val = ref(null);
const name = ref(null);
const displayName = ref(null);

onMounted(() => {
  const username = localStorage.getItem("username");
  if (username) {
    socket.auth = { username };
    socket.connect();
    toast.success("You have joined the game");
  }

  socket.on("session", ({ username }) => {
    console.log(username);
    socket.auth = { username };
    displayName.value = username;
    localStorage.setItem("username", username);
  });

  socket.on("user connected", ({ username }) => {
    toast.success(`${username} has joined the game`);
  });

  socket.on("play", (index) => {
    draw(index, true);
  });
  socket.on("logged", (data) => {
    toast.error(`${data} has left the game`);
  });
  // socket.on("reset", () => {
  //   reset();
  // });
});

onUnmounted(() => {
  // // socket.auth = { username: "" };
  // // localStorage.removeItem("username");
  // // socket.disconnect();
  // // displayName.value = null;
  // toast.error("You have left the game");
});

const connect = () => {
  reset();
  socket.auth = { username: name.value };
  socket.connect();
  toast.success("You have joined the game");
  name.value = null;
};

const disconnect = () => {
  socket.auth = { username: "" };
  localStorage.removeItem("username");
  socket.disconnect();
  displayName.value = null;
  toast.error("You have left the game");
  reset();
};

const draw = (index, drawFromOther) => {
  //send event to client
  if (is_over.value || is_tie.value) {
    return;
  } else {
    if (turn.value) {
      if (content[index] != "") {
        return;
      } else {
        content[index] = "X";
      }
    } else {
      if (content[index] != "") {
        return;
      } else {
        content[index] = "O";
      }
    }
    if (!drawFromOther) {
      socket.emit("play", index);
    }
    turn.value = !turn.value;
    calc_winner();
    calc_tie();
  }
};

const calc_winner = () => {
  const WIN_COND = [
    //rows
    [0, 1, 2],
    [3, 4, 5],
    [6, 7, 8],
    //cols
    [0, 3, 6],
    [1, 4, 7],
    [2, 5, 8],
    //diagonals
    [0, 4, 8],
    [2, 4, 6],
  ];
  for (let i = 0; i < WIN_COND.length; i++) {
    let firstIndex = WIN_COND[i][0];
    let secondIndex = WIN_COND[i][1];
    let thirdIndex = WIN_COND[i][2];
    if (
      content[firstIndex] == content[secondIndex] &&
      content[firstIndex] == content[thirdIndex] &&
      content[firstIndex] != ""
    ) {
      is_over.value = true;
      winner.value = content[firstIndex];
    }
  }
};

const calc_tie = () => {
  for (let i = 0; i <= 8; i++) {
    if (content[i] == "") {
      return;
    }
  }
  is_tie.value = true;
};

const reset = () => {
  content.map((item, index) => {
    content[index] = "";
  });
  winner.value = null;
  is_over.value = false;
  is_tie.value = false;
  // socket.emit("reset");
};
</script>

<template>
  <div id="game">
    <div class="container">
      <div class="row d-flex justify-content-center align-items-center">
        <div class="col-lg-7 col-md-9 game__box">
          <div class="px-1">
            <h1 class="fw-bolder text-white">Tic Tic Toe</h1>
            <p class="lead text-light">
              Tic-tac-toe is a paper-and-pencil game for two players who take
              turns marking the spaces in a three-by-three grid with X or O. The
              player who succeeds in placing three of their marks in a
              horizontal, vertical, or diagonal row is the winner. <br />
              Enter your name to begin the game.
            </p>
            <div class="newsletter__input">
              <input
                type="email"
                placeholder="Enter your name"
                class="form-control"
                v-model="name"
              />
              <button
                v-if="!displayName"
                @click="connect"
                class="btn btn-primary"
                type="button"
                :disabled="!name"
              >
                Connect
              </button>
              <button
                v-if="displayName"
                @click="disconnect"
                class="btn btn-primary"
                type="button"
              >
                Disconnect
              </button>
            </div>

            <div class="mt-2 d-flex align-items-center">
              <Icon
                :color="displayName ? 'green' : 'red'"
                height="20"
                width="20"
                icon="carbon:user-avatar-filled-alt"
              />
              <span v-if="displayName" class="text-light ms-1 small">{{
                displayName
              }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div class="container my-2 pb-5">
    <div class="row d-flex justify-content-center">
      <div class="col-lg-6">
        <div
          v-if="is_over"
          class="alert alert-success d-flex justify-content-between"
        >
          <p class="my-0 py-0">Winner is : {{ winner }}</p>
          <button
            class="btn btn-success btn-sm"
            @click="reset"
            v-if="is_over || is_tie"
          >
            Reset
          </button>
        </div>
        <div
          v-if="is_tie"
          class="alert alert-warning d-flex justify-content-between"
        >
          <p class="my-0 py-0">The game is tied</p>
          <button
            class="btn btn-warning btn-sm"
            @click="reset"
            v-if="is_over || is_tie"
          >
            Reset
          </button>
        </div>
        <div class="play-area mt-2 text-center">
          <div
            v-for="(item, index) in content"
            :key="item + index"
            :id="`block_${index}`"
            class="block shadow rounded"
            @click="draw(index, false)"
          >
            {{ item }}
          </div>
        </div>
      </div>
    </div>
  </div>

  <footer class="bg-dark p-1">
    <div class="container text-center">
      <p class="text-light">
        Powered by <a href="https://abiodunsamuel.com/">Abiodun Digital Hub</a>
      </p>
    </div>
  </footer>
</template>

<style scoped>
#game {
  background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.9)),
    url("./assets/bg.jpg");
}
.game__box {
  /* height: 400px; */
  padding: 3rem 0;
  display: flex;
  justify-content: center;
  align-items: center;
}
.newsletter__input {
  display: flex;
  position: relative;
  align-items: center;
}
.newsletter__input input {
  height: 2.8rem;
  z-index: 1;
}
.newsletter__input button {
  position: absolute;
  right: 3px;
  /* top: 0; */
  z-index: 1000;
}
</style>
