<template>
  <v-row justify="center" align="center">
    <v-col cols="12" sm="8" md="6">
      <v-card-title> 予測結果 </v-card-title>
      <v-card>
        <p>価格</p>
        <p>
          平均:
          {{ Math.round(predictions[0]?.price / 10000).toLocaleString() }}万円
        </p>
        <p>
          上位10%:
          {{
            Math.round(predictions[0]?.price_90 / 10000).toLocaleString()
          }}万円
        </p>
        <p>
          下位10%:
          {{
            Math.round(predictions[0]?.price_10 / 10000).toLocaleString()
          }}万円
        </p>
        <p>価格トレンド</p>
        <p v-for="(pred, idx) in predictions" :key="idx">
          {{ 2023 - idx }}年
          {{ Math.round(pred?.price / 10000).toLocaleString() }}万円
        </p>
      </v-card>

      <v-card-title> 入力データ </v-card-title>
      <v-card>
        <v-select
          v-model="type"
          label="種類 (必須)"
          :items="metadata.unique_values.type"
        ></v-select>
        <v-autocomplete
          v-model="prefecture"
          label="県 (必須)"
          :items="metadata.unique_values.prefecture"
        ></v-autocomplete>
        <v-autocomplete
          v-model="city"
          label="市区町村 (必須)"
          :items="cityItems[prefecture] || metadata.unique_values.city"
        ></v-autocomplete>
        <v-autocomplete
          v-model="city2"
          label="所在地"
          :items="
            city2Items[prefecture + ':' + city] || metadata.unique_values.city2
          "
        ></v-autocomplete>
        <v-autocomplete
          v-model="nearest_sta"
          label="最寄り駅"
          :items="
            // nearest_staItems[prefecture + ':' + city + ':' + city2] ||
            metadata.unique_values.nearest_sta
          "
        ></v-autocomplete>
        <v-text-field
          v-model="nearest_sta_dist"
          label="最寄り駅距離(分)"
          type="number"
          :max="Math.max(...metadata.unique_values.nearest_sta_dist)"
          :min="Math.min(...metadata.unique_values.nearest_sta_dist)"
        />
        <v-select
          v-model="city_plan"
          label="都市計画"
          :items="['', ...metadata.unique_values.city_plan]"
        ></v-select>
        <v-text-field
          v-model="area"
          label="面積(m^2) (土地面積。中古マンション等の場合は専有部分の床面積。必須)"
          type="number"
          :max="Math.max(...metadata.unique_values.area)"
          :min="Math.min(...metadata.unique_values.area)"
        />
        <v-text-field
          v-if="type == '宅地(土地と建物)'"
          v-model="floor_area"
          label="建物延床面積(m^2)"
          type="number"
          :max="Math.max(...metadata.unique_values.floor_area)"
          :min="Math.min(...metadata.unique_values.floor_area)"
        />
        <!--        <v-select-->
        <!--          label="前面道路:種類"-->
        <!--          :items="metadata.unique_values.front_road_type"-->
        <!--        ></v-select>-->
        <!--        <v-select-->
        <!--          label="前面道路:方位"-->
        <!--          :items="metadata.unique_values.front_road_dir"-->
        <!--        ></v-select>-->
        <v-text-field
          v-model="front_road_width"
          label="前面道路:幅員(m)"
          type="number"
          :max="Math.max(...metadata.unique_values.front_road_width)"
          :min="Math.min(...metadata.unique_values.front_road_width)"
        />
        <!--        <v-select-->
        <!--          label="建物構造"-->
        <!--          :items="metadata.unique_values.structure"-->
        <!--        ></v-select>-->
        <v-text-field
          v-if="type === '宅地(土地と建物)' || type === '中古マンション等'"
          v-model="building_year"
          label="築年 (例: 1990)"
          type="number"
          :max="Math.max(...metadata.unique_values.building_year)"
          :min="Math.min(...metadata.unique_values.building_year)"
        />
      </v-card>
      <v-card-title> 使い方 </v-card-title>
      <v-card>
        <ul>
          <li>値がわからない場合は空白にする</li>
          <li>
            <a href="https://www.land.mlit.go.jp/webland/note.html#syurui"
              >項目について(国土交通省)</a
            >
          </li>
        </ul>
      </v-card>
    </v-col>
  </v-row>
</template>

<script>
import _ from 'lodash'

export default {
  name: 'IndexPage',
  async asyncData({ $axios, $config }) {
    const metadata = await $axios.$get($config.apiBaseUrl + '/metadata')
    return {
      metadata,
      type: '中古マンション等',
      prefecture: '東京都',
      city: '渋谷区',
      city2: '渋谷',
      city_plan: '第１種低層住居専用地域',
      nearest_sta: '渋谷',
      nearest_sta_dist: 3,
      area: 100,
      floor_area: 100,
      front_road_width: 10,
      building_year: 2000,
      predictions: _.map(_.range(10), () => {}),
      timer: null,
    }
  },
  computed: {
    prefecture_city_city2_nearest_sta() {
      const x = _.compact(
        this.metadata.unique_values.prefecture_city_city2_nearest_sta
      )
      return _.map(x, (y) => {
        return y.split(':')
      })
    },
    cityItems() {
      const x = _.groupBy(this.prefecture_city_city2_nearest_sta, 0)
      return _.mapValues(x, (y) => {
        return _.sortedUniq(_.map(y, 1))
      })
    },
    city2Items() {
      const x = _.groupBy(this.prefecture_city_city2_nearest_sta, (y) => {
        return y[0] + ':' + y[1]
      })
      return _.mapValues(x, (y) => {
        return _.sortedUniq(_.map(y, 2))
      })
    },
    nearest_staItems() {
      const x = _.groupBy(this.prefecture_city_city2_nearest_sta, (y) => {
        return y[0] + ':' + y[1] + ':' + y[2]
      })
      return _.mapValues(x, (y) => {
        return _.sortedUniq(_.map(y, 3))
      })
    },
  },
  mounted() {
    this.timer = setInterval(async () => {
      const type = this.type

      const params = _.map(_.range(10), (i) => {
        const x = {
          type: this.type,
          prefecture: this.prefecture,
          city: this.city,
          city2: this.city2,
          city_plan: this.city_plan,
          nearest_sta: this.nearest_sta,
          nearest_sta_dist: this.nearest_sta_dist,
          area: Math.round(this.area),
          floor_area:
            type === '宅地(土地と建物)' ? Math.round(this.floor_area) : '',
          front_road_width: this.front_road_width,
          time: 202300 - 100 * i,
          building_year:
            type === '宅地(土地と建物)' || type === '中古マンション等'
              ? this.building_year
              : '',
        }
        for (const k in x) {
          if (!x[k]) {
            delete x[k]
          }
        }
        return x
      })

      try {
        const result = await this.$axios.$post(
          this.$config.apiBaseUrl + '/predict',
          params
        )

        for (let i = 0; i < 10; i++) {
          this.$set(this.predictions, i, result[i])
        }
      } catch (err) {
        console.error(err)
        for (let i = 0; i < 10; i++) {
          this.$set(this.predictions, i, {})
        }
      }
    }, 1000)
  },
  beforeDestroy() {
    clearInterval(this.timer)
  },
}
</script>
