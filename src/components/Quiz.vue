<template>
  <div>
    <center>
      <h1>THE GREAT WIKIPEDIA QUIZ</h1>
    </center>
    <div v-if="playerId === undefined">
      <p>What is your name?</p>
      <form v-on:submit="submitName">
        <input v-model="playerName" type="text" />
        <input type="submit" />
      </form>
    </div>
    <div v-if="playerId">
      <h2>{{ activeQuestion.topic }}</h2>
      <h3>{{ activeQuestion.section }}</h3>
      <p>{{ activeQuestion.question }}</p>
      <div v-for="(val) in activeQuestion.answers" v-bind:key="val">
        <button v-on:click="submitAnswer(val)" :disabled="answer">{{ val }}</button>
      </div>
      <div>
        <h2 v-if="userIsCorrect === true">CORRECT!</h2>
        <div v-if="userIsCorrect === false">
          <h2>WRONG!</h2>
          <p>The correct answer is {{ correctAnswer }}</p>
        </div>
      </div>
      <div>
        <h5>{{ countdownValue }}</h5>
      </div>
      <ul v-for="player in orderedScoreboard" v-bind:key="player.id">
        <li>{{ player.name }} {{ player.score }}</li>
      </ul>
    </div>
  </div>
</template>

<script>
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
      this.countdownValue = (new Date(finishTime) - currentTime) / 1000;
      clearInterval(this.activeTimer);
      this.activeTimer = setInterval(() => {
        const newValue = parseFloat(this.countdownValue - 0.01).toFixed(2);
        if (newValue > 0) {
          this.countdownValue = newValue;
        }
      }, 10);
    };

    this.websocket = new WebSocket("ws://127.0.0.1:8100");
    this.websocket.onmessage = msg => {
      const data = JSON.parse(msg.data);

      if (data.type === "question_changed") {
        setQuestion(data.value.question);
        startCountdownTimer(data.value.finish_time);
      }
      if (data.type === "player_registered") this.playerId = data.value;
      if (data.type === "leaderboard_updated") this.players = data.value;
      if (data.type === "correct_answer") {
        this.correctAnswer = data.value;
        this.userIsCorrect = this.correctAnswer === this.answer;
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
  },
  computed: {
    orderedScoreboard: function () {
      const playerMap = Object.entries(this.players).map(([, val]) => val);
      return playerMap.sort((a, b) => b.score > a.score);
    }
  }
};
</script>