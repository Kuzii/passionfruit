<template>
  <div>
    <b-modal :active="isDownloading" :width="640" scroll="keep" :canCancel="false">
      <article class="download-progress">
        <loading></loading>
        <strong>Preparing {{ downloadProgress }}%</strong>
      </article>
    </b-modal>

    <header class="hero">
      <div class="level container is-fluid">
        <nav class="breadcrumb nav-bar level-left" aria-label="breadcrumbs">
          <ul class="level-item">
            <li>
              <a href="/"><img width="133.33" height="32" src="../assets/logo.svg" alt="Passionfruit"></a>
            </li>
            <li v-if="err">Unknown device</li>
            <li v-else>
              <router-link v-if="device.id" :to="{name: 'apps', params: {device: device.id}}">
                <icon :icon="device.icon"></icon> {{ device.name }}</router-link>
            </li>
            <li v-if="err">Unknown App</li>
            <li v-else class="is-active">
              <a href="#" v-if="app" aria-current="page">
                <icon :icon="app.smallIcon"></icon> {{ app.name }}</a>
              <div class="tags has-addons">
                <span class="tag is-dark">{{ app.identifier }}</span>
                <span class="tag is-success" v-if="app.pid">pid: {{ app.pid }}</span>
              </div>
            </li>
          </ul>

          <a class="button is-small" @click="screenshot"><b-icon icon="camera"
            title="Capture screen"></b-icon></a>
        </nav>

        <div class="level-right">
          <nav class="level-item field has-addons">
            <p class="control">
              <b-dropdown position="is-bottom-left">
                <a class="button" slot="trigger">
                  <span class="is-size-7">Manage Hooks</span>
                  <b-icon icon="call_split"></b-icon>
                </a>
                <b-dropdown-item custom>
                  <ul class="hooks">
                    <li v-for="(hook, index) in hooks" :key="index">
                      <a class="delete is-pulled-right is-danger" @click="removeHook(index)"></a>
                      <span v-if="hook.type == 'dylib'">{{ hook.module }}!{{ hook.name }}</span>
                      <span v-else>{{ hook.clazz }}|{{ hook.method }}</span>
                    </li>
                  </ul>
                </b-dropdown-item>
              </b-dropdown>
            </p>
            <p class="control">
              <b-tooltip label="Kill Process" position="is-left">
                <button class="button is-danger" @click="kill">
                  <b-icon icon="power_settings_new"></b-icon>
                </button>
              </b-tooltip>
            </p>
          </nav>
        </div>
      </div>

      <div class="container is-fluid">
        <nav v-if="connected" class="tabs is-centered is-fullwidth is-boxed">
          <ul>
            <li>
              <router-link :to="{ name: 'general' }">
                <b-icon icon="dashboard"></b-icon>
                <span>General</span>
              </router-link>
            </li>
            <li>
              <router-link :to="{ name: 'files' }">
                <b-icon icon="folder_special"></b-icon>
                <span>Files</span>
              </router-link>
            </li>
            <li>
              <router-link :to="{ name: 'modules' }">
                <b-icon icon="view_module"></b-icon>
                <span>Modules</span>
              </router-link>
            </li>
            <li>
              <router-link :to="{ name: 'classes' }">
                <b-icon icon="gavel"></b-icon>
                <span>Classes</span>
              </router-link>
            </li>
            <li>
              <router-link :to="{ name: 'console' }">
                <b-icon icon="announcement"></b-icon>
                <span>Console</span>
                <b-tag rounded v-show="unreadMessage" type="is-info">{{ unreadMessage }}</b-tag>
              </router-link>
            </li>
            <li>
              <router-link :to="{ name: 'uidump' }">
                <b-icon icon="visibility"></b-icon>
                <span>UIDump</span>
              </router-link>
            </li>
            <li>
              <router-link :to="{ name: 'storage' }">
                <b-icon icon="storage"></b-icon>
                <span>Storage</span>
              </router-link>
            </li>
          </ul>
        </nav>
      </div>
    </header>

    <section class="container section is-fluid" v-if="err">
      <b-message type="is-danger" has-icon>{{ err }}</b-message>
    </section>
    <b-loading :active="loading" :canCancel="true" @cancel="home"></b-loading>

    <div v-if="connected" class="container is-fluid">
      <section class="tab-content main">
        <router-view class="tab-item"></router-view>
      </section>
    </div>
  </div>
</template>

<script>

import io from 'socket.io-client'
import { mapGetters, mapActions, mapMutations } from 'vuex'
import { AsyncSearch, debounce } from '~/lib/utils'
import {
  GET_SOCKET, STORE_SOCKET,
  CONSOLE_UNREAD, CONSOLE_APPEND, CONSOLE_CLEAR,
  DOWNLOADING, PROGRESS,
  ALL_HOOKS, DELETE_HOOK,
} from '~/vuex/types'

import Loading from '~/components/Loading.vue'
import Icon from '~/components/Icon.vue'

export default {
  components: { Icon, Loading },
  watch: {
    app(val, old) {
      if (val.name)
        document.title = `Passionfruit: ${val.name}`
    },
  },
  computed: {
    ...mapGetters({
      socket: GET_SOCKET,
      hooks: ALL_HOOKS,
      unreadMessage: CONSOLE_UNREAD,
      isDownloading: DOWNLOADING,
      downloadProgress: PROGRESS,
    })
  },
  methods: {
    home() {
      this.$route.push({ name: 'welcome' })
    },
    kill() {
      this.$dialog.confirm({
        title: 'Kill App',
        message: 'Are you sure you want to <b>kill</b> the process? The session will end.',
        confirmText: 'Kill',
        type: 'is-danger',
        hasIcon: true,
        onConfirm: () => {
          this.$router.push({ name: 'apps', params: this.$route.params })
          this.socket.call('kill').then(result => {
            if (result) {
              this.$toast.open(`${bundle} has been terminiated`)
            }
          })
        }
      })
    },
    createSocket() {
      let { device, bundle } = this.$route.params
      return io('/session', { path: '/msg', query: { device, bundle } })
        .on('attached', console.info.bind(console))
        .on('close', console.warn.bind(console))
        .on('disconnect', () => {
          this.$toast.open(`disconnected from ${bundle}`)
          this.err = 'Application disconnected. Reload the page to retry, or select another app in main menu.'
          this.connected = false
          this.loading = false
        })
        .on('unhandledRejection', ({ err, stack }) => {
          if (this.loading) {
            this.$toast.open(`disconnected from ${bundle}`)
            this.connected = false
            this.loading = false
            this.err = err
          } else {
            this.$toast.open(`an exception has occured: ${err}`)
          }
        })
        .on('console', this.consoleAppend)
        .on('app', ({ app, device }) => {
          this.device = device 
          this.app = app

          // todo: store in vuex
          // this.storeDeviceId(device.id)
          // this.storeAppId(app.identifier)
        })
        .on('ready', () => {
          this.loading = false
          this.connected = true
        })
        .on('err', err => {
          this.err = err
          this.loading = false
        })
    },
    async screenshot() {
      const b64 = await this.socket.call('screenshot')
      const url = `data:image/png;base64,${b64}`
      let link = document.createElement('a')
      link.setAttribute('href', url)
      link.setAttribute('download', name)
      link.click()
    },
    rejectionHandler(event) {
      event.preventDefault()
      this.$toast.open({
        duration: 10 * 1000,
        message: `unhandled rejection: ${event.reason}`,
        type: 'is-danger',
      })
    },
    ...mapActions({
      removeHook: DELETE_HOOK,
    }),
    ...mapMutations({
      storeSocket: STORE_SOCKET,
      consoleAppend: CONSOLE_APPEND,
      consoleClear: CONSOLE_CLEAR,
    })
  },
  data() {
    return {
      err: '',
      loading: true,
      connected: false,
      app: {},
      device: {},
    }
  },
  mounted() {
    const socket = this.createSocket()
    this.storeSocket(socket)
    window.addEventListener('unhandledrejection', this.rejectionHandler)
    this.consoleClear()
  },
  beforeDestroy() {
    if (this.socket)
      this.socket.call('detach')
    window.removeEventListener('unhandledrejection', this.rejectionHandler)
  },
}
</script>

<style lang="scss" scoped>
.breadcrumb {
  margin: 10px 0;

  canvas {
    margin-right: 4px;
  }
}

.search {
  margin-top: 0;
}

ul.hooks {
  li {
    white-space: nowrap;

    .delete {
      cursor: pointer;
    }
  }
}

.tab-content.main {
  margin-top: 20px;
}

.hero {
  background: whitesmoke;
  .level {
    margin-bottom: 0;
  }

  .tabs.is-boxed a {
    color: #7d7d7d;
    &.is-active {
      background: #fff;
      color: #222;

      &:hover {
        background: #fff;
      }
    }
    &:hover {
      background: #efefef;
      color: #222;
    }
  }
}

.download-progress {
  margin: auto;
  color: #fff;
  text-align: center;

  strong {
    color: #fff;
    font-size: 24px;
    font-weight: normal;
  }
}
</style>
