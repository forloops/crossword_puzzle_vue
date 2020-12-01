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
      <table class="puzzle-table">
        <tbody>
        <tr v-for="(item, i) in result">
          <td v-for="(char, j) in item" :class="{'existed-letter': char.targetChar !== '*'}" :style="boxStyling">
            <span v-if="wordStartIndex[i][j] !== -1" class="word-index">{{wordStartIndex[i][j]}}</span>
            <div class="input-wrapper"
                 v-if="char.targetChar !== '*'"
                 :class="{
                   'matched': char.targetChar === letters[i][j] && completed,
                   'unmatched': char.targetChar !== letters[i][j] && completed
                 }"
            >
              <input v-if="!completed" type="text" class="puzzle-input" maxlength="1" v-model="letters[i][j]" />
              <span v-else>{{ char.targetChar === '*' ? '' : char.targetChar}}</span>
            </div>
          </td>
        </tr>
        </tbody>
      </table>
    </div>

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
</template>

<script>
  import swal from 'sweetalert2'

  const GRID_WIDTH = 25, GRID_HEIGHT = 17, EMPTYCHAR = '*';

  // dummy data
  const jsonResponse = [
    {
      "question": 'What is that word 1?',
      "answer": 'apple',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 2?',
      "answer": 'bear',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 3?',
      "answer": 'continent',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 4?',
      "answer": 'puzzle',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 5?',
      "answer": 'stress',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 6?',
      "answer": 'background',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 7?',
      "answer": 'english',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 8?',
      "answer": 'usually',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 9?',
      "answer": 'success',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 10?',
      "answer": 'experiment',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 11?',
      "answer": 'basketball',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 12?',
      "answer": 'football',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 13?',
      "answer": 'tennis',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 14?',
      "answer": 'tiger',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 15?',
      "answer": 'lion',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 16?',
      "answer": 'position',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 17?',
      "answer": 'yesterday',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 18?',
      "answer": 'umbrella',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
    {
      "question": 'What is that word 19?',
      "answer": 'mountain',
      "helpText": 'This help text will be used on some questions as a hint and will only show up in the list of some questions'
    },
  ];

  const words = jsonResponse.map((item, index) => ({
    word: item.answer,
    clue: '',
    x: 0,
    y: 0,
    vertical: false,
    index
  }));

  export default {
    name: "CrosswordPuzzle",
    props: [
      'color',
      'timer',
      'puzzleId'
    ],
    data () {
      return {
        state: true,
        result: [],
        letters: [],
        activeWordList: [], //keeps array of words actually placed in board
        wordStartIndex: [],
        acrossCount: 0,
        downCount: 0,
        acrossQuestions: [],
        downQuestions: [],
        currentTime: 0,
        timerId: null,
        completed: false
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
      if (this.timer) {
        this.initBoardResult();
      }
      else {
        this.startPuzzle();
      }

      this.initData();
    },
    methods: {
      initData() {
        for(let i = 0; i < GRID_HEIGHT; i++) {
          this.letters[i] = [];
          for(let j = 0; j < GRID_WIDTH; j++) {
            this.letters[i][j] = '';
          }
        }

        this.letters = [...this.letters];
      },
      initBoardResult() {
        for(let i = 0; i < GRID_HEIGHT; i++) {
          this.result[i] = [];
          for(let j = 0; j < GRID_WIDTH; j++) {
            this.result[i][j] = {};
            this.result[i][j].targetChar = '*';
          }
        }

        this.initWordStartIndex();
        this.result = [...this.result];
      },
      initWordStartIndex(activeWordList = []) {
        for(let i = 0; i < GRID_HEIGHT; i++) {
          this.wordStartIndex[i] = [];
          for(let j = 0; j < GRID_WIDTH; j++) {
            this.wordStartIndex[i][j] = -1;
          }
        }

        activeWordList.forEach((item) => {
          if (this.wordStartIndex[item.x][item.y] === -1) {
            this.wordStartIndex[item.x][item.y] = item.index + 1;
          }
          else {
            this.wordStartIndex[item.x][item.y] += '/' + (item.index + 1);
          }
        });
      },
      estimateResult() {
        let matchCnt = 0;

        for(let i = 0; i < this.activeWordList.length; i++) {
          const word = this.activeWordList[i].word;

          let flag = true;

          for(let j = 0; j < word.length; j++) {
            let x, y;

            if (this.activeWordList[i].vertical) {
              x = this.activeWordList[i].x + j;
              y = this.activeWordList[i].y;
            }
            else {
              x = this.activeWordList[i].x;
              y = this.activeWordList[i].y + j;
            }

            if (this.result[x][y].targetChar !== this.letters[x][y]) {
              flag = false;
            }
          }

          if (flag) matchCnt++;
        }

        return matchCnt;
      },
      startPuzzle() {
        const boardResult = this.board(25, 17, words);
        this.result = boardResult.grid;

        const acrossWords = boardResult.activeWordList.filter((item) => !item.vertical);
        const downWords = boardResult.activeWordList.filter((item) => item.vertical);

        this.activeWordList = boardResult.activeWordList;

        this.acrossQuestions = acrossWords.map((item) => ({
          id: item.index + 1,
          question: jsonResponse[item.index].question
        }));
        this.downQuestions = downWords.map((item) => ({
          id: item.index + 1,
          question: jsonResponse[item.index].question
        }));

        this.acrossQuestions.sort((a, b) => {
          return a.id - b.id;
        });

        this.downQuestions.sort((a, b) => {
          return a.id - b.id;
        });

        this.initWordStartIndex(boardResult.activeWordList);

        if (this.timerId) {
          clearInterval(this.timerId);
          this.currentTime = 0;
        }

        this.timerId = setInterval(() => {
          this.currentTime++;
        }, 1000);

        this.completed = false;
        this.initData();
      },
      completePuzzle() {
        const correctWordsCnt = this.estimateResult();
        const wrongWordsCnt = jsonResponse.length - correctWordsCnt;

        console.log('Correct Words: ', correctWordsCnt);
        console.log('Wrong Words: ', wrongWordsCnt);
        console.log('Spent Time: ', this.currentTime);

        swal({
          type: 'success',
          title: `Correct: ${correctWordsCnt} words`,
          text: 'Thank you for completing the puzzle',
          confirmButtonText: 'OK',
        });

        if (this.timerId) {
          clearInterval(this.timerId);
          this.currentTime = 0;
        }

        this.completed = true;
      },
      board(columnCnt, rowCnt, words) { //instantiator object for making gameboards
        const cols = columnCnt;
        const rows = rowCnt;
        let activeWordList = []; //keeps array of words actually placed in board
        let acrossCount = 0;
        let downCount = 0;
        let coordList = [];
        let grid = []; //create 2 dimensional array for letter grid

        let wordArray = words.sort((a, b) => {
          return b.word.length - a.word.length;
        });

        for (let i = 0; i < rows; i++) {
          grid[i] = [];
        }

        function suggestCoords(word) { //search for potential cross placement locations

          let c = '';
          let coordCount = 0;
          for (let i = 0; i < word.length; i++) { //cycle through each character of the word
            for (let x = 0; x < GRID_HEIGHT; x++) {
              for (let y = 0; y < GRID_WIDTH; y++) {
                c = word[i];
                if (grid[x][y].targetChar === c) { //check for letter match in cell
                  if (x - i + 1> 0 && x - i + word.length-1 < GRID_HEIGHT) { //would fit vertically?
                    coordList[coordCount] = {};
                    coordList[coordCount].x = x - i;
                    coordList[coordCount].y = y;
                    coordList[coordCount].score = 0;
                    coordList[coordCount].vertical = true;
                    coordCount++;
                  }

                  if (y - i + 1 > 0 && y - i + word.length-1 < GRID_WIDTH) { //would fit horizontally?
                    coordList[coordCount] = {};
                    coordList[coordCount].x = x;
                    coordList[coordCount].y = y - i;
                    coordList[coordCount].score = 0;
                    coordList[coordCount].vertical = false;
                    coordCount++;
                  }
                }
              }
            }
          }
        }

        function checkFitScore(word, x, y, vertical) {

          let fitScore = 1; //default is 1, 2+ has crosses, 0 is invalid due to collision

          if (vertical) { //vertical checking
            for (let i = 0; i < word.length; i++) {
              if (i === 0 && x > 0) { //check for empty space preceeding first character of word if not on edge
                if (grid[x - 1][y].targetChar !== EMPTYCHAR) { //adjacent letter collision
                  fitScore = 0;
                  break;
                }
              } else if (i === word.length - 1 && x + i < GRID_HEIGHT - 1) { //check for empty space after last character of word if not on edge
                if (grid[x+i+1][y].targetChar !== EMPTYCHAR) { //adjacent letter collision
                  fitScore = 0;
                  break;
                }
              }
              if (x + i < GRID_HEIGHT) {
                if (grid[x + i][y].targetChar === word[i]) { //letter match - aka cross point
                  fitScore += 1;
                } else if (grid[x + i][y].targetChar !== EMPTYCHAR) { //letter doesn't match and it isn't empty so there is a collision
                  fitScore = 0;
                  break;
                } else { //verify that there aren't letters on either side of placement if it isn't a crosspoint
                  if (y < GRID_WIDTH - 1) { //check right side if it isn't on the edge
                    if (grid[x + i][y + 1].targetChar !== EMPTYCHAR) { //adjacent letter collision
                      fitScore = 0;
                      break;
                    }
                  }
                  if (y > 0) { //check left side if it isn't on the edge
                    if (grid[x + i][y - 1].targetChar !== EMPTYCHAR) { //adjacent letter collision
                      fitScore = 0;
                      break;
                    }
                  }
                }
              }
            }
          } else { //horizontal checking
            for (let i = 0; i < word.length; i++) {
              if (i === 0 && y > 0) { //check for empty space preceeding first character of word if not on edge
                if (grid[x][y-1].targetChar !== EMPTYCHAR) { //adjacent letter collision
                  fitScore = 0;
                  break;
                }
              } else if (i === word.length - 1 && y + i < GRID_WIDTH -1) { //check for empty space after last character of word if not on edge
                if (grid[x][y + i + 1].targetChar !== EMPTYCHAR) { //adjacent letter collision
                  fitScore = 0;
                  break;
                }
              }
              if (y + i < GRID_WIDTH) {
                if (grid[x][y + i].targetChar === word[i]) { //letter match - aka cross point
                  fitScore += 1;
                } else if (grid[x][y + i].targetChar !== EMPTYCHAR) { //letter doesn't match and it isn't empty so there is a collision
                  fitScore = 0;
                  break;
                } else { //verify that there aren't letters on either side of placement if it isn't a crosspoint
                  if (x < GRID_HEIGHT) { //check top side if it isn't on the edge
                    if (grid[x + 1][y + i].targetChar !== EMPTYCHAR) { //adjacent letter collision
                      fitScore = 0;
                      break;
                    }
                  }
                  if (x > 0) { //check bottom side if it isn't on the edge
                    if (grid[x - 1][y + i].targetChar !== EMPTYCHAR) { //adjacent letter collision
                      fitScore = 0;
                      break;
                    }
                  }
                }
              }

            }
          }

          return fitScore;
        }

        function placeWord(word, clue, x, y, vertical, index) { //places a new active word on the board
          let wordPlaced = false;

          if (vertical) {
            if (word.length + x < GRID_HEIGHT) {
              for (let i = 0; i < word.length; i++) {
                grid[x + i][y].targetChar = word[i];
              }
              wordPlaced = true;
            }
          } else {
            if (word.length + y < GRID_WIDTH) {
              for (let i = 0; i < word.length; i++) {
                grid[x][y + i].targetChar = word[i];
              }
              wordPlaced = true;
            }
          }

          if (wordPlaced) {
            let currentIndex = activeWordList.length;
            activeWordList[currentIndex] = {};
            activeWordList[currentIndex].word = word;
            activeWordList[currentIndex].clue = clue;
            activeWordList[currentIndex].x = x;
            activeWordList[currentIndex].y = y;
            activeWordList[currentIndex].vertical = vertical;
            activeWordList[currentIndex].index = index;

            if (activeWordList[currentIndex].vertical) {
              downCount++;
              activeWordList[currentIndex].number = downCount;
            } else {
              acrossCount++;
              activeWordList[currentIndex].number = acrossCount;
            }
          }

        }

        function isActiveWord(word) {

          if (activeWordList.length > 0) {
            for (let w = 0; w < activeWordList.length; w++) {
              if (word === activeWordList[w].word) {
                return true;
              }
            }
          }
          return false;
        }

        function getRandom(number) {
          return Math.floor(Math.random() * number);
        }

        function shuffleArray (array) {
          for (let i = 0; i < 50; i ++) {
            const position1 = getRandom(array.length);
            const position2 = getRandom(array.length);

            const temp = array[position1];
            array[position1] = array[position2];
            array[position2] = temp;
          }
        }

        //for each word in the source array we test where it can fit on the board and then test those locations for validity against other already placed words
        function generateBoard(seed = 0) {
          let bestScoreIndex = 0;
          let top = 0;
          let topScore = 0;
          let fitScore = 0;
          let startTime;

          activeWordList = [];
          coordList = [];
          acrossCount = downCount = 0;

          for (let x = 0; x < rows; x++) {
            for (let y = 0; y < cols; y++) {
              grid[x][y] = {};
              grid[x][y].targetChar = EMPTYCHAR; //target character, hidden
              grid[x][y].indexDisplay = ''; //used to display index number of word start
              grid[x][y].value = '-'; //actual current letter shown on board
            }
          }

          const randomX = Math.floor(Math.random() * (GRID_HEIGHT - 1));
          const randomY = Math.floor(Math.random() * GRID_WIDTH);

          //manually place the longest word horizontally at 0,0, try others if the generated board is too weak
          placeWord(wordArray[seed].word, wordArray[seed].clue, randomX, randomY, false, wordArray[seed].index);

          //attempt to fill the rest of the board
          for (let iy = 0; iy < 2; iy++) { //usually 2 times is enough for max fill potential
            for (let ix = 1; ix < wordArray.length; ix++) {
              if (!isActiveWord(wordArray[ix].word)) { //only add if not already in the active word list
                topScore = 0;
                bestScoreIndex = 0;

                suggestCoords(wordArray[ix].word); //fills coordList and coordCount
                shuffleArray(coordList); //adds some randomization

                if (coordList[0]) {
                  for (let c = 0; c < coordList.length; c++) { //get the best fit score from the list of possible valid coordinates
                    fitScore = checkFitScore(wordArray[ix].word, coordList[c].x, coordList[c].y, coordList[c].vertical);
                    if (fitScore > topScore) {
                      topScore = fitScore;
                      bestScoreIndex = c;
                    }
                  }
                }

                if (topScore > 1) { //only place a word if it has a fitscore of 2 or higher
                  placeWord(wordArray[ix].word, wordArray[ix].clue, coordList[bestScoreIndex].x, coordList[bestScoreIndex].y, coordList[bestScoreIndex].vertical, wordArray[ix].index);
                }
              }
            }
          }
          if(activeWordList.length !== wordArray.length) { //regenerate board if if less than half the words were placed
            generateBoard();
          }
        }

        generateBoard();

        return {
          grid,
          activeWordList
        };
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
          top: 0;
          left: 0;
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
    display: flex;
    width: 700px;
    margin: 30px auto;
    justify-content: space-between;

    .questions {
      max-width: 50%;

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
