<template>
  <div id="wrapper">
    <div class="tabs" style="margin-top: 7px;">
      <ul>
        <li :class="{'is-active' : currentTab === 'filtered'}"><a @click="currentTab = 'filtered'">筛选</a></li>
        <li :class="{'is-active' : currentTab === 'log'}"><a @click="currentTab = 'log'">日志</a></li>
      </ul>
    </div>
    <div class="toolbar has-text-right">
      <button class="button" :class="isRunning ? 'is-warning': 'is-primary'" @click="toggle()">{{isRunning ? '停止' : '开始'}}</button>
      <button class="button is-primary" @click="openSettings()">设置</button>
    </div>

    <div class="main">
      <table v-show="currentTab === 'filtered'" class="table is-bordered is-striped is-narrow is-hoverable is-fullwidth">
        <tbody>
          <tr v-for="pet in filteredData" :key="pet.petId">
            <td class="has-text-centered">{{pet.desc}}{{pet.id | formatId}}</td>
            <td class="has-text-centered"><span class="tag" :class="rareStyles[pet.rareDegree]">{{pet.rareDegree | rareName}} ({{pet.rareDegree}})</span></td>
            <td class="has-text-right">{{pet.amount}}</td>
            <td class="has-text-centered">第{{pet.generation}}代</td>
            <td class="has-text-centered">{{pet.isCooling?'休息中':'正常'}}</td>
            <td class="has-text-centered">{{pet.coolingInterval}}</td>
            <td class="has-text-centered">{{pet.timestamp | formatDate}}</td>
            <td class="has-text-centered"><a class="button is-primary is-small" @click="open(pet)">前往</a></td>
          </tr>
        </tbody>
      </table>
      <table v-show="currentTab === 'log'" class="table is-bordered is-striped is-narrow is-hoverable is-fullwidth">
        <tbody>
          <tr v-for="log in logData" :key="log.logId">
            <td class="has-text-centered">{{log.data.desc}}{{log.data.id | formatId}}</td>
            <td class="has-text-centered"><span class="tag" :class="rareStyles[log.data.rareDegree]">{{log.data.rareDegree | rareName}} ({{log.data.rareDegree}})</span></td>
            <td class="has-text-right">{{log.data.amount}}</td>
            <td class="has-text-centered">第{{log.data.generation}}代</td>
            <td class="has-text-centered">{{log.data.isCooling?'休息中':'正常'}}</td>
            <td class="has-text-centered">{{log.data.coolingInterval}}</td>
            <td class="has-text-centered">{{log.data.timestamp | formatDate}}</td>
            <td class="has-text-centered"><a class="button is-primary is-small" @click="open(log)">前往</a></td>
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
              <label class="label">最小稀有度</label>
            </div>
            <div class="field-body">
              <div class="field">
                <div class="control">
                  <div class="select">
                    <select v-model="settings.filter.rareDegree">
                      <option value="-1">全部</option>
                      <option v-for="(style, index) in rareStyles" :key="index" :value="index">{{index | rareName}}</option>
                    </select>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="field is-horizontal">
            <div class="field-label is-normal">
              <label class="label">最高售价</label>
            </div>
            <div class="field-body">
              <div class="field">
                <div class="control">
                  <input class="input" type="number" v-model="settings.filter.threshold">
                </div>
              </div>
            </div>
          </div>

          <div class="field is-horizontal">
            <div class="field-label is-normal">
              <label class="label">最大代数</label>
            </div>
            <div class="field-body">
              <div class="field">
                <div class="control">
                  <input class="input" type="number" v-model="settings.filter.generation">
                </div>
              </div>
            </div>
          </div>

          <div class="field is-horizontal">
            <div class="field-label is-normal">
              <label class="label">休息时间</label>
            </div>
            <div class="field-body">
              <div class="field">
                <div class="control">
                  <input class="input" type="number" v-model="settings.filter.cooling">
                </div>
              </div>
              <div class="field">
                <div class="control">
                  <div class="select">
                    <select v-model="settings.filter.coolingInterval">
                      <option v-for="(value, name) in coolingInterval" :key="value" :value="name">{{name}}</option>
                    </select>
                  </div>
                </div>
              </div>
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
'use strict'

import { ipcRenderer } from 'electron'

let runner
const rareNames = ['普通', '稀有', '卓越', '史诗', '神话', '传说']
const rareStyles = ['is-light', 'is-success', 'is-info', 'is-link', 'is-warning', 'is-danger']
const coolingInterval = {
  '分钟': 1,
  '天': 24 * 60,
  '周': 7 * 24 * 60,
  '个月': 30 * 24 * 60,
  '年': 365 * 24 * 60
}

export default {
  name: 'home',
  data () {
    return {
      isRunning: false,
      isSettingsOpen: false,
      settings: {
        interval: 2000,
        pageSize: 10,
        filter: {
          rareDegree: -1,
          threshold: 0,
          generation: -1,
          cooling: 0,
          coolingInterval: '分钟'
        }
        // thresholds: [0, 0, 0, 0, 0, 0]
      },
      currentTab: 'filtered',
      filteredData: [],
      filteredDataMap: {},
      logData: [],
      rareStyles: rareStyles,
      coolingInterval: coolingInterval
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
        if (runner !== undefined) {
          window.clearInterval(runner)
        }
        runner = window.setInterval(this.query, this.settings.interval)
      } else {
        window.clearInterval(runner)
        runner = undefined
      }
    },
    judge (pet) {
      let result = true
      const threshold = parseFloat(this.settings.filter.threshold)
      if (threshold > 0 && parseFloat(pet.amount) > threshold) {
        result = false
      }
      const rareDegree = parseFloat(this.settings.filter.rareDegree)
      if (rareDegree > 0 && parseFloat(pet.rareDegree) < rareDegree) {
        result = false
      }
      const generation = parseFloat(this.settings.filter.generation)
      if (generation > -1 && parseFloat(pet.generation) > generation) {
        result = false
      }
      const cooling = parseInt(coolingInterval[this.settings.filter.coolingInterval] * this.settings.filter.cooling)
      if (parseFloat(pet.cooling) > cooling) {
        result = false
      }
      return result
    },
    intervalString2Number (str) {
      let result = null
      Object.keys(coolingInterval).forEach(i => {
        if (str.indexOf(i) > -1) {
          result = parseInt(parseFloat(str.replace(i, '')) * coolingInterval[i])
        }
      })
      return result
    },
    query () {
      this.$http.post('https://pet-chain.baidu.com/data/market/queryPetsOnSale', {
        'appId': 1,
        'filterCondition': '{}',
        'lastAmount': '',
        'lastRareDegree': '',
        'nounce': null,
        'pageNo': 1,
        'pageSize': this.settings.pageSize,
        'petIds': [],
        'phoneType': 'ios',
        'querySortType': 'CREATETIME_DESC',
        'requestId': new Date().getTime(),
        'timeStamp': null,
        'token': null,
        'tpl': '',
        'type': null
      }).then(res => {
        if (res.data.data) {
          const ts = new Date(res.data.timestamp)
          res.data.data.petsOnSale.forEach(pet => {
            pet.timestamp = ts
            pet.cooling = this.intervalString2Number(pet.coolingInterval)
            if (this.judge(pet)) {
              if (this.filteredDataMap[pet.petId]) {
                this.filteredDataMap[pet.petId].amount = pet.amount
              } else {
                this.filteredDataMap[pet.petId] = pet
                this.filteredData.unshift(pet)
                ipcRenderer.send('notice', pet)
              }
            }

            this.logData.push({logId: pet.petId + '_' + pet.timestamp, data: pet})
          })

          if (this.logData.length > 1000) {
            this.logData.splice(0, this.logData.length - 1000)
          }
        }
      })
    },
    openSettings () {
      this.isSettingsOpen = true
    },
    applySettings () {
      if (runner !== undefined) {
        window.clearInterval(runner)
        runner = window.setInterval(this.query, this.settings.interval)
      }
      this.isSettingsOpen = false
    }
  },
  filters: {
    formatId (value) {
      return '0000000000'.substr(0, 8 - value.toString().length) + value
    },
    formatDate (date, fmt) {
      fmt || (fmt = 'yyyy-MM-dd hh:mm:ss')
      const o = {
        'M+': date.getMonth() + 1,
        'd+': date.getDate(),
        'h+': date.getHours(),
        'm+': date.getMinutes(),
        's+': date.getSeconds(),
        'q+': Math.floor((date.getMonth() + 3) / 3),
        'S': date.getMilliseconds()
      }
      if (/(y+)/.test(fmt)) {
        fmt = fmt.replace(RegExp.$1, (date.getFullYear() + '').substr(4 - RegExp.$1.length))
      }
      for (let k in o) {
        if (new RegExp('(' + k + ')').test(fmt)) {
          fmt = fmt.replace(RegExp.$1, (RegExp.$1.length === 1) ? (o[k]) : (('00' + o[k]).substr(('' + o[k]).length)))
        }
      }
      return fmt
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
  right: 0;
  padding: 5px;
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
