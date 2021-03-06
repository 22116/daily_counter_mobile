<template>
  <q-page v-if='loaded' class="justify-evenly">
    <history-calendar
      id='calendar'
      :history='history'
      :white-list-hashes='whitelist'
      class="row q-gutter-md q-pa-lg"
      @navigation='fetchMonth'
    />

    <div class="row q-gutter-md q-pa-lg">
      <q-badge color="red">{{ $t('pages.history.mark.red') }}</q-badge>
      <q-badge color="green">{{ $t('pages.history.mark.green') }}</q-badge>
    </div>

    <div class="row q-gutter-md q-pa-lg text-grey">
      {{ $t('pages.history.help') }}
    </div>

    <q-page-sticky v-if='history.length' position="top-right" :offset="[5, 5]">
      <q-btn fab icon="more_vert" color="primary" @click='() => showList = !showList' />
    </q-page-sticky>

    <whitelist v-model='showList' :counters='counters' @update='(hashes) => this.whitelist = hashes' />
  </q-page>
</template>

<script lang="ts">
import { Vue, Component } from 'vue-property-decorator'
import HistoryCalendar from 'components/statistics/HistoryCalendar.vue'
import Whitelist from 'components/modals/counter/Whitelist.vue'
import { Counter, History } from 'src/core/entities'
import { Score } from 'src/core/entities/Counter'
import { InMemory } from 'src/utility/database/proxy/InMemory'
import { generateMonthlyCacheKey } from 'src/utility/helper/string'

@Component({
  components: { Whitelist, HistoryCalendar },
})
export default class PageHistory extends Vue {
  public history: Array<History> = []
  public counters: Array<Counter<Score>> = []
  public showList = false
  public whitelist = []
  public loaded = false

  constructor() {
    super()

    this.$q.loading.show()
  }

  async mounted() {
    this.history = await this.$orm.repository.history
      .setProxy(new InMemory(generateMonthlyCacheKey(new Date())))
      .findByMonth(new Date())
    this.counters = await this.$orm.repository.counter
      .setProxy(new InMemory())
      .findAll()

    this.loaded = true

    this.$q.loading.hide()
  }

  async fetchMonth(date: { year: number, month: number }) {
    this.history = await this.$orm.repository.history
      .setProxy(new InMemory(JSON.stringify(date)))
      .findByMonth(new Date(date.year, date.month))
  }
};
</script>

<style lang='sass'>
.q-date__header
  display: none

#calendar
  margin-top: 0 !important
</style>
