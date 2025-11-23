<script setup>
import { computed, onMounted, ref } from 'vue'
import moment from 'moment'
import CPU from '@/components/CPU.vue'
import Mem from '@/components/Mem.vue'
import NetIn from '@/components/NetIn.vue'
import NetOut from '@/components/NetOut.vue'
import axios from 'axios'
// import { Message } from '@arco-design/web-vue'
import StatsCard from '@/components/StatsCard.vue'
import { calculateRemainingDays, formatBytes, formatTimeStamp, formatUptime } from '@/utils/utils'
import { darkTheme, useOsTheme } from 'naive-ui'
import TheMessage from '@/components/naive/TheMessage.vue'
import Message from '@/utils/message.js'

import { Edit, Delete } from '@vicons/carbon'

const message = new Message()

const socketURL = ref('')
const apiURL = ref('')

const osThemeRef = useOsTheme()

const theme = computed(() => (osThemeRef.value === 'dark' ? darkTheme : null))
// const theme = window.localStorage.getItem('theme') || 'light'
// const dark = ref(theme !== 'light')

const area = ref([])
const selectArea = ref('all')

const type = ref('all')

const data = ref([])

const selectHost = ref('')

const charts = ref({})

const cpuRef = ref(null)
const memRef = ref(null)
const netInRef = ref(null)
const netOutRef = ref(null)

const host = computed(() => {
  const result = data.value
  result.sort((a, b) => {
    if (a.Host.Name < b.Host.Name) {
      return -1
    }
    if (a.Host.Name > b.Host.Name) {
      return 1
    }
    return 0
  })
  if (selectArea.value === 'all') {
    return result
  }

  return result.filter((item) => item.Host.Name.slice(0, 2) === selectArea.value)
})

const hosts = computed(() => {
  if (type.value === 'all') {
    return host.value
  } else if (type.value === 'online') {
    return host.value.filter((item) => item.status)
  } else {
    return host.value.filter((item) => !item.status)
  }
})

const stats = computed(() => {
  const online = host.value.filter((item) => item.status)
  let bandwidth_up = 0
  let bandwidth_down = 0
  let traffic_up = 0
  let traffic_down = 0

  host.value.forEach((item) => {
    bandwidth_up += item.State.NetOutSpeed
    bandwidth_down += item.State.NetInSpeed
    traffic_up += item.State.NetOutTransfer
    traffic_down += item.State.NetInTransfer
  })

  return {
    total: host.value.length,
    online: online.length,
    offline: host.value.length - online.length,
    bandwidth_up: bandwidth_up,
    bandwidth_down: bandwidth_down,
    traffic_up: traffic_up,
    traffic_down: traffic_down
  }
})

let socket = null

let nowtime = Math.floor(Date.now() / 1000)

const fetchConfig = async () => {
  try {
    const res = await axios.get('/config.json')
    socketURL.value = res.data.socket
    apiURL.value = res.data.apiURL
  } catch (e) {
    message.error('获取配置失败')
  }
}

const initScoket = async () => {
  await fetchConfig()

  socket = new WebSocket(socketURL.value) // 替换为实际的 WebSocket 服务器 URL

  socket.onmessage = function (event) {
    try {
      const messageWs = event.data
      const res = JSON.parse(messageWs.replace('data: ', ''))
      if (res != null) {
        area.value = Array.from(new Set(res.map((item) => item.Host.Name.slice(0, 2))))
      }
      data.value = res.map((host) => {
        if (!charts.value[host.Host.Name]) {
          charts.value[host.Host.Name] = {
            cpu: [],
            mem: [],
            net_in: [],
            net_out: []
          }
        }

        if (charts.value[host.Host.Name].cpu.length === 2) {
          charts.value[host.Host.Name].cpu = charts.value[host.Host.Name].cpu.slice(1)
          charts.value[host.Host.Name].mem = charts.value[host.Host.Name].mem.slice(1)
          charts.value[host.Host.Name].net_in = charts.value[host.Host.Name].net_in.slice(1)
          charts.value[host.Host.Name].net_out = charts.value[host.Host.Name].net_out.slice(1)
        }

        charts.value[host.Host.Name].cpu.push([host.TimeStamp * 1000, host.State.CPU])
        charts.value[host.Host.Name].mem.push([host.TimeStamp * 1000, host.State.MemUsed])
        charts.value[host.Host.Name].net_in.push([host.TimeStamp * 1000, host.State.NetOutSpeed])
        charts.value[host.Host.Name].net_out.push([host.TimeStamp * 1000, host.State.NetInSpeed])

        return {
          ...host,
          status: nowtime - host.TimeStamp <= 10 ? 1 : 0
        }
      })

      setTimeout(() => sendPing(), 1000)
    } catch (error) {
      console.error('解析 WebSocket 消息时出错:', error)
    }
  }

  socket.onopen = function () {
    sendPing()
  }

  socket.onclose = function () {
    message.warning('WebSocket已断连，正在重连中...')

    initScoket()
  }
}

const sendPing = () => {
  nowtime = Math.floor(Date.now() / 1000)
  socket.send('ping')
}

onMounted(async () => {
  // if (dark.value) {
  //   document.body.setAttribute('arco-theme', 'dark')
  // }

  await initScoket()
  await handleFetchHostInfo()
})

const progressStatus = (value) => {
  if (value < 80) {
    return 'success'
  } else if (value < 90) {
    return 'warning'
  } else {
    return 'danger'
  }
}

const handleSelectArea = (area) => {
  selectArea.value = area
}

const handleSelectHost = (host) => {
  if (selectHost.value === host) {
    selectHost.value = ''
    return
  }

  selectHost.value = host
}

// 删除主机
const deleteVisible = ref(false)
const authSecret = ref('')
const deleteHostName = ref('')

const handleShowDelete = (name) => {
  authSecret.value = window.localStorage.getItem('auth_secret') || ''
  deleteHostName.value = name
  deleteVisible.value = true
}

const handleDeleteHost = async () => {
  try {
    await axios.post(apiURL.value + '/delete', {
      auth_secret: authSecret.value,
      name: deleteHostName.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    message.success('删除成功')

    deleteVisible.value = false
  } catch (e) {
    message.error('删除失败，管理密钥错误')
  }
}

const handleClose = () => {
  deleteVisible.value = false
}

const hostInfo = ref({})

const handleFetchHostInfo = async () => {
  try {
    const res = await axios.get(apiURL.value + '/info')
    res.data.forEach((item) => {
      hostInfo.value[item.name] = item
    })
  } catch (e) {
    // message.error('删除失败，管理密钥错误')
  }
}

const handleChangeType = (value) => {
  type.value = value
}

const editHostName = ref('')
const duetime = ref(null)
const buy_url = ref('')
const seller = ref('')
const price = ref('')

const editVisible = ref(false)

const handleShowEdit = (name) => {
  if (!hostInfo.value[name]) {
    editHostName.value = name
    duetime.value = null
    buy_url.value = ''
    seller.value = ''
    price.value = ''
    editVisible.value = true
    authSecret.value = window.localStorage.getItem('auth_secret') || ''
    return
  }

  editHostName.value = name
  duetime.value = hostInfo.value[name].due_time
  buy_url.value = hostInfo.value[name].buy_url
  seller.value = hostInfo.value[name].seller
  price.value = hostInfo.value[name].price
  editVisible.value = true
  authSecret.value = window.localStorage.getItem('auth_secret') || ''
}

const handleEditHost = async () => {
  try {
    await axios.post(apiURL.value + '/info', {
      name: editHostName.value,
      due_time: new Date(duetime.value).getTime(),
      buy_url: buy_url.value,
      seller: seller.value,
      auth_secret: authSecret.value,
      price: price.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    message.success('更新成功')

    await handleFetchHostInfo()

    editVisible.value = false
  } catch (e) {
    message.error('更新失败，管理密钥错误')
  }
}

const handleEditClose = () => {
  editVisible.value = false
}
</script>

<template>
  <n-config-provider :theme="theme">
    <n-message-provider>
      <the-message />
      <n-layout :native-scrollbar="false" style="height: 100dvh">
        <n-el class="max-container">
          <n-el style="padding-inline: 0.75rem">
            <div class="header">
              <n-el class="logo">
                <img class="icon" src="/favicon.ico" alt="" />
                <n-text style="font-weight: 400; opacity: 0.8">LoCyanFrp｜节点服务器监控</n-text>
              </n-el>
              <!--      <a-button class="theme-btn" :shape="'round'" @click="handleChangeDark">-->
              <!--        <template #icon>-->
              <!--          <icon-sun-fill v-if="!dark" />-->
              <!--          <icon-moon-fill v-else />-->
              <!--        </template>-->
              <!--      </a-button>-->
            </div>
            <div class="area-tabs">
              <n-button
                class="area-tab-item"
                :type="selectArea === 'all' ? 'success' : ''"
                :class="selectArea === 'all' ? 'is-active' : ''"
                @click="handleSelectArea('all')"
              >
                全部地区
              </n-button>
              <n-button
                class="area-tab-item"
                :type="selectArea === item ? 'success' : ''"
                :class="selectArea === item ? 'is-active' : ''"
                v-for="item in area"
                :key="item"
                @click="handleSelectArea(item)"
              >
                <span
                  :class="`flag-icon flag-icon-${item.replace('TW', 'CN').replace('UK', 'GB').toLowerCase()}`"
                  style="margin-right: 3px"
                ></span>
                {{ item }}
              </n-button>
            </div>
            <StatsCard :type="type" :stats="stats" @handleChangeType="handleChangeType" />
            <n-divider />
            <div class="monitor-card">
              <n-card
                hoverable
                class="monitor-item"
                :class="selectHost === item.Host.Name ? 'is-active' : ''"
                v-for="(item, index) in hosts"
                @click="handleSelectHost(item.Host.Name)"
                :key="index"
              >
                <div class="name">
                  <div class="title">
                    <span
                      :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('TW', 'CN').replace('UK', 'GB').toLowerCase()}`"
                    ></span>
                    {{ item.Host.Name }}
                  </div>
                  <div class="status" :class="item.status ? 'online' : 'offline'">
                    <span>{{ item.status ? '在线' : '离线' }}</span>
                    <span style="margin-left: 6px">{{ formatUptime(item.State.Uptime) }}</span>
                  </div>
                </div>
                <div class="platform">
                  <div class="monitor-item-title">系统</div>
                  <div class="monitor-item-value">
                    {{ item.Host.Platform }} {{ item.Host.PlatformVersion }}
                  </div>
                </div>
                <div class="cpu">
                  <div class="monitor-item-title">CPU</div>
                  <div class="monitor-item-value">{{ item.State.CPU.toFixed(2) + '%' }}</div>
                  <a-progress
                    class="monitor-item-progress"
                    :status="progressStatus(item.State.CPU)"
                    :percent="item.State.CPU / 100"
                    :show-text="false"
                    style="width: 60px"
                  />
                </div>
                <div class="mem">
                  <div class="monitor-item-title">内存使用情况</div>
                  <div class="monitor-item-value">
                    {{ ((item.State.MemUsed / item.Host.MemTotal) * 100).toFixed(2) + '%' }}
                  </div>
                  <a-progress
                    class="monitor-item-progress"
                    :status="progressStatus((item.State.MemUsed / item.Host.MemTotal) * 100)"
                    :percent="item.State.MemUsed / item.Host.MemTotal"
                    :show-text="false"
                    style="width: 60px"
                  />
                </div>
                <div class="network">
                  <div class="monitor-item-title">网络速度（IN|OUT）</div>
                  <div class="monitor-item-value">
                    {{
                      `${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`
                    }}
                  </div>
                </div>
                <div class="average">
                  <div class="monitor-item-title">负载平均值(1|5|15)</div>
                  <div class="monitor-item-value">
                    {{ `${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}` }}
                  </div>
                </div>
                <div class="uptime" style="width: 120px">
                  <div class="monitor-item-title">剩余时间</div>
                  <div class="monitor-item-value">
                    {{
                      hostInfo[item.Host.Name]
                        ? calculateRemainingDays(hostInfo[item.Host.Name].due_time)
                        : '-'
                    }}
                  </div>
                </div>
                <div class="uptime">
                  <div class="monitor-item-title">上报时间</div>
                  <div class="monitor-item-value">{{ formatTimeStamp(item.TimeStamp) }}</div>
                </div>
                <template #action class="detail" v-if="selectHost === item.Host.Name">
                  <a-row>
                    <a-col :span="10" :xs="24" :sm="24" :md="10" :lg="10" :sl="10">
                      <div class="detail-item-list">
                        <div class="detail-item">
                          <n-text class="name">主机名</n-text>
                          <n-text class="value">{{ item.Host.Name }}</n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">地区</n-text>
                          <n-text class="value">
                            <span
                              :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('TW', 'CN').replace('UK', 'GB').toLowerCase()}`"
                            ></span>
                            {{ item.Host.Name.slice(0, 2).toUpperCase() }}
                          </n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">系统</n-text>
                          <n-text class="value"
                            >{{ item.Host.Platform }} {{ item.Host.PlatformVersion }}</n-text
                          >
                        </div>
                        <div class="detail-item">
                          <n-text class="name">架构</n-text>
                          <n-text class="value">{{ item.Host.Arch }}</n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">虚拟化</n-text>
                          <n-text class="value">{{ item.Host.Virtualization || '-' }}</n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">CPU</n-text>
                          <n-text class="value">{{ item.Host.CPU.join(',') }}</n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">CPU占用</n-text>
                          <n-text class="value">{{ item.State.CPU.toFixed(2) + '%' }}</n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">内存使用情况</n-text>
                          <n-text class="value"
                            >{{
                              ((item.State.MemUsed / item.Host.MemTotal) * 100).toFixed(2) + '%'
                            }}
                            ({{ formatBytes(item.State.MemUsed) }} /
                            {{ formatBytes(item.Host.MemTotal) }})
                          </n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">虚拟内存(Swap)</n-text>
                          <n-text class="value"
                            >{{ formatBytes(item.State.SwapUsed) }} /
                            {{ formatBytes(item.Host.SwapTotal) }}
                          </n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">网络速度（IN|OUT）</n-text>
                          <n-text class="value">
                            {{
                              `${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`
                            }}
                          </n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">负载平均值(1|5|15)</n-text>
                          <n-text class="value"
                            >{{
                              `${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`
                            }}
                          </n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">流量使用↑|↓</n-text>
                          <n-text class="value"
                            >{{ formatBytes(item.State.NetOutTransfer) }} |
                            {{ formatBytes(item.State.NetInTransfer) }}
                          </n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">开机时间</n-text>
                          <n-text class="value">{{ formatTimeStamp(item.Host.BootTime) }}</n-text>
                        </div>
                        <div class="detail-item">
                          <n-text class="name">上报时间</n-text>
                          <n-text class="value">{{ formatTimeStamp(item.TimeStamp) }}</n-text>
                        </div>
                        <div
                          class="detail-item"
                          v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].seller"
                        >
                          <n-text class="name">商家名称</n-text>
                          <n-text class="value">{{ hostInfo[item.Host.Name].seller }}</n-text>
                        </div>
                        <div
                          class="detail-item"
                          v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].price"
                        >
                          <n-text class="name">主机价格</n-text>
                          <n-text class="value">{{ hostInfo[item.Host.Name].price }}</n-text>
                        </div>
                        <div
                          class="detail-item"
                          v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].due_time"
                        >
                          <n-text class="name">到期时间</n-text>
                          <n-text class="value"
                            >{{ moment(hostInfo[item.Host.Name].due_time).format('YYYY-MM-DD') }}
                          </n-text>
                        </div>
                        <div
                          class="detail-item"
                          v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].buy_url"
                        >
                          <n-text class="name">购买链接</n-text>
                          <n-text class="value">
                            <a
                              style="color: #0077ff"
                              :href="hostInfo[item.Host.Name].buy_url"
                              target="_blank"
                              @click.stop="() => {}"
                              >{{ hostInfo[item.Host.Name].buy_url }}</a
                            >
                          </n-text>
                        </div>
                      </div>
                    </a-col>
                    <a-col :span="14" :xs="24" :sm="24" :md="14" :lg="14" :sl="14">
                      <a-row :gutter="20">
                        <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                          <div style="margin-bottom: 20px">
                            <CPU ref="cpuRef" :data="charts[item.Host.Name].cpu" />
                          </div>
                        </a-col>
                        <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                          <div style="margin-bottom: 20px">
                            <Mem
                              ref="memRef"
                              :max="item.Host.MemTotal"
                              :data="charts[item.Host.Name].mem"
                            />
                          </div>
                        </a-col>
                        <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                          <NetIn ref="netInRef" :data="charts[item.Host.Name].net_in" />
                        </a-col>
                        <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                          <NetOut ref="netOutRef" :data="charts[item.Host.Name].net_out" />
                        </a-col>
                      </a-row>
                    </a-col>
                  </a-row>
                </template>
                <n-button
                  circle
                  type="info"
                  secondary
                  class="edit-btn"
                  @click.stop="handleShowEdit(item.Host.Name)"
                >
                  <n-icon>
                    <Edit />
                  </n-icon>
                </n-button>
                <n-button
                  circle
                  type="error"
                  secondary
                  class="delete-btn"
                  @click.stop="handleShowDelete(item.Host.Name)"
                >
                  <n-icon>
                    <Delete />
                  </n-icon>
                </n-button>
              </n-card>
            </div>
          </n-el>
          <n-modal v-model:show="deleteVisible" style="width: 360px" preset="card" title="删除主机">
            <!--              <div class="akile-modal-title">-->
            <!--                <span>删除主机</span>-->
            <!--                <n-button @click="handleClose">-->
            <!--                  <template #icon>-->
            <!--                    <icon-close />-->
            <!--                  </template>-->
            <!--                </n-button>-->
            <!--              </div>-->
            <n-el class="akile-modal-content">
              <n-input
                type="password"
                v-model:value="authSecret"
                placeholder="请输入管理密钥"
              ></n-input>
              <n-text class="tips">提示：删除后无法恢复，请确定后再删除操作</n-text>
            </n-el>
            <n-el class="akile-modal-action">
              <n-button type="error" secondary style="width: 100%" @click="handleDeleteHost">
                确认删除
              </n-button>
            </n-el>
          </n-modal>
          <n-modal
            v-model:show="editVisible"
            style="width: 360px"
            preset="card"
            title="编辑主机信息"
          >
            <!--              <div class="akile-modal-title">-->
            <!--                <span></span>-->
            <!--                <a-button @click="handleEditClose">-->
            <!--                  <template #icon>-->
            <!--                    <icon-close />-->
            <!--                  </template>-->
            <!--                </a-button>-->
            <!--              </div>-->
            <n-el class="akile-modal-content">
              <n-date-picker
                v-model:value="duetime"
                type="datetime"
                clearable
                placeholder="请选择到期时间"
                style="margin-bottom: 10px; width: 100%"
              ></n-date-picker>
              <n-input
                v-model:value="seller"
                placeholder="请输入卖家"
                style="margin-bottom: 10px"
              ></n-input>
              <n-input
                v-model:value="price"
                placeholder="请输入价格"
                style="margin-bottom: 10px"
              ></n-input>
              <n-input
                v-model:value="buy_url"
                placeholder="请输入购买链接"
                style="margin-bottom: 10px"
              ></n-input>
              <n-input
                type="password"
                v-model:value="authSecret"
                placeholder="请输入管理密钥"
              ></n-input>
            </n-el>
            <n-el class="akile-modal-action">
              <n-button type="success" secondary style="width: 100%" @click="handleEditHost">
                更新信息
              </n-button>
            </n-el>
          </n-modal>
          <div class="footer">
            <n-text>Copyright © {{ new Date().getFullYear() }} LoCyanTeam.</n-text>
            <br />
            <n-text style="margin-bottom: 30px">
              Powered by
              <a target="_blank" href="https://github.com/akile-network/akile_monitor">
                Akile Monitor
              </a>
            </n-text>
            <n-text>
              Frontend
              <a target="_blank" href="https://github.com/LoCyan-Team/Natsuka"> Natsuka </a>
            </n-text>
          </div>
        </n-el>
      </n-layout>
    </n-message-provider>
  </n-config-provider>
</template>

<style lang="scss">
body {
  margin: 0;
  //background-color: #fafafa;
  font-family:
    Inter,
    -apple-system,
    BlinkMacSystemFont,
    Roboto,
    PingFang SC,
    Noto Sans CJK,
    WenQuanYi Micro Hei,
    Microsoft YaHei,
    sans-serif;
}

a {
  text-decoration: none;
}

.max-container {
  margin: 0 auto;
  width: 100%;
  max-width: 1440px;
}

.header {
  margin: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;

  .theme-btn {
    //border: 1px solid #eeeeee!important;
    //background-color: #ffffff!important;
    color: #333333 !important;
  }
}

.area-tabs {
  margin: 20px 0;

  .area-tab-item {
    margin-bottom: 10px;
    margin-right: 10px;
    padding: 6px 12px;
    //border-radius: 6px;
    //cursor: pointer;
    //border: 1px solid #e5e5e5;
    //background: #ffffff;
    //box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    display: inline-block;

    .flag-icon {
      border-radius: 3px;
    }

    //&.is-active {
    //  //background: #005fe710;
    //  //color: #054db4;
    //  border: 1px solid #005fe7;
    //}
  }
}

.monitor-card {
  position: relative;
  transition: 1s;
  margin: 15px auto 0;
  //padding: 10px;

  .monitor-item {
    position: relative;
    margin-bottom: 12px;
    //padding: 12px 24px;
    //border-radius: 6px;
    //border: 1px solid #e5e5e5;
    display: block;
    //background: #ffffff;
    //box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    cursor: pointer;

    &.is-active {
      //background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: none !important;
      }

      & > .detail {
        margin-top: 15px;
        border-top: 1px solid #eeeeee;
        padding-top: 15px;
        display: block;
      }
    }

    &:hover {
      //background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: flex;
      }
    }

    .edit-btn {
      right: 60px !important;
      //background: rgba(22, 131, 255, 0.13) !important;

      //&:hover {
      //  background: rgba(22, 131, 255, 0.19) !important;
      //}
    }

    .delete-btn,
    .edit-btn {
      position: absolute;
      right: 20px;
      top: calc(50% - 16px);
      cursor: pointer;
      //background: #ff161620;
      //border-radius: 100px;
      width: 32px;
      height: 32px;
      display: none;
      align-items: center;
      justify-content: center;
      transition: 0.15s background-color ease-in-out;

      //&:hover {
      //  background: #ff161630;
      //}
    }

    .flag-icon {
      margin-right: 5px;
      border-radius: 3px;
      transform: translateY(-2px);
    }

    .monitor-item-title {
      margin-bottom: 3px;
      font-size: 11px;
      opacity: 0.6;
    }

    .monitor-item-value {
      font-weight: 500;
    }

    .monitor-item-progress {
      margin-top: 4px;
      height: 4px;
      display: block;
    }

    .detail {
      width: 100%;
    }

    .name {
      display: inline-block;
      vertical-align: middle;
      width: 250px;

      .title {
        margin-bottom: 5px;
        display: flex;
        align-items: center;
        font-weight: 600;
        font-size: 16px;
      }

      .status {
        display: flex;
        align-items: center;

        &::before {
          margin-right: 10px;
          position: relative;
          display: block;
          content: '';
          width: 8px;
          height: 8px;
          border-radius: 12px;
          background-color: #fb2c36;
        }

        &.online {
          &::before {
            background-color: #00c951;
          }
        }

        span {
          font-size: 13px;
          opacity: 0.6;
        }
      }
    }

    .platform {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .cpu {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .mem {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .average {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .network {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .uptime {
      display: inline-block;
      vertical-align: middle;
      width: 200px;
    }

    .detail {
      display: none;

      .detail-item-list {
        margin-bottom: 20px;
      }

      .detail-item {
        .name {
          width: 20%;
          font-size: 12px;
          color: #666;
          margin-bottom: 5px;
          display: inline-block;
          vertical-align: top;
        }

        .value {
          width: 80%;
          font-size: 12px;
          font-weight: 500;
          display: inline-block;
          vertical-align: top;
        }
      }
    }
  }
}

.logo {
  margin-top: 20px;
  margin-bottom: 20px;
  position: relative;
  cursor: pointer;
  line-height: 54px;
  height: 54px;
  font-size: 16px;
  font-weight: 900;
  color: #333;
  display: flex;
  align-items: center;

  .icon {
    margin-right: 5px;
    height: 28px;
    width: 28px;
    color: rgb(22, 93, 255) !important;
  }
}

.monitor-action-bar {
  margin: 0 10px;
  display: inline-block;
  border: 1px solid #e5e5e5;
  background: #ffffff;
  box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
  border-radius: 4px;

  :deep(.arco-tabs-content) {
    display: none;
  }
}

.footer {
  margin-block: 2rem;
  line-height: 22px;
  text-align: center;
}

.akile-modal-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 20px;
}

.akile-modal-content {
  margin-bottom: 20px;

  .tips {
    font-size: 12px;
    color: #333333;
    margin-top: 10px;
  }
}

//
//body[arco-theme='dark'] {
//  background-color: #111111;
//
//  .arco-modal {
//    background-color: #0e0e0e;
//    border: 1px solid rgba(255,255,255,0.05);
//  }
//
//  .header {
//    .logo {
//      span,
//      small {
//        color: #ffffff;
//      }
//    }
//
//    .theme-btn {
//      border: 1px solid #333333!important;
//      background-color: #000000!important;
//      color: #ffffff!important;
//    }
//  }
//
//  .area-tabs {
//    .area-tab-item {
//      border-color: #333333;
//      background: #000000;
//      color: #ffffff;
//      box-shadow: none;
//
//      &.is-active {
//        background: #005fe705;
//        color: #005fe7;
//        border: 1px solid #005fe7;
//      }
//    }
//  }
//
//  .footer {
//    color: #ffffffAA;
//  }
//
//  .monitor-card {
//    .monitor-item {
//      border: 1px solid rgb(255 255 255 / 16%);
//      box-shadow: none;
//      background-color: #000000;
//      color: #ffffff;
//
//      &:hover {
//        background-color: #101010;
//      }
//
//      .detail {
//        border-color: #333333AA;
//        .detail-item-list {
//          .detail-item {
//            .name {
//              color: #999999;
//            }
//          }
//        }
//      }
//    }
//  }
//
//}

@media screen and (max-width: 768px) {
  .monitor-item {
    & > div {
      margin-bottom: 10px;
    }
  }

  .detail {
    .detail-item {
      .name {
        width: 25% !important;
      }

      .value {
        width: 75% !important;
      }
    }
  }
}

@media screen and (max-width: 1920px) {
  .max-container {
    max-width: 1440px;
  }
}

@media screen and (max-width: 1280px) {
  .max-container {
    max-width: 1140px;
  }
}

@media screen and (max-width: 992px) {
  .max-container {
    max-width: 960px;
  }
}

@media screen and (max-width: 768px) {
  .max-container {
    max-width: 720px;
  }
}

@media screen and (max-width: 576px) {
  .max-container {
    max-width: 540px;
  }
}
</style>
