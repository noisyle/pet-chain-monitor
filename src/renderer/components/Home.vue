<template>
  <div id="wrapper">
    <div class="toolbar has-text-right">
      <button class="button" :class="isRunning ? 'is-warning': 'is-primary'" @click="toggle()">{{isRunning ? '停止' : '开始'}}</button>
      <button class="button is-primary" @click="openSettings()">设置</button>
    </div>

    <div class="main">
      <table class="table is-bordered is-striped is-narrow is-hoverable is-fullwidth">
        <tbody>
          <tr v-for="pet in tableData" :key="pet.petId">
            <td class="has-text-centered">{{pet.desc}}{{pet.id | formatId}}</td>
            <td class="has-text-centered"><span class="tag" :class="rareStyles[pet.rareDegree]">{{pet.rareDegree | rareName}} ({{pet.rareDegree}})</span></td>
            <td class="has-text-right">{{pet.amount}}</td>
            <td class="has-text-centered">{{pet.timestamp}}</td>
            <td class="has-text-centered"><a class="button is-primary is-small" @click="open(pet)">前往</a></td>
          </tr>
        </tbody>
      </table>
    </div>

    <div class="modal" :class="{ 'is-active': isSettingsOpen }">
      <div class="modal-background"></div>
      <div class="modal-card">
        <header class="modal-card-head">
          <p class="modal-card-title">设置</p>
          <button class="delete" aria-label="close" @click="isSettingsOpen = false"></button>
        </header>
        <section class="modal-card-body">
          <div class="field is-horizontal">
            <div class="field-label is-normal">
              <label class="label">查询间隔</label>
            </div>
            <div class="field-body">
              <div class="field">
                <div class="control">
                  <input class="input" type="number" placeholder="查询间隔（单位：毫秒）" v-model="settings.interval">
                </div>
              </div>
            </div>
          </div>

          <div class="field is-horizontal">
            <div class="field-label is-normal">
              <label class="label">每次查询</label>
            </div>
            <div class="field-body">
              <div class="field">
                <div class="control">
                  <input class="input" type="number" placeholder="每次查询条数" v-model="settings.pageSize">
                </div>
              </div>
            </div>
          </div>

          <div class="field is-horizontal">
            <div class="field-label is-normal">
              <label class="label">售价阈值</label>
            </div>
            <div class="field-body">
              <table class="table is-bordered is-striped is-narrow is-hoverable is-fullwidth">
                <thead>
                  <tr>
                    <th v-for="(style, index) in rareStyles" :key="style" class="has-text-centered"><span class="tag" :class="style">{{index | rareName}}</span></th>
                  </tr>
                </thead>
                <tbody>
                  <tr>
                    <td v-for="(rare, index) in settings.thresholds" :key="index"><input type="number" class="input is-small" name="threshold" v-model="settings.thresholds[index]"></td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </section>
        <footer class="modal-card-foot">
          <button class="button is-primary" @click="applySettings()">应用</button>
        </footer>
      </div>
    </div>
  </div>
</template>

<script>
let runner
const rareNames = ['普通', '稀有', '卓越', '史诗', '神话', '传说']
const rareStyles = ['is-light', 'is-success', 'is-info', 'is-link', 'is-warning', 'is-danger']

export default {
  name: 'home',
  data () {
    return {
      isRunning: false,
      isSettingsOpen: false,
      settings: {
        interval: 2000,
        pageSize: 20,
        thresholds: [0, 0, 0, 0, 0, 0]
      },
      tableData: [],
      tableDataMap: {},
      rareStyles: rareStyles
    }
  },
  methods: {
    open (pet) {
      const link = 'https://pet-chain.baidu.com/chain/detail?channel=market&petId=' + pet.petId + '&validCode='
      this.$electron.shell.openExternal(link)
    },
    toggle () {
      this.isRunning = !this.isRunning
      if (this.isRunning) {
        runner = window.setInterval(this.query, this.settings.interval)
      } else {
        window.clearInterval(runner)
        runner = undefined
      }
    },
    judge (pet) {
      let result = false
      const thresholds = this.settings.thresholds
      for (let i = 0; i < thresholds.length; i++) {
        const threshold = parseFloat(thresholds[i])
        if (threshold > 0 && pet.rareDegree >= i && parseFloat(pet.amount) <= threshold) {
          result = true
          break
        }
      }
      return result
    },
    query () {
      this.$http.post('https://pet-chain.baidu.com/data/market/queryPetsOnSale', {
        'pageNo': 1,
        'pageSize': this.settings.pageSize,
        'querySortType': 'CREATETIME_DESC',
        'petIds': [],
        'requestId': new Date().getTime(),
        'appId': 1
      }).then(res => {
        if (res.data.data.petsOnSale) {
          res.data.data.petsOnSale.forEach(pet => {
            if (this.judge(pet)) {
              if (this.tableDataMap[pet.petId]) {
                this.tableDataMap[pet.petId].amount = pet.amount
              } else {
                pet.timestamp = res.data.timestamp
                this.tableDataMap[pet.petId] = pet
                this.tableData.unshift(pet)
              }
            }
          })
        }
      })
    },
    openSettings () {
      this.isSettingsOpen = true
    },
    applySettings () {
      this.isSettingsOpen = false
      window.clearInterval(runner)
      runner = window.setInterval(this.query, this.settings.interval)
    }
  },
  filters: {
    formatId (value) {
      return '0000000000'.substr(0, 8 - value.toString().length) + value
    },
    rareName (value) {
      return rareNames[value]
    }
  },
  destroyed () {
    window.clearInterval(runner)
    runner = undefined
  }
}
</script>

<style>
html {
  overflow: hidden;
}
body {
  font-family: 'Source Sans Pro', sans-serif;
}
.toolbar {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  padding: 5px;
  border-bottom: 2px solid #ccc;
}
.main {
  position: fixed;
  top: 0;
  left: 0;
  bottom: 0;
  width: 100%;
  margin-top: 48px;
  overflow-y: scroll;
}
</style>
