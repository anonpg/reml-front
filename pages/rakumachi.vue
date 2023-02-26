<template>
  <v-row justify="center">
    <v-col cols="12" sm="12" md="12">
      <v-card-title> 物件一覧 </v-card-title>
      <v-card>
        <v-switch v-model="tableDebugMode" label="Debug Mode"></v-switch>
        <v-text-field
          v-model="search"
          append-icon="mdi-magnify"
          label="Search"
          single-line
          hide-details
        ></v-text-field>
        <v-data-table :headers="headers" :items="rakumachis" :search="search">
          <template #[`item.name`]="{ item }">
            <a
              :href="'https://www.rakumachi.jp' + item.path"
              target="_blank"
              rel="noopener noreferrer"
              >{{ item.name }}</a
            >
          </template>
          <template #[`item.price_disp`]="{ item }">
            {{ Math.round(item.price_disp / 10000) }}万円
          </template>
          <template #[`item.price_predict_disp`]="{ item }">
            {{ Math.round(item.price_predict_disp / 10000) }}万円
          </template>
          <template #[`item.price_predict_old_disp`]="{ item }">
            {{ Math.round(item.price_predict_old_disp / 10000) }}万円
          </template>
          <template #[`item.price_relative_disp`]="{ item }">
            {{ Math.round(item.price_relative_disp * 100) }}%
          </template>
          <template #[`item.yield_processed`]="{ item }">
            {{ Math.round(item.yield_processed * 100 * 10) / 10 }}%
          </template>
        </v-data-table>
      </v-card>
    </v-col>
  </v-row>
</template>

<script>
import _ from 'lodash'

export default {
  async asyncData({ $axios, $config }) {
    const metadata = await $axios.$get($config.apiBaseUrl + '/metadata')
    let rakumachis = await $axios.$get($config.rakumachisUrl)

    rakumachis = _.filter(rakumachis, (x) => {
      return x.address.includes('東京都') || x.address.includes('神奈川県')
    })

    let prefecture_city_city2_nearest_sta = []
    {
      const x = _.compact(
        metadata.unique_values.prefecture_city_city2_nearest_sta
      )
      prefecture_city_city2_nearest_sta = _.map(x, (y) => {
        return y.split(':')
      })
    }

    let cityItems = {}
    {
      const x = _.groupBy(prefecture_city_city2_nearest_sta, 0)
      cityItems = _.mapValues(x, (y) => {
        return new Set(_.sortedUniq(_.map(y, 1)))
      })
    }
    let city2Items = {}
    {
      const x = _.groupBy(prefecture_city_city2_nearest_sta, (y) => {
        return y[0] + ':' + y[1]
      })
      city2Items = _.mapValues(x, (y) => {
        return new Set(_.sortedUniq(_.map(y, 2)))
      })
    }

    const nearest_sta_set = new Set(metadata.unique_values.nearest_sta)

    rakumachis = _.map(rakumachis, (x) => {
      x.use_districts = x.use_districts
        ?.replace('一', '１')
        ?.replace('二', '２')

      const type_processed = x.exclusive_area
        ? '中古マンション等'
        : x.floor_area
        ? '宅地(土地と建物)'
        : '宅地(土地)'

      const front_road_processed =
        _.max(
          _.map([...(x.front_road || '').matchAll(/[0-9,.]+/g)], (x) => {
            return +x[0]
          })
        ) || ''

      const built_at_processed = _.get(
        (x.built_at || '').match(/[12][90][0-9][0-9]/),
        0
      )

      const price_processed =
        (+(_.get(x.price.match(/([0-9]+)億/), 1) || 0) * 10000 +
          +_.get(x.price.match(/([0-9]+)万/), 1)) *
        10000

      const income_yearly_processed =
        +(_.get(x.income_yearly?.match(/([0-9,]+)円/), 1) || '').replace(
          /,/g,
          ''
        ) || ''
      const management_fee_monthly_processed =
        +(
          _.get(x.management_fee_monthly?.match(/([0-9,]+)円/), 1) || ''
        ).replace(/,/g, '') || ''
      const repair_reserve_fund_monthly_processed =
        +(
          _.get(x.repair_reserve_fund_monthly?.match(/([0-9,]+)円/), 1) || ''
        ).replace(/,/g, '') || ''
      const yield_processed =
        (income_yearly_processed -
          12 *
            (management_fee_monthly_processed +
              repair_reserve_fund_monthly_processed)) /
        price_processed

      const land_area_processed =
        +(_.get(x.land_area?.match(/([0-9.]+)m2/), 1) || '') || ''
      const floor_area_processed =
        +(_.get(x.floor_area?.match(/([0-9.]+)m2/), 1) || '') || ''
      const exclusive_area_processed =
        +(_.get(x.exclusive_area?.match(/([0-9.]+)m2/), 1) || '') || ''

      let address = x.address || ''
      const prefecture_processed = _.get(address.match(/^.+[都道府県]/), 0)
      address = address.slice(prefecture_processed.length)

      let city_processed = ''
      {
        const x = cityItems[prefecture_processed]
        if (x) {
          for (let i = 1; i < Math.min(10, address.length); i++) {
            if (x.has(address.slice(0, i))) {
              city_processed = address.slice(0, i)
              address = address.slice(i)
              break
            }
          }
        }
      }

      let city2_processed = ''
      {
        const x = city2Items[prefecture_processed + ':' + city_processed]
        if (x) {
          for (let i = 1; i < Math.min(10, address.length); i++) {
            if (x.has(address.slice(0, i))) {
              city2_processed = address.slice(0, i)
              address = address.slice(i)
              break
            }
          }
        }
      }

      let nearest_sta_processed = ''
      let nearest_sta_dist_processed = ''
      {
        const nearest_stas = _.map(x.access.split('\n'), (y) => {
          const dist = _.get(y.match(/徒歩([0-9]+)分/), 1)
          const sta = y
            .replace('駅', '')
            .replace(/徒歩([0-9]+)分/, '')
            .trim()
          let sta2 = ''
          for (let i = 1; i < Math.min(10, sta.length); i++) {
            if (
              nearest_sta_set.has(sta.slice(sta.length - i)) ||
              nearest_sta_set.has(
                sta.slice(sta.length - i) +
                  '(' +
                  prefecture_processed.slice(
                    0,
                    prefecture_processed.length - 1
                  ) +
                  ')'
              )
            ) {
              sta2 = sta.slice(sta.length - i)
              break
            }
          }
          const valid = sta2 && dist
          return valid ? [sta2, +dist] : ['', 10000]
        })

        const z = _.minBy(nearest_stas, 1)
        if (z[0]) {
          nearest_sta_processed = z[0]
          nearest_sta_dist_processed = z[1]
        }
      }

      return {
        ...x,
        type_processed,
        built_at_processed,
        front_road_processed,
        price_processed,
        prefecture_processed,
        city_processed,
        city2_processed,
        nearest_sta_processed,
        nearest_sta_dist_processed,
        income_yearly_processed,
        management_fee_monthly_processed,
        repair_reserve_fund_monthly_processed,
        yield_processed,
        land_area_processed,
        floor_area_processed,
        exclusive_area_processed,
      }
    })

    const params = _.flatten(
      _.map(rakumachis, (y) => {
        y.predict_enabled = y.exclusive_area_processed || y.land_area_processed

        const x = {
          type: y.type_processed,
          prefecture: y.prefecture_processed,
          city: y.city_processed,
          city2: y.city2_processed,
          city_plan: y.use_districts,
          nearest_sta: y.nearest_sta_processed,
          nearest_sta_dist: y.nearest_sta_dist_processed,
          area:
            Math.round(y.exclusive_area_processed || y.land_area_processed) ||
            1,
          floor_area: Math.round(y.floor_area_processed),
          front_road_width: y.front_road_processed,
          time: 202300,
          building_year: y.built_at_processed,
        }
        for (const k in x) {
          if (!x[k]) {
            delete x[k]
          }
        }
        return [x, { ...x, time: 201300 }]
      })
    )

    const base = location.protocol + '//' + location.host + '/'

    const result = await $axios.$post($config.apiBaseUrl + '/predict', params)
    rakumachis = _.map(rakumachis, (x, i) => {
      const price_predict = x.predict_enabled
        ? Math.round(result[2 * i + 0].price)
        : ''
      const price_predict_old = x.predict_enabled
        ? Math.round(result[2 * i + 1].price)
        : ''

      const shareUrl =
        base +
        '?' +
        _.map(
          [
            'type',
            'prefecture',
            'city',
            'city2',
            'city_plan',
            'nearest_sta',
            'nearest_sta_dist',
            'area',
            'floor_area',
            'front_road_width',
            'building_year',
          ],
          (k) => k + '=' + (params[2 * i + 0][k] || '')
        ).join('&')

      return {
        ...x,
        price_predict,
        price_disp: x.price_processed,
        price_predict_disp: price_predict,
        price_predict_old_disp: price_predict_old,
        price_relative_disp: x.price_processed / price_predict - 1,
        shareUrl,
      }
    })

    return {
      search: '',
      tableDebugMode: false,
      rakumachis,
    }
  },
  computed: {
    headers() {
      return this.tableDebugMode
        ? [
            {
              text: 'Name',
              value: 'name',
            },
            {
              text: 'Path',
              value: 'path',
            },
            {
              text: 'Price',
              value: 'price',
            },
            {
              text: 'Address',
              value: 'address',
            },
            {
              text: 'Access',
              value: 'access',
            },
            {
              text: '収益(年額)',
              value: 'income_yearly',
            },
            {
              text: '管理費(月額)',
              value: 'management_fee_monthly',
            },
            {
              text: '修繕積立金(月額)',
              value: 'repair_reserve_fund_monthly',
            },
            {
              text: '土地面積',
              value: 'land_area',
            },
            {
              text: '専有面積',
              value: 'exclusive_area',
            },
            {
              text: '建物面積',
              value: 'floor_area',
            },
            {
              text: '用地地域',
              value: 'use_districts',
            },
            {
              text: '前面道路',
              value: 'front_road',
            },
            {
              text: '築年月',
              value: 'built_at',
            },
            {
              text: '種類(処理)',
              value: 'type_processed',
            },
            {
              text: '築年(処理)',
              value: 'built_at_processed',
            },
            {
              text: '前面道路(処理)',
              value: 'front_road_processed',
            },
            {
              text: '価格(処理)',
              value: 'price_processed',
            },
            {
              text: '県(処理)',
              value: 'prefecture_processed',
            },
            {
              text: '市区町村(処理)',
              value: 'city_processed',
            },
            {
              text: '所在地(処理)',
              value: 'city2_processed',
            },
            {
              text: '最寄り駅(処理)',
              value: 'nearest_sta_processed',
            },
            {
              text: '最寄り駅距離(処理)',
              value: 'nearest_sta_dist_processed',
            },
            {
              text: '収益(処理)',
              value: 'income_yearly_processed',
            },
            {
              text: '管理費(処理)',
              value: 'management_fee_monthly_processed',
            },
            {
              text: '修繕積立金(処理)',
              value: 'repair_reserve_fund_monthly_processed',
            },
            {
              text: '利回り(処理)',
              value: 'yield_processed',
            },
            {
              text: '土地面積(処理)',
              value: 'land_area_processed',
            },
            {
              text: '専有面積(処理)',
              value: 'exclusive_area_processed',
            },
            {
              text: '建物面積(処理)',
              value: 'floor_area_processed',
            },
            {
              text: '価格(予測)',
              value: 'price_predict',
            },
          ]
        : [
            {
              text: 'Name',
              value: 'name',
            },
            {
              text: 'Type',
              value: 'type_processed',
            },
            {
              text: '都道府県',
              value: 'prefecture_processed',
            },
            {
              text: '価格',
              value: 'price_disp',
            },
            {
              text: '価格(予測)',
              value: 'price_predict_disp',
            },
            {
              text: '10年前価格(予測)',
              value: 'price_predict_old_disp',
            },
            {
              text: '相対価格',
              value: 'price_relative_disp',
            },
            {
              text: '利回り',
              value: 'yield_processed',
            },
            {
              text: 'Path',
              value: 'path',
            },
            {
              text: 'シェアリング',
              value: 'shareUrl',
            },
          ]
    },
  },
  mounted() {},
  beforeDestroy() {},
}
</script>
