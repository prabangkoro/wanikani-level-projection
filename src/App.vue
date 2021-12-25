<template>
  <v-app>
    <v-main>
      <v-container>
        <div
          class="text-h4 text-center mt-4 mb-6"
        >
          Wanikani Level Projection
        </div>

        <v-expansion-panels
          v-model="panels"
          multiple
        >
          <v-expansion-panel>
            <v-expansion-panel-header>
              <div class="text-h6">
                Configuration
              </div>
            </v-expansion-panel-header>
            <v-expansion-panel-content>
              <v-form ref="form">
                <v-text-field
                  v-model="apiKey"
                  class="mt-2"
                  label="API Token v2"
                  outlined
                  clearable
                  dense
                ></v-text-field>

                <div
                  class="text-h6 mb-2"
                >
                  Normal Level
                </div>
                <div class="subtitle-2 mb-2">
                  Fastest: 6d 20h
                </div>
                <div class="d-flex">
                  <v-text-field
                    v-model="normalLevelDay"
                    :rules="[rules.required, rules.number]"
                    class="pr-2"
                    label="Days"
                    outlined
                    clearable
                    dense
                  ></v-text-field>
                  <v-text-field
                    v-model="normalLevelHour"
                    :rules="[rules.required, rules.number]"
                    class="pl-2"
                    label="Hours"
                    outlined
                    clearable
                    dense
                  ></v-text-field>
                </div>

                <div
                  class="text-h6 mb-2"
                >
                  Fast Level
                </div>
                <div class="subtitle-2">
                  Fastest: 3d 10h 
                </div>
                <div class="subtitle-2 mb-2">
                  Levels: {{ FAST_LEVEL.join(', ') }}
                </div>
                <div class="d-flex">
                  <v-text-field
                    v-model="fastLevelDay"
                    :rules="[rules.required, rules.number]"
                    class="pr-2"
                    label="Days"
                    outlined
                    clearable
                    dense
                  ></v-text-field>
                  <v-text-field
                    v-model="fastLevelHour"
                    :rules="[rules.required, rules.number]"
                    class="pl-2"
                    label="Hours"
                    outlined
                    clearable
                    dense
                  ></v-text-field>
                </div>

                <div class="d-flex">
                  <v-btn
                    class="flex-grow-1 flex-shrink-0 mr-2"
                    color="primary"
                    :disabled="!enableLoad || isLoading"
                    :loading="isLoading"
                    @click="submit()"
                  >
                    Load
                  </v-btn>

                  <v-btn
                    class="flex-grow-0 flex-shrink-0"
                    color="error"
                    :disabled="!enableLoad || isLoading"
                    @click="reset()"
                  >
                    Reset
                  </v-btn>
                </div>
                <div
                  v-if="errorMessage"
                  class="mt-2 text-center red--text"
                >
                  {{ errorMessage }}
                </div>
              </v-form>
            </v-expansion-panel-content>
          </v-expansion-panel>
        </v-expansion-panels>

        <v-simple-table
          v-if="forecast.levelProgressions.length !== 0"
          class="mt-12"
          fixed-header
          height="70vh"
        >
          <template v-slot:default>
            <thead>
              <tr>
                <th class="text-left subtitle-1 font-weight-medium">
                  Level
                </th>
                <th class="text-left subtitle-1 font-weight-medium">
                  Projection
                </th>
                <th class="text-left subtitle-1 font-weight-medium">
                  Time
                </th>
                <th class="text-left subtitle-1 font-weight-medium">
                  Fastest
                </th>
                <th class="text-left subtitle-1 font-weight-medium">
                  Time
                </th>
                <th class="text-left subtitle-1 font-weight-medium">
                  Median
                </th>
                <th class="text-left subtitle-1 font-weight-medium">
                  Time
                </th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="(item, index) in forecast.levelProgressions"
                :key="index"
                :class="{
                  'font-weight-bold blue lighten-5': item.current,
                }"
              >
                <td>{{ item.level }}</td>

                <td>{{ formatDate(item.unlockedAt) }}</td>
                <td>{{ index === forecast.levelProgressions.length-1 ? '-' : formatElapsedTime(item.diff) }}</td>

                <td>{{ formatDate(fastest.levelProgressions[index].unlockedAt) }}</td>
                <td>{{ index === forecast.levelProgressions.length-1 ? '-' : formatElapsedTime(fastest.levelProgressions[index].diff) }}</td>

                <td>{{ formatDate(median.levelProgressions[index].unlockedAt) }}</td>
                <td>{{ index === forecast.levelProgressions.length-1 ? '-' : formatElapsedTime(median.levelProgressions[index].diff) }}</td>
              </tr>
            </tbody>
          </template>
        </v-simple-table>
      </v-container>
    </v-main>

    <v-footer padless>
      <v-col
        class="text-center"
      >
        {{ new Date().getFullYear() }} — <strong>bagusprabangkoro</strong>
      </v-col>
    </v-footer>
  </v-app>
</template>

<script>
import Axios from 'axios'

const MAX_LEVEL = 60
const FAST_LEVEL = [43, 44, 46, 47, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60]

export default {
  name: 'App',
  data () {
    return {
      fastest: {
        normalSpeed: 0,
        fastSpeed: 0,
        levelProgressions: [] // level, unlockedAt, passedAt, diff, current
      },
      forecast: {
        normalSpeed: 0,
        fastSpeed: 0,
        levelProgressions: []
      },
      median: {
        normalSpeed: 0,
        fastSpeed: 0,
        levelProgressions: []
      },
      medianSpeed: 0,
      isLoading: false,
      panels: [0],
      FAST_LEVEL,
      apiKey: '',
      normalLevelDay: '',
      normalLevelHour: '',
      fastLevelDay: '',
      fastLevelHour: '',
      errorMessage: '',
      rules: {
        required: value => !!value || 'Required.',
        number: value => !isNaN(value) || 'Must be a number.',
      },
    }
  },

  computed: {
    apiInstance () {
      return Axios.create({
        baseURL: 'https://api.wanikani.com/v2/',
        headers: { Authorization: `Bearer ${this.apiKey}` }
      })
    },
    enableLoad () {
      return [
        this.apiKey,
        this.normalLevelDay, this.normalLevelHour,
        this.fastLevelDay, this.fastLevelHour
      ].every(i => !!i)
    }
  },

  methods: {
    initializeSpeed () {
      this.fastest.normalSpeed = new Date(this.getMillisecondFromDayHour(6, 20))
      this.fastest.fastSpeed = new Date(this.getMillisecondFromDayHour(3, 10))

      // will get from user input
      this.forecast.normalSpeed = new Date(this.getMillisecondFromDayHour(this.normalLevelDay, this.normalLevelHour))
      this.forecast.fastSpeed = new Date(this.getMillisecondFromDayHour(this.fastLevelDay, this.fastLevelHour))

      // will calculate median after get response
      this.median.normalSpeed = new Date(0)
      this.median.fastSpeed = new Date(0)
    },
    submit () {
      this.errorMessage = ''

      this.initializeSpeed()
      if (!this.validateNormalLevel()) {
        this.errorMessage = 'Invalid Normal Level Speed'
        return
      }
      if (!this.validateFastLevel()) {
        this.errorMessage = 'Invalid Fast Level Speed'
        return
      }
      
      this.isLoading = true
      this.saveDataToLocalStorage()

      this.apiInstance.get('level_progressions').then(response => {
        this.initLevelProgress(response.data.data.map(i => i.data))
        this.initMedian()

        this.calculateAllProjection()
        
        this.isLoading = false
        this.panels = []
      })
      .catch(e => {
        this.errorMessage = e.message
        this.isLoading = false
      })
    },
    calculateAllProjection () {
      this.fastest.levelProgressions = this.calculateProjection(this.fastest)
      this.forecast.levelProgressions = this.calculateProjection(this.forecast)
      this.median.levelProgressions = this.calculateProjection(this.median)
    },
    initLevelProgress (data) {
      const initProgress = data.map(i => {
        const unlockedAt = new Date(i.created_at)
        const passedAt = new Date(i.passed_at ? i.passed_at : i.abandoned_at)
        const diff = new Date(passedAt.getTime() === 0 ? 0 : Math.abs(unlockedAt.getTime() - passedAt.getTime()))

        return {
          level: i.level,
          unlockedAt,
          passedAt,
          diff,
          current: false
        }
      })

      this.fastest.levelProgressions = initProgress.slice()
      this.forecast.levelProgressions = initProgress.slice()
      this.median.levelProgressions = initProgress.slice()
    },
    calculateProjection (data) {
      var levelProgressions = data.levelProgressions
      const currentLevelProgression = levelProgressions[levelProgressions.length - 1]
      if (currentLevelProgression.diff.getTime() !== 0) return levelProgressions

      let actualSpeed = FAST_LEVEL.includes(currentLevelProgression.level) ? data.fastSpeed : data.normalSpeed
      let passedAt = new Date(this.modHour(currentLevelProgression.unlockedAt.getTime() + actualSpeed.getTime()))
      const now = new Date()
      if (passedAt < now) {
        actualSpeed = new Date(this.modHour(now - currentLevelProgression.unlockedAt.getTime()))
        passedAt = new Date(this.modHour(now))
      }

      // current level forecast
      levelProgressions[levelProgressions.length - 1] = {
        ...levelProgressions[levelProgressions.length - 1],
        passedAt,
        diff: actualSpeed,
        current: true
      }

      // after current level forecast
      // forecast speed
      let speed = FAST_LEVEL.includes(currentLevelProgression.level) ? data.fastSpeed : data.normalSpeed
      for (var i = levelProgressions.length; levelProgressions[i - 1].level < MAX_LEVEL; i++) {
        const level = levelProgressions[levelProgressions.length - 1].level + 1
        const unlockedAt = levelProgressions[i - 1].passedAt
        const speed = FAST_LEVEL.includes(level) ? data.fastSpeed : data.normalSpeed 
        const passedAt = new Date(unlockedAt.getTime() + speed.getTime())

        levelProgressions.push({
          level,
          unlockedAt,
          passedAt,
          diff: speed,
          current: false
        })
      }

      return levelProgressions
    },
    getDataFromLocalStorage () {
      if (!localStorage.getItem('wanikani-level-progression')) return
      try {
        const result = JSON.parse(localStorage.getItem('wanikani-level-progression'))
        this.apiKey = result.apiKey
        this.normalLevelDay = result.normalLevelDay
        this.normalLevelHour = result.normalLevelHour
        this.fastLevelDay = result.fastLevelDay
        this.fastLevelHour = result.fastLevelHour
      } catch(e) {
        localStorage.removeItem('wanikani-level-progression')
      }
    },
    saveDataToLocalStorage () {
      const {
        apiKey, normalLevelDay, normalLevelHour, fastLevelDay, fastLevelHour
      } = this

      localStorage.setItem('wanikani-level-progression', JSON.stringify({
        apiKey, normalLevelDay, normalLevelHour, fastLevelDay, fastLevelHour
      }))
    },
    reset() {
      this.$refs.form.reset()
      localStorage.removeItem('wanikani-level-progression')
    },
    initMedian () {
      var times = this.median.levelProgressions
        .filter(i => i.diff.getTime() !== 0)
        .map(i => i.diff.getTime())

      this.medianSpeed = new Date(Math.max(this.getMedian(times), this.getMillisecondFromDayHour(6, 20)))

      this.median.normalSpeed = this.medianSpeed
      this.median.fastSpeed = this.medianSpeed
    },

    getMedian (times) {
      if (times.length === 0) return 0

      times.sort((a, b) => a - b)

      var half = Math.floor(times.length / 2);

      if (times.length % 2)
        return times[half];

      return (times[half - 1] + times[half]) / 2.0;
    },
    validateNormalLevel () {
      return this.validateSpeed(this.normalLevelDay, this.normalLevelHour, this.fastest.normalSpeed.getTime())
    },
    validateFastLevel () {
      return this.validateSpeed(this.fastLevelDay, this.fastLevelHour, this.fastest.fastSpeed.getTime())
    },
    validateSpeed (day, hour, speedMin) {
      const speed = day * (1000 * 60 * 60 * 24) + hour * (1000 * 60 * 60)

      return speed >= speedMin
    },
    modHour (milli) {
      const mod = milli % (1000 * 60 * 60)
      return milli - mod
    },
    formatDate (rawDate) {
      var date = new Date(rawDate)
      return date.toLocaleString('en-US', {
        year: '2-digit', month: 'short', day: 'numeric',
        hour: 'numeric', minute: '2-digit'
      })
    },
    formatElapsedTime (time) {
      var diffHours = Math.round(time.getTime() % (1000 * 60 * 60 * 24) / (1000 * 60 * 60))
      var diffDays = Math.floor(time.getTime() / (1000 * 60 * 60 * 24))
      if (diffHours === 24) {
        diffHours = 0
        diffDays++
      }

      return `${diffDays}d ${diffHours}h`
    },
    getMillisecondFromDayHour (day, hour) {
      return day * (1000 * 60 * 60 * 24) + hour * (1000 * 60 * 60)
    }
  },
  mounted () {
    this.getDataFromLocalStorage()
  }
}
</script>
