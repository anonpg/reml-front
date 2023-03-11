<template>
  <v-row justify="center">
    <v-col cols="12" sm="6" md="6">
      <v-card-title> 計算結果 </v-card-title>
      <v-card>
        <v-data-table :headers="headers" :items="results"> </v-data-table>
        <p>表面利回り: {{ idealYield * 100 }}%</p>
        <p>純利益利回り: {{ profitYield * 100 }}%</p>
        <p>キャッシュフロー利回り: {{ cashFlowYield * 100 }}%</p>
        <p>売上: {{ Math.round(income).toLocaleString() }}円</p>
        <p>経費: {{ Math.round(expense).toLocaleString() }}円</p>
        <p>税引前利益: {{ Math.round(profitBeforeTax).toLocaleString() }}円</p>
        <p>純利益: {{ Math.round(profit).toLocaleString() }}円</p>
        <p>キャッシュフロー: {{ Math.round(cashFlow).toLocaleString() }}円</p>
        <p>利子: {{ Math.round(interest).toLocaleString() }}円</p>
        <p>返済額: {{ Math.round(repayment).toLocaleString() }}円</p>
        <p>減価償却費: {{ Math.round(depreciation).toLocaleString() }}円</p>
        <p>法定耐用年数: {{ legalLifetime }}年</p>
        <p>耐用年数: {{ lifetime }}年</p>
        <p>{{ cumProfits }}</p>
        <p>{{ cumCashFlows }}</p>
      </v-card>
    </v-col>
    <v-col cols="12" sm="6" md="6">
      <v-card-title> 入力 </v-card-title>
      <v-card>
        <v-text-field
          v-model="priceMan"
          label="価格(万円)"
          type="number"
          :max="100000"
          :min="1"
        />
        <v-text-field
          v-model="fundOnHandMan"
          label="自己資金(万円)"
          type="number"
          :max="priceMan"
          :min="0"
        />
        <v-text-field
          v-model="interestPercent"
          label="金利(%)"
          type="number"
          :max="100"
          :min="0"
        />
        <v-text-field
          v-model="repaymentYears"
          label="返済期間(年)"
          type="number"
          :max="40"
          :min="1"
        />
        <v-text-field
          v-model="buildingPricePercent"
          label="建物価格率(%)"
          type="number"
          :max="100"
          :min="0"
        />
        <v-text-field
          v-model="buildingYear"
          label="築年(西暦)"
          type="number"
          :max="2030"
          :min="1900"
        />
        <v-select
          v-model="structure"
          :items="structureToLegalLifetimeKeys"
          label="建物構造"
        ></v-select>
        <v-text-field
          v-model="rent"
          label="家賃(円/月)"
          type="number"
          :max="1000000"
          :min="10000"
        />
        <v-text-field
          v-model="managementFee"
          label="管理費(円/月)"
          type="number"
          :max="100000"
          :min="1000"
        />
        <v-text-field
          v-model="repairReserveFund"
          label="修繕積立金(円/月)"
          type="number"
          :max="100000"
          :min="1000"
        />
        <v-text-field
          v-model="rentManagementFee"
          label="賃貸管理費(円/月)"
          type="number"
          :max="100000"
          :min="1000"
        />
        <v-text-field
          v-model="fireInsurance"
          label="火災保険(円/年)"
          type="number"
          :max="100000"
          :min="100"
        />
        <v-text-field
          v-model="earthquakeInsurance"
          label="地震保険(円/年)"
          type="number"
          :max="100000"
          :min="100"
        />
        <v-text-field
          v-model="corporateTaxPercent"
          label="法人税率(%)"
          type="number"
          :max="100"
          :min="0"
        />
        <v-text-field
          v-model="propertyTaxPercent"
          label="固定資産税率(%)"
          type="number"
          :max="100"
          :min="0"
        />
      </v-card>
    </v-col>
  </v-row>
</template>

<script>
import _ from 'lodash'

export default {
  name: 'IndexPage',
  data() {
    return {
      priceMan: 2300,
      fundOnHandMan: 1000,
      // running
      rent: 86000,
      managementFee: 6500,
      repairReserveFund: 5500,
      rentManagementFee: 5300,
      fireInsurance: 2200,
      earthquakeInsurance: 4200,
      corporateTaxPercent: 37,
      propertyTaxPercent: 0.2,
      interestPercent: 3,
      repaymentYears: 10,
      structure: 'RC,SRC',
      buildingYear: 2002,
      buildingPricePercent: 20,
      // initial
      execCostRate: 0.05,
    }
  },
  computed: {
    headers() {
      return [
        {
          text: 'Year',
          value: 'year',
        },
        {
          text: '資産',
          value: 'assets',
        },
        {
          text: '現金',
          value: 'cash',
        },
        {
          text: '不動産簿価',
          value: 'bookValue',
        },
        {
          text: '純資産',
          value: 'netAssets',
        },
        {
          text: '負債',
          value: 'debt',
        },
        {
          text: '利子',
          value: 'interest',
        },
        {
          text: '経費',
          value: 'expense',
        },
        {
          text: '減価償却費',
          value: 'depreciation',
        },
        {
          text: '税引前利益',
          value: 'profitBeforeTax',
        },
        {
          text: '法人税',
          value: 'corporateTax',
        },
        {
          text: '純利益',
          value: 'profit',
        },
        {
          text: 'キャッシュフロー',
          value: 'cashFlow',
        },
        {
          text: '返済額',
          value: 'repayment',
        },
      ]
    },
    results() {
      const x = []
      let debt = this.price - this.fundOnHand
      let cash = 0
      let bookValue = +this.price

      for (let i = 0; i < 30; i++) {
        // 元金均等返済
        const interest = (debt * this.interestPercent) / 100
        const repayment = Math.min(
          debt,
          (this.price - this.fundOnHand) / this.repaymentYears
        )
        debt -= repayment

        // 減価償却
        const depreciation =
          i < this.lifetime
            ? (this.price * this.buildingPricePercent) / 100 / this.lifetime
            : 0

        const expense =
          (+this.managementFee +
            +this.repairReserveFund +
            +this.rentManagementFee) *
            12 +
          +this.fireInsurance +
          +this.earthquakeInsurance +
          interest +
          (this.price * this.propertyTaxPercent) / 100 +
          depreciation

        const profitBeforeTax = this.income - expense
        const corporateTax =
          (Math.max(0, this.income - expense) * this.corporateTaxPercent) / 100
        const profit = profitBeforeTax - corporateTax

        const cashFlow = profit + depreciation - repayment
        cash += cashFlow

        bookValue -= depreciation
        const assets =
          cash +
          (this.price -
            ((+this.price - bookValue) * this.corporateTaxPercent) / 100)
        const netAssets = assets - debt

        x.push({
          year: i,
          debt,
          interest,
          depreciation,
          expense,
          repayment,
          profitBeforeTax,
          corporateTax,
          profit,
          cashFlow,
          assets,
          netAssets,
          cash,
          bookValue,
        })
      }

      return x
    },
    price() {
      return this.priceMan * 10000
    },
    fundOnHand() {
      return this.fundOnHandMan * 10000
    },
    idealYield() {
      return this.income / this.price
    },
    income() {
      return this.rent * 12
    },
    expense() {
      return (
        (+this.managementFee +
          +this.repairReserveFund +
          +this.rentManagementFee) *
          12 +
        +this.fireInsurance +
        +this.earthquakeInsurance +
        +this.interest +
        (+this.price * this.propertyTaxPercent) / 100 +
        this.depreciation
      )
    },
    profit() {
      return (this.income - this.expense) * (1 - this.corporateTaxPercent / 100)
    },
    profitBeforeTax() {
      return this.income - this.expense
    },
    cashFlow() {
      return this.profit - this.repayment + this.depreciation
    },
    cumProfits() {
      return _.reduce(
        this.profits,
        (s, x) => {
          return [...s, (_.last(s) || 0) + x]
        },
        []
      )
    },
    cumCashFlows() {
      return _.reduce(
        this.cashFlows,
        (s, x) => {
          return [...s, (_.last(s) || 0) + x]
        },
        []
      )
    },
    profits() {
      const profits = []
      for (let i = 0; i < Math.max(this.lifetime, this.repaymentYears); i++) {
        let pf = this.profitBeforeTax
        if (i >= this.lifetime) {
          pf += this.depreciation
        }
        if (i >= this.repaymentYears) {
          pf += this.interest
        }
        profits.push(pf * (1 - this.corporateTaxPercent / 100))
      }
      return profits
    },
    cashFlows() {
      const flows = []

      for (let i = 0; i < Math.max(this.lifetime, this.repaymentYears); i++) {
        let flow = this.profits[i]
        if (i < this.lifetime) {
          flow += this.depreciation
        }
        if (i < this.repaymentYears) {
          flow -= this.repayment
        }
        flows.push(flow)
      }
      return flows
    },
    interest() {
      return ((this.price - this.fundOnHand) * this.interestPercent) / 100
    },
    repayment() {
      return (this.price - this.fundOnHand) / this.repaymentYears
    },
    profitYield() {
      return this.profit / this.fundOnHand
    },
    cashFlowYield() {
      return this.cashFlow / this.fundOnHand
    },
    depreciation() {
      return (this.price * this.buildingPricePercent) / 100 / this.lifetime
    },
    legalLifetime() {
      return this.structureToLegalLifetime[this.structure]
    },
    lifetime() {
      const age = 2023 - this.buildingYear
      if (age < this.legalLifetime) {
        return Math.floor(this.legalLifetime - age + age * 0.2)
      } else {
        return Math.floor(this.legalLifetime * 0.2)
      }
    },
    structureToLegalLifetime() {
      // https://www.nta.go.jp/taxes/shiraberu/taxanswer/shotoku/pdf/2100_01.pdf
      return {
        木造: 22,
        'RC,SRC': 47,
        鉄骨: 27,
      }
    },
    structureToLegalLifetimeKeys() {
      return _.keys(this.structureToLegalLifetime)
    },
    // stampDuty() {
    //   // https://www.nta.go.jp/law/shitsugi/inshi/08/10.htm
    //   const x = this.price / 10000
    //   if (x > 50 * 10000) {
    //     return 48 * 10000
    //   } else if (x > 10 * 10000) {
    //     return 32 * 10000
    //   }else if (x > 5 * 10000) {
    //     return 16 * 10000
    //   }else if (x > 1 * 10000) {
    //     return 6 * 10000
    //   }else if (x > 5000) {
    //     return 3 * 10000
    //   }else if (x > 1000) {
    //     return 1 * 10000
    //   }else if (x > 500) {
    //     return 5000
    //   }else if (x > 100) {
    //     return 1000
    //   }else if (x > 50) {
    //     return 500
    //   }else if (x > 10) {
    //     return 200
    //   }
    //   return 0
    // }
  },
  mounted() {},
  beforeDestroy() {},
}
</script>
