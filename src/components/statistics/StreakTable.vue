<template>
  <div>
    <template v-if='data.length'>
      <q-table
        grid
        :title='$t("general.statistics.streak-table.title")'
        dense
        :data='data'
        :columns='columns'
        row-key='name'
      />
    </template>
    <template v-else>
      <q-item>
        <q-item-section>
          {{ $t('general.statistics.streak-table.empty') }}
        </q-item-section>
      </q-item>
    </template>
  </div>
</template>

<script lang='ts'>
import { Vue, Component, VModel } from 'vue-property-decorator'
import { formatEnum } from 'src/utility/helper'
import { CounterType } from 'src/core/entities/counter'
import { History } from 'src/core/entities'
import { CounterService } from 'src/core/services/CounterService'
import { cloneDeep } from 'lodash'
import { InMemory } from 'src/utility/database/proxy/InMemory'

type Option = {
  counter: string,
  type: string,
  createdAt: string,
  maxStreak: number,
  curStreak: number,
};

@Component
export default class StreakTable extends Vue {
  @VModel({ type: Array, required: true }) public history!: Array<History>
  public columns = [
    {
      name: 'Counter',
      align: 'left',
      label: this.$t('general.statistics.streak-table.columns.counter'),
      field: 'counter',
      sortable: true
    },
    {
      name: 'Type',
      align: 'center',
      label: this.$t('general.statistics.streak-table.columns.type'),
      field: 'type',
      sortable: true
    },
    {
      name: 'Created Date',
      align: 'center',
      label: this.$t('general.statistics.streak-table.columns.created'),
      field: 'createdAt',
      sortable: true
    },
    {
      name: 'MaxStreak',
      align: 'center',
      label: this.$t('general.statistics.streak-table.columns.max'),
      field: 'maxStreak',
      sortable: true
    },
    {
      name: 'CurStreak',
      align: 'center',
      label: this.$t('general.statistics.streak-table.columns.current'),
      field: 'curStreak',
      sortable: true
    }
  ]
  public data: Array<Option> = []

  constructor() {
    super()

    this.$q.loading.show()
  }

  async mounted() {
    const streaks: Record<string, {
      maxStreak: number,
      curStreak: number,
      prev: History,
      first: History,
    }> = {}
    const history = cloneDeep(this.history).sort((a, b) => a.date.getTime() > b.date.getTime() ? 1 : -1)

    for (const row of history) {
      if (!streaks[row.counter_id]) {
        streaks[row.counter_id] = {
          maxStreak: 1,
          curStreak: 1,
          prev: row,
          first: row
        }
      }

      if (
        this.$container.resolve(CounterService).isDue(row.getCounter(), row.date) &&
        this.$container.resolve(CounterService).isSucceed(row.getCounter())
      ) {
        streaks[row.counter_id].curStreak++
      } else {
        streaks[row.counter_id].curStreak = 1
        streaks[row.counter_id].prev = row
        streaks[row.counter_id].first = row
      }

      if (streaks[row.counter_id].curStreak > streaks[row.counter_id].maxStreak) {
        streaks[row.counter_id].maxStreak = streaks[row.counter_id].curStreak
      }

      streaks[row.counter_id].prev = row

      row.date.setTime(row.date.getTime() + (86400 * 1000))
    }

    for (const id in streaks) {
      const counter = await this.$orm.repository.counter.setProxy(new InMemory(id)).find(id)

      if (!counter) {
        continue
      }

      this.data.push({
        counter: counter.name,
        type: formatEnum(CounterType)[counter.type],
        createdAt: counter.createdAt.toDateString(),
        maxStreak: streaks[id].maxStreak - 1,
        curStreak: streaks[id].curStreak - 1
      })
    }

    this.$q.loading.hide()
  }
};
</script>
