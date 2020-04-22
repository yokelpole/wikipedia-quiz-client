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
      <div v-for="(val, index) in activeQuestion.answers" v-bind:key="val">
        <button v-on:click="submitAnswer(index)" :disabled="answer">{{ val }}</button>
      </div>
      <div>
        <h2 v-if="correctAnswer === answer">CORRECT!</h2>
        <div v-if="answer !== undefined && correctAnswer !== answer">
          <h2>WRONG!</h2>
          <p>The correct answer is {{ activeQuestion.correct_answer }}</p>
        </div>
      </div>
      <div>
        <h5>{{ countdownValue }}</h5>
      </div>
      <ul v-for="player in players" v-bind:key="player.id">
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
    answer: undefined,
    correctAnswer: undefined,
    countdownValue: 0,
    playerName: undefined,
    playerId: undefined,
    players: [],
  }),
  created: async function() {
    const setQuestion = (json) => {
      this.answer = undefined;
      json.answers.forEach((answer, i) => {
        if (answer === json.correct_answer) this.correctAnswer = i;
      });
      json.topic = json.topic.replace(/_/g, " ");
      json.section = json.section.replace(/_/g, " ");
      this.activeQuestion = json;
     }

    const startCountdownTimer = () => {
      setInterval(() => {
        const newValue = parseFloat(this.countdownValue - 0.1).toFixed(1);
        if (newValue > 0) {
          this.countdownValue = newValue;
        }
      }, 100);
    }

    const getQuestionData = async () => {
      const questionResp = await fetch(
        "http://localhost:8090/get_current_question"
      );
      const json = await questionResp.json();
      const activeQuestionStartTime = new Date(
        json["active_question_start_time"]
      );
      const questionTime = json["question_time"];
      const currentTime = new Date(
        new Date().toLocaleString("en-US", { timeZone: "UTC" })
      );
      activeQuestionStartTime.setSeconds(
        activeQuestionStartTime.getSeconds() + questionTime
      );

      return {
        json,
        currentTime,
        activeQuestionStartTime
      };
    }

    const fetchNextQuestion = time => {
      // TODO: Add debounce for when server isn't updated quite yet or in an error condition.
      setTimeout(async () => {
        const {
          json,
          activeQuestionStartTime,
          currentTime
        } = await getQuestionData();
        this.players = json.players.sort((a, b) => b.score - a.score);
        setQuestion(json);
        this.countdownValue = json.question_time;
        fetchNextQuestion(
          activeQuestionStartTime.getTime() - currentTime.getTime()
        );
      }, time);
    }

    const {
      json,
      activeQuestionStartTime,
      currentTime
    } = await getQuestionData();

    this.players = json.players.sort((a, b) => b.score - a.score);
    this.countdownValue = (activeQuestionStartTime - currentTime) / 1000;
    startCountdownTimer();
    setQuestion(json);
    fetchNextQuestion(
      activeQuestionStartTime.getTime() - currentTime.getTime()
    );
  },
  methods: {
    submitName: async function(e) {
      e.preventDefault();
      const playerResp = await fetch("http://localhost:8090/join_game", {
        method: "POST",
        headers: {
          "Accept": "application/json",
          "Content-Type": "application/json",
          "Access-Control-Request-Headers": "x-requested-with"
        },
        body: JSON.stringify({ name: this.playerName })
      });
      const json = await playerResp.json();
      this.playerId = json.ID;
      this.$emit("playerActive", json);
    },
    submitAnswer: async function (index) {
      this.answer = index;

      fetch("http://localhost:8090/answer_question", {
        method: "POST",
        headers: {
          Accept: "application/json",
          "Content-Type": "application/json"
        },
        // FIXME: really shouldn't trust the client to say the answer is right...
        body: JSON.stringify({
          name: this.playerName,
          correct: this.correctAnswer === this.answer
        })
      });
    }
  }
};
</script>