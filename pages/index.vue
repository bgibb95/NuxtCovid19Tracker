<template>
  <div class="page-container chart-page">
    <v-layout column justify-center align-center>
      <v-row>
        <!-- <v-card-title class="justify-center">COVID-19 Tracker</v-card-title> -->
        <transition name="fade">
          <v-autocomplete
            v-if="affectedCountries.length > 0"
            v-model="selectedCountry"
            :append-icon="mdiMenuDown"
            :items="affectedCountries"
            shaped
            dense
            :menu-props="{ maxHeight: 170 }"
            filled
            label="Search country"
          ></v-autocomplete>
        </transition>
      </v-row>

      <transition name="fade">
        <h5 v-if="percentageChange" class="my-3 percent-change-title" :class="{ hideBg: historyByCountryLoading }">
          <span v-if="!historyByCountryLoading">
            <span :class="percentClass">
              <b>{{ percentageChange }}%</b>
            </span>
            over previous day
          </span>
          <span v-if="historyByCountryLoading">&nbsp;</span>
        </h5>
      </transition>

      <h4 class="active-cases" :class="{ hideBg: !activeCases.length > 0 && !historyByCountryLoading }">
        <span v-if="activeCases.length > 0 && dates.length > 0 && !historyByCountryLoading">
          Active cases in {{ selectedCountry }}
          <span v-if="selectedActiveCases">
            :
            <span class="blue--text text--lighten-1">
              <b>{{ selectedActiveCases }}</b>
            </span>
            on
            <span>{{ selectedDate }}</span>
          </span>
        </span>
        <span v-if="historyByCountryLoading">Updating...</span>
      </h4>

      <div
        v-if="(activeCases.length > 0 && dates.length > 0) || !loading"
        class="chartContainer"
        :class="{ hideBg: historyByCountryLoading }"
      >
        <transition name="fade">
          <div v-if="!activeCases.length > 0 && !dates.length > 0 && !historyByCountryLoading" class="centerVH">
            <!-- <v-icon>{{ mdiAlertCircle }}</v-icon> -->
            <br />
            <h4>
              Sorry, the history data is currently unavailable.
              <br />Try again soon.
              <br />
              <br />
              <nuxt-link to="/stats">View stats instead</nuxt-link>
            </h4>
          </div>
        </transition>
        <transition name="fade">
          <v-progress-circular
            v-if="historyByCountryLoading"
            class="center-loader"
            :size="45"
            :width="5"
            color="white"
            indeterminate
          ></v-progress-circular>
        </transition>
        <transition name="fade">
          <TrendChart
            v-if="activeCases.length > 0 && dates.length > 0"
            class="chart"
            :class="{ zeroOpacity: historyByCountryLoading }"
            :interactive="true"
            :datasets="[dataset]"
            :grid="{
              verticalLines: true,
              horizontalLines: true
            }"
            :min="0"
            :labels="labels"
            @mouse-move="onMouseMove"
          ></TrendChart>
        </transition>
      </div>
    </v-layout>
  </div>
</template>

<script>
import { mdiMenuDown } from '@mdi/js'
import { mapState } from 'vuex'

export default {
  components: {},
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  transition(to, from) {
    if (!from) {
      return 'slide-left'
    }
    // if (to.name === 'stats' || to.name === 'learn') {
    //   return 'slide-right'
    // }
    return 'slide-left'
  },
  data() {
    return {
      mdiMenuDown,
      active_cases: [],
      selectedActiveCases: 0,
      selectedDate: 0,
      loading: false
    }
  },
  watch: {
    selectedCountry() {
      if (this.selectedCountry) {
        document.querySelector('input').blur()
      }
      this.loading = true
      this.$store.dispatch('fetchCasesByCountry').finally(() => (this.loading = false))
      this.$store.dispatch('fetchLatestStatByCountry')
    }
  },
  mounted() {
    const element = document.querySelector('.chart-page')
    // eslint-disable-next-line no-new, no-undef
    new ResizeSensor(element, () => {
      this.$store.commit('setPageHeight', element.offsetHeight + 15)
    })
  },
  methods: {
    numberWithSpaces(n) {
      return n.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ' ')
    },
    onMouseMove(params) {
      if (!params) {
        this.selectedActiveCases = null
        return
      }
      this.selectedActiveCases = this.numberWithSpaces(params.data[0])
      this.selectedDate = this.historyByCountry.find((history) => Number(history.Active) === params.data[0]).Date
      // this.selectedDate = this.historyByCountry.find(
      //   (history) => Number(history.active_cases) === params.data[0]
      // ).record_date
    }
  },
  computed: {
    historyByCountry() {
      return this.showAllHistory ? this.historyByCountryFull : this.historyByCountryRecent
    },
    percentClass() {
      return this.percentageChange > 0 ? 'red--text text--lighten-2' : 'green--text text--lighten-2'
    },
    selectedCountry: {
      get() {
        return this.$store.state.selectedCountry
      },
      set(value) {
        this.$store.commit('setSelectedCountry', value)
      }
    },
    labels() {
      return {
        xLabels: this.dates || ['11/11', '11/11', '11/11', '11/11', '11/11'],
        yLabels: 10,
        yLabelsTextFormatter: (val) => Math.ceil(val)
      }
    },
    dates() {
      if (this.historyByCountry.length > 6) {
        return [this.historyByCountry[0].Date, this.historyByCountry[this.historyByCountry.length - 1].Date]
      }
      return this.historyByCountry.map((history) => history.Date)
    },
    activeCases() {
      return this.historyByCountry.map((history) => Number(history.Active))
    },
    five() {
      return this.activeCases.filter((a, b) => this.activeCases.indexOf(a) === b)
    },
    dataset() {
      if (!this.activeCases.length > 0) {
        return {
          data: [1000, 2000, 3000, 4000, 5000],
          showPoints: true,
          fill: true,
          className: 'active-cases'
        }
      }
      return {
        data: this.activeCases,
        showPoints: true,
        fill: true,
        className: 'active-cases'
      }
    },
    ...mapState({
      historyByCountryRecent: (state) => state.historyByCountry,
      historyByCountryFull: (state) => state.historyByCountryFull,
      historyByCountryLoading: (state) => state.historyByCountryLoading,
      affectedCountries: (state) => state.affectedCountries.map((country) => (country ? country.name : null)),
      percentageChange: (state) => state.percentageChange,
      showAllHistory: (state) => state.showAllHistory
    })
  }
}
</script>

<style lang="scss" scoped>
h4,
h5 {
  background-color: rgba(#2e2e2e, 1);
  //border: 1px solid rgba(white, 0.7);
  border-radius: 8px;
  padding: 8px;
  font-weight: normal;
}
.hideBg {
  border: 0px solid transparent !important;
  background: none !important;
}
.centerVH {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  text-align: center;

  @media screen and (max-width: 780px) {
    width: 90%;
  }
}
.percent-change-title {
  margin-bottom: 0 !important;
}
.v-card__title {
  //font-size: 1rem;
  @media screen and (max-width: 780px) {
    padding-top: 0 !important;
  }
}
.active-cases {
  margin-top: 10px;
  margin-bottom: 10px;
  text-align: center;
}
.center-loader {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
.chartContainer {
  position: relative;
  width: 100%;
  height: 58vh;
  border: 0px solid rgba(white, 0.7);
  background-color: rgba(#2e2e2e, 1);
  padding: 2%;
  border-radius: 8px;
  transition: background-color 0.4s ease, border 0.4s ease;
  @media screen and (max-width: 780px) {
    height: 54vh;
  }
}
.chart {
  opacity: 1;
  transition: all 0.4s ease;
}
.zeroOpacity {
  opacity: 0;
}
</style>
