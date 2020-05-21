<template>
  <div>
    <center>
      <h1>THE GREAT WIKIPEDIA QUIZ</h1>
    </center>
    <div v-if="playerId === undefined">
      <form v-on:submit="submitName">
        <p>
          <input v-model="playerName" type="text" placeholder="What is your name?" />
        </p>
        <p>
          <input type="submit" value="Start the quiz!" />
        </p>
      </form>
    </div>
    <div v-if="playerId">
      <h2>{{ activeQuestion.topic }}</h2>
      <h3>{{ activeQuestion.section }}</h3>
      <p>{{ activeQuestion.question }}</p>
      <div class="button-wrapper">
        <div class="button" v-for="(val) in activeQuestion.answers" v-bind:key="val">
          <button v-on:click="submitAnswer(val)" :disabled="answer || countdownValue <= 0">{{ val }}</button>
        </div>
      </div>
      <div>
        <h2 v-if="userIsCorrect === true">CORRECT!</h2>
        <div v-if="userIsCorrect === false">
          <h2>WRONG!</h2>
        </div>
        <div v-if="userIsCorrect === false || answer === undefined && countdownValue <= 0">
          <p>The correct answer is {{ correctAnswer }}</p>
        </div>
      </div>
      <div v-if="countdownValue > 0">
        <div v-bind:style="timeBarStyle"></div>
      </div>
      <ul v-for="player in orderedScoreboard" v-bind:key="player.id">
        <li>{{ player.name }} {{ player.score }} {{ answerIndicator(player.answered_correctly) }}</li>
      </ul>
    </div>
  </div>
</template>

<script>
import moment from "moment-timezone";

export default {
  name: "Quiz",
  data: () => ({
    activeQuestion: {},
    activeTimer: undefined,
    answer: undefined,
    correctAnswer: undefined,
    countdownValue: 0,
    playerName: undefined,
    playerId: undefined,
    serverTimeOffset: undefined,
    userIsCorrect: undefined,
    websocket: undefined,
    players: {}
  }),
  created: async function() {
    const setQuestion = json => {
      this.answer = undefined;
      this.userIsCorrect = undefined;
      json.topic = json.topic.replace(/_/g, " ");
      json.section = json.section.replace(/_/g, " ");
      this.activeQuestion = json;
    };

    const startCountdownTimer = finishTime => {
      const currentTime = new Date(
        new Date().toLocaleString("en-US", { timeZone: "UTC" })
      );
      const serverAdjustedTime = currentTime.setSeconds(
        currentTime.getSeconds() + this.serverTimeOffset
      );
      this.countdownValue = (new Date(finishTime) - serverAdjustedTime) / 1000;
      clearInterval(this.activeTimer);
      this.activeTimer = setInterval(() => {
        const newValue = parseFloat(this.countdownValue - 0.01).toFixed(2);
        if (newValue >= 0) {
          this.countdownValue = newValue;
        } else {
          clearInterval(this.activeTimer);
        }
      }, 10);
    };

    this.websocket = new WebSocket("ws://192.168.0.116:8100");
    const connectionTime = moment().format('x') / 1000;

    this.websocket.onmessage = msg => {
      const data = JSON.parse(msg.data);

      if (data.type === "question_changed") {
        setQuestion(data.value.question);
        startCountdownTimer(data.value.finish_time);
      }
      if (data.type === "player_registered" && !this.playerId) {
        if (data.value.name === this.playerName)
          this.playerId = data.value.player_id;
      }
      if (data.type === "leaderboard_updated") {
        this.players = data.value;
      }
      if (data.type === "correct_answer") {
        this.correctAnswer = data.value;
        if (this.answer) {
          this.userIsCorrect = this.correctAnswer === this.answer;
        }
      }
      if (data.type === "server_time") {
        const clientTime = moment().tz("utc").format('x') / 1000;
        const serverTime = moment.tz(data.value, "utc").format('x') / 1000;
        this.serverTimeOffset = (serverTime - connectionTime) + (connectionTime - clientTime);
      }
    };
  },
  methods: {
    submitName: async function(e) {
      e.preventDefault();
      await this.websocket.send(
        JSON.stringify({
          type: "register_player",
          value: {
            name: this.playerName
          }
        })
      );
    },
    submitAnswer: async function(index) {
      this.answer = index;
      await this.websocket.send(
        JSON.stringify({
          type: "player_answer",
          value: {
            player_id: this.playerId,
            answer: index
          }
        })
      );
    },
    answerIndicator: function(answeredCorrectly) {
      if (answeredCorrectly === null) return;

      return answeredCorrectly ? "✅" : "❌";
    }
  },
  computed: {
    orderedScoreboard: function() {
      const playerMap = Object.entries(this.players).map(([, val]) => val);
      return playerMap.sort((a, b) => b.score > a.score);
    },
    timeBarStyle: function() {
      return {
        margin: "10px",
        height: "30px",
        width: `${(this.countdownValue / 21) * 100}%`,
        background: "white"
      };
    }
  }
};
</script>

<style scoped>
button {
  height: 100%;
  min-height: 125px;
  max-height: 200px;
  width: 100%;
  padding: 10px;
}

.button {
  margin: 10px;
}

.button-wrapper {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-auto-rows: 1fr;
  grid-column-gap: 10px;
  grid-row-gap: 10px;
}
</style>