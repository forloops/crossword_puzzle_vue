<template>
  <div>
    <h1>Crossword Puzzle</h1>

    <div v-if="timer" class="time-wrapper">
      <span class="current-time">{{ currentTime }}</span>
    </div>

    <div class="actions-container">
      <button v-if="timer" class="action-btn" @click="startPuzzle">Start</button>
      <button class="action-btn" @click="completePuzzle">Complete</button>
    </div>

    <div class="puzzle-table-wrapper">
      <table v-if="result && letters" class="puzzle-table">
        <tbody>
        <tr v-for="(item, i) in result">
          <td v-for="(char, j) in item" :class="{'existed-letter': char !== '.'}" :style="boxStyling">
            <span v-if="idArray[i][j] !== -1" class="word-index">{{idArray[i][j]}}</span>
<!--            {{ char === '.' ? '' : char }}-->
            <div class="input-wrapper"
                 v-if="char !== '.'"
                 :class="{
                   'matched': char === letters[i][j] && completed,
                   'unmatched': char !== letters[i][j] && completed
                 }"
            >
              <input v-if="!completed" type="text" class="puzzle-input" maxlength="1" v-model="letters[i][j]" />
              <span v-else>{{ char === '.' ? '' : char }}</span>
            </div>
          </td>
        </tr>
        </tbody>
      </table>

      <div class="questions-container">
        <div class="questions">
          <div class="title-wrapper">
            <span class="question-title">Down</span>
          </div>
          <div class="question-list">
          <span v-for="item in downQuestions">
            {{ item.id }}. {{ item.question }}
          </span>
          </div>
        </div>

        <div class="questions">
          <div class="title-wrapper">
            <span class="question-title">Across</span>
          </div>
          <div class="question-list">
          <span v-for="item in acrossQuestions">
            {{ item.id }}. {{ item.question }}
          </span>
          </div>
        </div>
      </div>
    </div>


  </div>
</template>

<script>
  // dummy data
  let repeat;

  let horizontalStart;
  let horizontalEnd;
  let verticalStart;
  let verticalEnd;
  let neverVisit;

  let board = [];
  let commonWord;
  let wordIndex;

  let pos_x;
  let pos_y;

  const words = [
    'Apple', 'lion', 'continent', 'puzzle', 'stress', 'background', 'english',
    'usually', 'success', 'experiment', 'basketball', 'football', 'tennis',
    'tiger', 'lion', 'position', 'yesterday', 'umbrella', 'mountain'];

  // dummy data
  const jsonResponse = [];
  for(let i = 0; i < words.length; i++) {
    jsonResponse.push({
      "question": `What is that word ${i+1}?`,
      "answer": words[i],
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    });
  }

  export default {
    name: "CrosswordPuzzle",
    props: [
      'color',
      'timer',
      'puzzleId'
    ],
    data () {
      return {
        completed: false,
        result: [],
        letters: [],
        idArray: [],
        currentTime: 0,
        timerId: null,
        acrossQuestions: [],
        downQuestions: [],
        puzzleWidth: 25,
        puzzleHeight: 17,
        wordDirections: []  // array of word directions 0: across, 1: down
      }
    },
    computed: {
      boxStyling: function () {
        return {
          background: this.color ? this.color : '#ffffff',
          borderColor: this.color ? this.color : '#ffffff',
        }
      }
    },
    mounted() {
      this.initData();
      this.drawPuzzle();
      this.letters = [...this.letters];
    },
    methods: {
      startPuzzle() {
        if (this.timerId) {
          clearInterval(this.timerId);
          this.currentTime = 0;
        }

        this.timerId = setInterval(() => {
          this.currentTime++;
        }, 1000);

        this.completed = false;
        for (let i = 0; i < this.letters.length; i++) {
          for (let j = 0; j < this.letters[i].length; j++) {
            this.letters[i][j] = '';
          }
        }
      },
      completePuzzle() {
        if (this.timerId) {
          clearInterval(this.timerId);
          this.currentTime = 0;
        }

        this.completed = true;
      },
      getRandom (number) {
        return Math.floor(Math.random() * number);
      },
      hasCommonLetter(word1, word2) {
        for (let i = 0; i < word1.length; i++) {
          if (word2.indexOf(word1[i]) !== -1)
            return true;
        }

        return false;
      },
      sortWords() {
        const wordsCnt = words.length;

        wordIndex = new Array(wordsCnt);
        for (let i = 0; i < wordsCnt; ++ i) {
          words[i] = words[i].toLowerCase();
          wordIndex[i] = i;
        }

        for (let i = 0; i < wordsCnt - 1; ++ i) {
          for (let j = i + 1; j < wordsCnt; ++ j) {
            if (words[i].length < words[j].length) {
              const p = words[i];
              words[i] = words[j];
              words[j] = p;
              const q = wordIndex[i];
              wordIndex[i] = wordIndex[j];
              wordIndex[j] = q;
            }
          }
        }

        for (let i = 0; i < wordsCnt - 1; ++ i) {
          for (let j = i + 1; j < wordsCnt; ++ j) {
            if (commonWord[i][j]) {
              const p = words[i + 1];
              words[i + 1] = words[j];
              words[j] = p;
              const q = wordIndex[i + 1];
              wordIndex[i + 1] = wordIndex[j];
              wordIndex[j] = q;
              break;
            }
          }
        }
      },
      initData() {
        //Init Variable
        const maxWordCount = 20;
        this.puzzleWidth	= 25;
        this.puzzleHeight = 19;
        repeat = 100;

        this.wordDirections	= new Array(maxWordCount);
        pos_x = new Array(maxWordCount);
        pos_y = new Array(maxWordCount);

        board = new Array(this.puzzleHeight);
        this.result = new Array(this.puzzleHeight);
        this.letters = new Array(this.puzzleHeight);
        horizontalStart = new Array(this.puzzleHeight);
        horizontalEnd = new Array(this.puzzleHeight);
        verticalStart = new Array(this.puzzleHeight);
        verticalEnd	= new Array(this.puzzleHeight);
        neverVisit = new Array(this.puzzleHeight);
        this.idArray = new Array(this.puzzleHeight);

        for (let i = 0; i < this.puzzleHeight; ++ i) {
          board[i] = new Array (this.puzzleWidth);
          this.result[i] = new Array (this.puzzleWidth);
          this.letters[i] = new Array(this.puzzleWidth);
          horizontalStart[i] = new Array (this.puzzleWidth);
          horizontalEnd[i] = new Array (this.puzzleWidth);
          verticalStart[i] = new Array (this.puzzleWidth);
          verticalEnd[i] = new Array (this.puzzleWidth);
          neverVisit[i] = new Array (this.puzzleWidth);
          this.idArray[i] = new Array(this.puzzleWidth);
        }

        let N = words.length;
        commonWord = new Array (N);
        for (let i = 0; i < N; i ++) {
          commonWord[i] = new Array(N);
        }

        for (let i = 0; i < N; i ++) {
          for (let j = 0; j < N; j ++) {
            if (i === j) commonWord[i][j] = true;
            else commonWord[i][j] = this.hasCommonLetter(words[i], words[j]);
          }
        }

        //Sort words
        this.sortWords();
      },
      initVariable() {
        for (let i = 0; i < this.puzzleHeight; ++ i) {
          for (let j = 0; j < this.puzzleWidth; ++ j) {
            board[i][j] = '.';
            horizontalStart[i][j] = false;
            horizontalEnd[i][j] = false;
            verticalStart[i][j]	= false;
            verticalEnd[i][j]	= false;
            neverVisit[i][j] = false;
          }
        }
      },
      drawPuzzle() {
        let i, j;
        let N = words.length;
        let maxMark = -10000;

        while(true) {
          repeat --;
          if (repeat === 0) break;

          this.initVariable();

          let mark = 0;
          let flag = false;
          for (i = 0; i < N; ++ i) {
            if (i === 0) this.wordDirections[i] = this.getRandom(2);
            else this.wordDirections[i] = 1 - this.wordDirections[i - 1];

            if(words[i].length === 0) {
              if (words[i].length > this.puzzleWidth) {
                flag = true;
                break;
              }
            }

            else {
              if (words[i].length > this.puzzleHeight) {
                flag = true;
                break;
              }
            }

            let error = false;
            let predictPossible = true;
            if (i === 0) predictPossible = false;

            let count = 0;
            let limit = 1;
            let stx, sty, enx, eny;

            let minX, minY, x, y;
            let reverse = false;

            while (true) {
              error = false;
              ++ count;

              if (count > 500 && predictPossible) {
                limit ++;
                count = 0;
                if (limit > i) {
                  if (reverse) {
                    predictPossible = false;
                  }
                  else {
                    reverse = true;
                    this.wordDirections[i] = 1 - this.wordDirections[i];
                    limit = 1;
                  }

                }
              }

              if (count > 600 && !predictPossible) {
                flag = true;
                break;
              }

              if (i >= limit && predictPossible) {
                while (true) {
                  if(commonWord[i][i - limit] && this.wordDirections[i - limit] !== this.wordDirections[i]) break;
                  limit ++;
                  if (limit > i) {
                    if (reverse) {
                      predictPossible = false;
                      break;
                    }
                    else {
                      reverse = true;
                      this.wordDirections[i] = 1 - this.wordDirections[i];
                      limit = 1;
                    }

                  }
                }
                if (predictPossible) {
                  if (this.wordDirections[i] === 0) {
                    minX = pos_x[i - limit] - words[i].length + 1;
                    minY = pos_y[i - limit];
                    pos_x[i] = Math.max(0, minX + this.getRandom(words[i].length)) % (this.puzzleWidth - words[i].length + 1);
                    pos_y[i] = (minY + this.getRandom(words[i - limit].length)) % this.puzzleHeight;
                  }
                  else {
                    minY = pos_y[i - limit] - words[i].length + 1;
                    minX = pos_x[i - limit];
                    pos_y[i] = Math.max(0, minY + this.getRandom(words[i].length)) % (this.puzzleHeight - words[i].length + 1);
                    pos_x[i] = (minX + this.getRandom(words[i - limit].length)) % this.puzzleWidth;
                  }
                }
              }

              if (!predictPossible) {
                if (this.wordDirections[i] === 0) {
                  pos_x[i] = this.getRandom(this.puzzleWidth - words[i].length + 1);
                  pos_y[i] = this.getRandom(this.puzzleHeight);
                }
                else {
                  pos_x[i] = this.getRandom(this.puzzleWidth);
                  pos_y[i] = this.getRandom(this.puzzleHeight - words[i].length + 1);
                }
              }

              if (this.wordDirections[i] === 0) {
                stx = pos_x[i];
                sty = pos_y[i];
                eny = pos_y[i];
                enx = pos_x[i] + words[i].length - 1;

                if (horizontalStart[sty][stx] || horizontalEnd[eny][enx]) continue;
              }
              else {
                stx = pos_x[i];
                enx = pos_x[i];
                sty = pos_y[i];
                eny = pos_y[i] + words[i].length - 1;

                if (verticalStart[sty][stx] || verticalEnd[eny][enx]) continue;
              }

              for(j = 0; j < words[i].length; ++ j) {
                if (this.wordDirections[i] === 0) {
                  y = pos_y[i];
                  x = pos_x[i] + j;
                }
                else {
                  y = pos_y[i] + j;
                  x = pos_x[i];
                }
                if (neverVisit[y][x]) {
                  error = true;
                  break;
                }
                if (board[y][x] === '.') continue;
                if (board[y][x] !== words[i][j]) {
                  error = true;
                  break;
                }
              }

              if (error) continue;
              break;
            }

            if(flag) break;
            if(!predictPossible) mark -= 100;
            //Vertical, Horizontal
            if (this.wordDirections[i] === 0) {
              if (sty > 0) {
                for (j = stx; j <= enx; ++ j) {
                  horizontalStart[sty - 1][j] = true;
                  horizontalEnd[sty - 1][j] = true;
                  verticalEnd[sty - 1][j] = true;
                }
              }
              for (j = stx; j <= enx; ++ j) {
                horizontalStart[sty][j] = true;
                horizontalEnd[sty][j] = true;
              }
              if (sty < this.puzzleHeight - 1) {
                for (j = stx; j <= enx; ++ j) {
                  horizontalStart[sty + 1][j] = true;
                  horizontalEnd[sty + 1][j] = true;
                  verticalStart[sty + 1][j] = true;
                }
              }
              if (stx > 0) {
                neverVisit[sty][stx - 1] = true;
              }
              if (enx < this.puzzleWidth - 1) {
                neverVisit[sty][enx + 1] = true;
              }
            }
            else {
              if (stx > 0) {
                for (j = sty; j <= eny; ++ j) {
                    verticalStart[j][stx - 1] = true;
                    verticalEnd[j][stx - 1] = true;
                    horizontalEnd[j][stx - 1] = true;
                }
              }
              for (j = sty; j <= eny; ++ j) {
                  verticalStart[j][stx] = true;
                  verticalEnd[j][stx] = true;
              }
              if (stx < this.puzzleWidth - 1) {
                for (j = sty; j <= eny; ++ j) {
                    verticalStart[j][stx + 1] = true;
                    verticalEnd[j][stx + 1] = true;
                    horizontalStart[j][stx + 1] = true;
                }
              }
              if (sty > 0) {
                neverVisit[sty - 1][stx] = true;
              }
              if (eny < this.puzzleHeight - 1) {
                neverVisit[eny + 1][stx] = true;
              }
            }

            for (j = 0; j < words[i].length; ++ j) {
              if (this.wordDirections[i] === 0) {
                y = pos_y[i];
                x = pos_x[i] + j;
              } else {
                y = pos_y[i] + j;
                x = pos_x[i];
              }
                board[y][x] = words[i][j];
            }
          }

          if(flag) continue;

          for (i = 0; i < this.puzzleHeight; ++ i) {
            for (j = 0; j < this.puzzleWidth; ++ j) {
              if (board[i][j] === '.') mark ++;
            }
          }

          if (maxMark < mark) {
            console.log(maxMark);
            maxMark = mark;
            for (i = 0; i < this.puzzleHeight; ++ i) {
              for (j = 0; j < this.puzzleWidth; ++ j) {
                this.result[i][j] = board[i][j];
              }
            }
            for (i = 0; i < this.puzzleHeight; ++ i) {
              for (j = 0; j < this.puzzleWidth; ++ j) {
                this.idArray[i][j] = -1;
              }
            }
            this.acrossQuestions = [];
            this.downQuestions = [];

            for (i = 0; i < N; i ++) {
              this.idArray[pos_y[i]][pos_x[i]] = wordIndex[i] + 1;

              if(this.wordDirections[i] === 0) {
                this.acrossQuestions.push({
                  id: wordIndex[i] + 1,
                  question: jsonResponse[wordIndex[i]].question
                });
              }
              else {
                this.downQuestions.push({
                  id: wordIndex[i] + 1,
                  question: jsonResponse[wordIndex[i]].question
                });
              }
            }

            this.acrossQuestions.sort((a, b) => {
              return a.id - b.id;
            });

            this.downQuestions.sort((a, b) => {
              return a.id - b.id;
            });
          }
        }
      }
    }
  }
</script>

<style lang="scss" scoped>
  .time-wrapper {
    .current-time {
      font-size: 30px;
    }
  }

  .actions-container {
    display: flex;
    width: 100%;
    justify-content: center;

    .action-btn {
      width: 100px;
      padding: 7px 10px;
      border: none;
      outline: none;
      margin: 10px;
      border-radius: 3px;
      box-shadow: 1px 1px 3px rgba(0, 0, 0, 0.1);
      background: #E91E63;
      color: #fff;

      &:hover {
        cursor: pointer;
        background: #dc3d73;
      }
    }
  }

  .puzzle-table-wrapper {
    width: 100%;
    display: flex;
    justify-content: center;

    .puzzle-table {
      border-collapse: collapse;

      td {
        width: 25px;
        height: 25px;
        border: 1px solid #ffffff;
        position: relative;

        &.existed-letter {
          background: white !important;
        }

        .word-index {
          position: absolute;
          top: 0px;
          left: 0px;
          font-size: 10px;
        }

        .input-wrapper {
          .puzzle-input {
            width: 20px;
            border: none;
            outline: none;
            text-align: center;
          }
        }

        .matched {
          color: #42b983;
        }

        .unmatched {
          color: #ff2343;
        }
      }
    }
  }

  .questions-container {
    .questions {
      margin: 0 30px 30px 30px;

      .title-wrapper {
        text-align: left;
        margin-bottom: 10px;

        .question-title {
          font-size: 20px;
          font-weight: bold;
        }
      }

      .question-list {
        display: flex;
        flex-direction: column;
      }
    }
  }
</style>
