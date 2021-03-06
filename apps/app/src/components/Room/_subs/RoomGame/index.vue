<template>
  <div class="room-game">
    <div class="room-game__circle">
      <div
        class="room-game__message"
      >
        <template
          v-if="getCurrentRoom.state === 'created'"
        >
          <div
            class="room-game__message__title"
          >
            Waiting for players...
          </div>
        </template>
        <template
          v-if="getCurrentRoom.state === 'started'"
        >
          <div
            class="room-game__message__title"
          >
            Find a word with "{{ getCurrentRoom.word_to_guess }}"
          </div>
        </template>
        <template
          v-if="getCurrentRoom.state === 'starting'"
        >
          Game starts in {{ waitingTimer }}...
        </template>
        <template
          v-if="getCurrentRoom.state === 'finished'"
        >
          {{ winner.name }} won
        </template>

        <button
          v-if="(getCurrentRoom.state === 'starting' || getCurrentRoom.state === 'created') && !hasJoinedGame"
          type="button"
          class="btn btn-primary"
          :disabled="$wait.is('joining game')"
          @click="joinGame"
        >
          Join game
        </button>
      </div>

      <div class="room-game__circle__players">
        <RoomGamePlayer
          v-for="player in players"
          :key="player.id"
          :index="player.id"
          :name="player.name"
          :hearts="player.heart"
          :typing="player.typing"
          :style="{
            left: player.position.left,
            top: player.position.top
          }"
          :class="{
            'room-game-player--is-self': isPlaying && player.id === $socket.id
          }"
        />
      </div>

      <div
        class="room-game__circle__center"
      >
        <div
          class="room-game__circle__center__inner"
          :style="{
            'transform': `scale(${dotScale})`
          }"
        ></div>
      </div>
      <div
        v-if="getCurrentRoom.state === 'started'"
        class="room-game__circle__arrow"
        :style="{
          'transform': `rotate(${arrowRotation}deg)`
        }"
      />
    </div>
    <div
      v-if="getCurrentRoom.state === 'started' && isPlaying"
      class="room-game__form"
    >
      <form
        @submit.prevent="guessWord"
      >
        <label for="word">
          Your word
        </label>
        <input
          ref="guess-word"
          v-model="guessingWord"
          type="text"
          name="word"
          id="word"
          class="field"
          @input="typeWord"
        >
      </form>
    </div>

    <ul
      v-if="getCurrentRoom.state === 'started'"
      class="room-game__letters-to-use"
    >
      <li
        v-for="letter in computedLetters"
        :key="letter.letter"
        :class="{
          'room-game__letters-to-use--used': letter.isUsed
        }"
      >
        {{ letter.letter }}
      </li>
    </ul>
  </div>
</template>

<script>
  import { mapGetters } from 'vuex'

  import RoomGamePlayer from './_subs/RoomGamePlayer'

  const LETTERS_TO_USE = ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'L', 'M', 'N', 'O', 'P', 'R', 'S', 'T', 'U', 'V']

  /**
   * @module component - RoomGame
   */
  export default {
    name: 'RoomGame',
    components: {
      RoomGamePlayer
    },
    data () {
      return {
        guessingWord: null,
        typingWord: null,
        waitingTimer: null,
        totalRotations: 0,
        dotScale: 0
      }
    },
    watch: {
      async isPlaying (v) {
        if (v) {
          await this.$nextTick()
          if (this.$refs['guess-word']) {
            this.$refs['guess-word'].focus()
          }
        }
      }
    },
    mounted () {
      this.$socket.on('room_starting', ({ game_start_at, state }) => {
        const room = this.$store.getters.getCurrentRoom
        this.$store.commit('SET_ROOM', {
          ...room,
          state
        })
        this.waitingTimer = game_start_at - Date.now()
        const interval = setInterval(() => {
          this.waitingTimer -= 1000
        }, 1000)

        setTimeout(() => {
          this.waitingTimer = null
          clearInterval(interval)
        }, this.waitingTimer)
      })
      this.$socket.on('room_next_player', () => {
        this.typingWord = null
        this.guessingWord = null
        this.totalRotations += 0.5
      })
      this.$socket.on('room_type_guess_word', ({ word }) => {
        this.typingWord = word
      })
      this.$socket.on('room_tick', ({ timer }) => {
        const percent = ((timer - Date.now()) * 100) / 5000
        this.dotScale = Math.ceil(100 - percent) / 100
      })
    },
    computed: {
      ...mapGetters(['getCurrentRoom']),
      winner () {
        const { activePlayers } = this.getCurrentRoom
        return activePlayers.filter(player => player.heart > 0)[0]
      },
      arrowRotation () {
        const { currentPlayer } = this.getCurrentRoom
        const playerIndex = this.players.findIndex(player => player.id === currentPlayer.id)

        return 180 + ((360 / this.players.length) * playerIndex)
      },
      computedLetters () {
        const { activePlayers } = this.getCurrentRoom
        const playerIndex = activePlayers.findIndex(player => player.id === this.$socket.id)
        
        if (playerIndex === -1) {
          return []
        }

        return LETTERS_TO_USE
          .map(letter => ({
            isUsed: !activePlayers[playerIndex].lettersToUse.includes(letter),
            letter
          }))
      },
      hasJoinedGame () {
        const { activePlayers } = this.getCurrentRoom
        return activePlayers.map(v => v.id).includes(this.$socket.id)
      },
      isPlaying () {
        const { currentPlayer } = this.getCurrentRoom
        return currentPlayer && this.$socket.id === currentPlayer.id
      },
      players () {
        const { activePlayers, currentPlayer } = this.getCurrentRoom
        return activePlayers.map((player, k) => {
          const angle = (Math.PI * 2) / activePlayers.length

          return {
            ...player,
            typing: currentPlayer && currentPlayer.id === player.id ? this.typingWord : null,
            position: {
              left: `${((Math.cos(k * angle)).toFixed(2) * 250) - 70}px`,
              top: `${((Math.sin(k * angle)).toFixed(2) * 250) - 33}px`
            }
          }
        })
      }
    },
    methods: {
      joinGame () {
        this.$wait.start('joining game')
        this.$socket.emit('room_join_game', {
          id: this.getCurrentRoom.id,
        }, () => {
          this.$wait.end('joining game')
        })
      },
      typeWord () {
        this.$socket.emit('type_guess_word', {
          room: this.getCurrentRoom.id,
          word: this.guessingWord
        })
      },
      guessWord () {
        this.$socket.emit('guess_word', {
          room: this.getCurrentRoom.id,
          word: this.guessingWord
        })
      }
    }
  }
</script>

<style lang="css" scoped>
  .room-game {
    display: flex;
    flex-direction: column;
  }

  .room-game__circle {
    position: relative;
    width: 500px;
    height: 500px;
    border-radius: 500px;
    border: 1px solid #434C5B;
    margin: auto;
    margin-bottom: 64px;
  }

  .room-game__circle__center {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    width: 54px;
    height: 54px;
    border-radius: 54px;
    margin: auto;
    border: 1px solid #434C5B;
  }

  .room-game__circle__center__inner {
    position: absolute;
    content: '';
    left: 0px;
    top: 0px;
    width: 52px;
    height: 52px;
    border-radius: 54px;
    background: #434C5B;
    transition: transform 200ms;
    transform: scale(1);
  }


  .room-game__circle__arrow {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    margin: auto;
    width: 500px;
    height: 28px;
    transition: transform 500ms ease-in-out;
  }

  .room-game__circle__arrow::before {
    content: '';
    position: absolute;
    width: 0;
    height: 0;
    left: 119px;
    border-style: solid;
    border-width: 14px 16px 14px 0;
    border-color: transparent #434C5B transparent transparent;
  }

  .room-game__circle__arrow::after {
    content: '';
    position: absolute;
    left: 135px;
    top: 9px;
    width: 120px;
    height: 10px;
    border-radius: 0 9px 9px 0;
    background-color: #434C5B;
  }

  .room-game__message {
    position: absolute;
    display: flex;
    flex-direction: column;
    text-align: center;
    bottom: 120px;
    left: 0;
    right: 0;
    width: 200px;
    margin: auto;
    z-index: 10;
  }

  .room-game__message__title {
    margin-bottom: 16px;
  }

  .room-game-player {
    position: absolute;
    z-index: 2;
  }

  .room-game__circle__players {
    position: absolute;
    left: 250px;
    top: 250px;
  }

  .room-game__word-to-guess {
    position: absolute;
  }

  .room-game__form form {
    display: flex;
    flex-direction: column;
    width: 250px;
    margin: auto;
    margin-bottom: 32px;
  }

  .room-game__form label {
    color: rgba(255, 255, 255, 0.70);
    text-align: center;
    margin-bottom: 8px;
  }

  .room-game__letters-to-use {
    display: flex;
    flex-wrap: wrap;
    margin: 0 auto;
    padding: 0;
    list-style: none;
    width: 500px;
  }

  .room-game__letters-to-use li {
    width: 40px;
    height: 40px;
    border-radius: 4px;
    text-align: center;
    line-height: 40px;
    background-color: #242C3B;
    margin-bottom: 4px;
  }

  .room-game__letters-to-use li:not(:last-child) {
    margin-right: 4px;
  }

  .room-game__letters-to-use li.room-game__letters-to-use--used {
    opacity: 0.2;
  }
</style>
