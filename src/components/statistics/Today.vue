<template>
  <div class="stat-today">
    <ul>
      <li :style="{background: colorSelections[1]}">
        <span class="title">今日派单：</span>
        <span class="num">{{today.today_assigned}}</span>
      </li>
      <li :style="{background: colorSelections[2]}">
        <span class="title">今日完成：</span>
        <span class="num">{{today.today_finished}}</span>
      </li>
      <li :style="{background: colorSelections[3]}">
        <span class="title">今日下单：</span>
        <span class="num">{{today.today_ordered}}</span>
      </li>
      <li :style="{background: colorSelections[4]}">
        <span class="title">今日退单：</span>
        <span class="num">{{today.today_failed}}</span>
      </li>
      <li :style="{background: colorSelections[5]}">
        <span class="title">今日撤销：</span>
        <span class="num">{{today.today_dismissed}}</span>
      </li>
    </ul>
    <ul>
      <li :style="{background: colorSelections[4]}">
        <span class="title">本月派单：</span>
        <span class="num">{{today.monthly_assigned}}</span>
      </li>
      <li :style="{background: colorSelections[3]}">
        <span class="title">本月完成：</span>
        <span class="num">{{today.monthly_finished}}</span>
      </li>
      <li :style="{background: colorSelections[6]}">
        <span class="title">本月下单：</span>
        <span class="num">{{today.monthly_ordered}}</span>
      </li>
      <li :style="{background: colorSelections[1]}">
        <span class="title">本月退单：</span>
        <span class="num">{{today.monthly_failed}}</span>
      </li>
      <li :style="{background: colorSelections[5]}">
        <span class="title">本月撤销：</span>
        <span class="num">{{today.monthly_dismissed}}</span>
      </li>
    </ul>
    <ul>
      <li :style="{background: colorSelections[6]}">
        <span class="title">全部派单：</span>
        <span class="num">{{today.total_assigned}}</span>
      </li>
      <li :style="{background: colorSelections[2]}">
        <span class="title">全部完成：</span>
        <span class="num">{{today.total_finished}}</span>
      </li>
      <li :style="{background: colorSelections[1]}">
        <span class="title">全部下单：</span>
        <span class="num">{{today.total_ordered}}</span>
      </li>
      <li :style="{background: colorSelections[5]}">
        <span class="title">全部退单：</span>
        <span class="num">{{today.total_failed}}</span>
      </li>
      <li :style="{background: colorSelections[3]}">
        <span class="title">全部撤销：</span>
        <span class="num">{{today.total_dismissed}}</span>
      </li>
    </ul>
  </div>
</template>

<script>
export default {
  name: 'stat-today',
  data () {
    return {
      apiItem: {
        apiHost: '',
        apiService: 'zztAuth',
        apiAction: 'Login',
        apiQuery: {}
      },
      apiData: null,
      colorSelections: ['#f2b98e', '#56bced', '#f4afcb', '#5fb985', '#e27719', '#0193de', '#dd1e6c', '#2ac56a'],
      filterShop: null,
      buyers: [],
      today: {
        today_assigned: 0,
        today_finished: 0,
        today_ordered: 0,
        today_failed: 0,
        today_dismissed: 0,
        total_assigned: 0,
        total_finished: 0,
        total_ordered: 0,
        total_failed: 0,
        total_dismissed: 0,
        monthly_assigned: 0,
        monthly_finished: 0,
        monthly_ordered: 0,
        monthly_failed: 0,
        monthly_dismissed: 0
      },
      finishedTrades: null,
      todayProfit: 0
    }
  },
  mounted () {
    this.colorSelections.sort(randomSort) // 随机排序颜色
    this.entrance()
    // this.calcTodayProfit('today')
    // this.calcTodayProfit('monthly')
    // this.calcTodayProfit('total')
  },
  computed: {
  },
  methods: {
    entrance () {
      if (!this.$store.getters.user.role) {
        this.$emit('on-checkauth', this.$route.path)
        setTimeout(() => {
          this.entrance()
        }, 2000)
      } else if (this.$store.getters.user.role === 'employee') { // 买手
        this.getTodayBuyerStatistics(this.$store.getters.user.userid)
      } else {
        this.getTodayStatistics()
      }
    },
    async getBuyers () {
      this.loading = true
      this.apiItem = {
        apiHost: '',
        apiService: 'buyers',
        apiAction: 'listall',
        apiQuery: {}
      }
      this.apiData = {}
      if (this.filterShop) {
        this.apiData.filterShop = this.filterShop
      }
      this.$store.dispatch('setAPIStore', this.apiItem)
      var apiUrl = this.$store.getters.apiUrl
      return new Promise(async (resolve, reject) => {
        await this.$http.post(apiUrl, this.apiData).then(async (response) => {
          var respBody = response.data
          if (respBody.status === 'fail') {
            this.$Message.error('买手列表获取失败！(' + respBody.message + ')')
            reject(new Error('买手列表获取失败！(' + respBody.message + ')'))
          } else {
            // this.$Message.success('列表载入成功!')
            this.$store.dispatch('setAPILastResponse', respBody)
            this.buyers = respBody.data
            resolve(respBody.data)
          }
        }).catch(err => {
          this.$store.dispatch('setAPILastResponse', err)
          reject(err)
        })
      })
    },
    async getTodayStatistics () {
      await this.getBuyers().then((buyers) => {
        buyers.forEach(buyer => {
          this.today.today_assigned += buyer.today_assigned ? buyer.today_assigned : 0
          this.today.today_finished += buyer.today_finished ? buyer.today_finished : 0
          this.today.today_ordered += buyer.today_ordered ? buyer.today_ordered : 0
          this.today.today_failed += buyer.today_failed ? buyer.today_failed : 0
          this.today.today_dismissed += buyer.today_dismissed ? buyer.today_dismissed : 0
          this.today.total_assigned += buyer.total_assigned ? buyer.total_assigned : 0
          this.today.total_finished += buyer.total_finished ? buyer.total_finished : 0
          this.today.total_ordered += buyer.total_ordered ? buyer.total_ordered : 0
          this.today.total_failed += buyer.total_failed ? buyer.total_failed : 0
          this.today.total_dismissed += buyer.total_dismissed ? buyer.total_dismissed : 0
          this.today.monthly_assigned += buyer.monthly_assigned ? buyer.monthly_assigned : 0
          this.today.monthly_finished += buyer.monthly_finished ? buyer.monthly_finished : 0
          this.today.monthly_ordered += buyer.monthly_ordered ? buyer.monthly_ordered : 0
          this.today.monthly_failed += buyer.monthly_failed ? buyer.monthly_failed : 0
          this.today.monthly_dismissed += buyer.monthly_dismissed ? buyer.monthly_dismissed : 0
        })
      })
    },
    getTodayBuyerStatistics (userid) {
      this.apiItem = {
        apiHost: '',
        apiService: 'buyers',
        apiAction: 'getbyuserid',
        apiQuery: {}
      }
      this.apiData = {
        userid: userid
      }
      this.$store.dispatch('setAPIStore', this.apiItem)
      var apiUrl = this.$store.getters.apiUrl
      this.$http.post(apiUrl, this.apiData).then(res => {
        if (res.data.status === 'ok') {
          res.data.data.forEach(buyer => {
            this.today.today_assigned += buyer.today_assigned ? buyer.today_assigned : 0
            this.today.today_finished += buyer.today_finished ? buyer.today_finished : 0
            this.today.today_ordered += buyer.today_ordered ? buyer.today_ordered : 0
            this.today.today_failed += buyer.today_failed ? buyer.today_failed : 0
            this.today.today_dismissed += buyer.today_dismissed ? buyer.today_dismissed : 0
            this.today.total_assigned += buyer.total_assigned ? buyer.total_assigned : 0
            this.today.total_finished += buyer.total_finished ? buyer.total_finished : 0
            this.today.total_ordered += buyer.total_ordered ? buyer.total_ordered : 0
            this.today.total_failed += buyer.total_failed ? buyer.total_failed : 0
            this.today.total_dismissed += buyer.total_dismissed ? buyer.total_dismissed : 0
            this.today.monthly_assigned += buyer.monthly_assigned ? buyer.monthly_assigned : 0
            this.today.monthly_finished += buyer.monthly_finished ? buyer.monthly_finished : 0
            this.today.monthly_ordered += buyer.monthly_ordered ? buyer.monthly_ordered : 0
            this.today.monthly_failed += buyer.monthly_failed ? buyer.monthly_failed : 0
            this.today.monthly_dismissed += buyer.monthly_dismissed ? buyer.monthly_dismissed : 0
          })
        } else {
          this.$Message.error('获取统计信息失败！(' + res.data.message + ')')
        }
      }).catch(err => {
        console.log(err)
        this.$Message.error('获取统计信息失败,请重试。')
      })
    },
    getFinishedTrades (timespan) {
      this.loading = true
      this.apiItem = {
        apiHost: '',
        apiService: 'trades',
        apiAction: 'listfinished',
        apiQuery: {}
      }
      this.apiData = {
        timespan: timespan
      }
      this.$store.dispatch('setAPIStore', this.apiItem)
      var apiUrl = this.$store.getters.apiUrl
      return new Promise(async (resolve, reject) => {
        await this.$http.post(apiUrl, this.apiData).then(async (response) => {
          var respBody = response.data
          if (respBody.status === 'fail') {
            this.$Message.error('订单列表获取失败！(' + respBody.message + ')')
            reject(new Error('订单列表获取失败！(' + respBody.message + ')'))
          } else {
            // this.$Message.success('列表载入成功!')
            this.$store.dispatch('setAPILastResponse', respBody)
            this.finishedTrades = respBody.data
            // console.log(respBody.data)
            resolve(respBody.data)
          }
        }).catch(err => {
          this.$store.dispatch('setAPILastResponse', err)
          reject(err)
        })
      })
    },
    async calcTodayProfit (timespan) {
      await this.getFinishedTrades(timespan).then((trades) => {
        // console.log(trades)
        this.todayProfit = (trades.tradePaymentTotal - trades.orderPaymentTotal).toFixed(2)
        console.log('[' + timespan + '] income: ' + trades.tradePaymentTotal.toFixed(2) + ' | expenditure: ' + trades.orderPaymentTotal.toFixed(2) + ' | gross: ' + (trades.tradePaymentTotal - trades.orderPaymentTotal).toFixed(2))
      })
    }
  }
}
function randomSort (a, b) {
  return Math.random() > 0.5 ? -1 : 1
}
</script>

<style lang="less" scoped>
ul {
  display: flex;
  flex-direction: row;
  justify-content: space-around;
  align-items: center;
  margin-top: 10px;
  margin-bottom: 10px;
  li {
    font-size: 20px;
    position: relative;
    color: #fff;
    display: inline-block;
    position: relative;
    min-height: 90px;
    padding: 10px;
    background:cadetblue;
    width: 19%;
    .title {
      position: absolute;
      top: 10px;
      left: 10px;
      font-size: 18px;
      font-weight: 300;
    }
    .num {
      position: absolute;
      bottom: 10px;
      right: 20px;
      font-size: 36px;
      font-weight: 800;
    }
  }
}
</style>
