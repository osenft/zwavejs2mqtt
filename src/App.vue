<template>
  <v-app :dark="dark">
    <div v-if="$route.meta.requiresAuth && auth !== undefined">
      <v-navigation-drawer
        clipped-left
        :mini-variant="mini"
        v-model="drawer"
        app
      >
        <v-list nav class="py-0">
          <v-list-item :class="mini && 'px-0'">
            <v-list-item-avatar>
              <img
                style="padding:3px;border-radius:0"
                :src="`${baseURI}/static/logo.png`"
              />
            </v-list-item-avatar>
            <v-list-item-content>
              <v-list-item-title>{{ 'ZWaveJS2MQTT' }}</v-list-item-title>
            </v-list-item-content>
          </v-list-item>
        </v-list>
        <v-divider style="margin-top:8px"></v-divider>
        <v-list nav>
          <v-list-item
            v-for="item in pages"
            :key="item.title"
            :to="item.path == '#' ? '' : item.path"
            :color="item.path === $route.path ? 'primary' : ''"
          >
            <v-list-item-action>
              <v-icon>{{ item.icon }}</v-icon>
            </v-list-item-action>
            <v-list-item-content>
              <v-list-item-title class="subtitle-2 font-weight-bold">{{
                item.title
              }}</v-list-item-title>
            </v-list-item-content>
          </v-list-item>
          <v-list-item v-if="!mini">
            <v-switch label="Dark theme" hide-details v-model="dark"></v-switch>
          </v-list-item>
        </v-list>
        <v-footer absolute v-if="!mini" class="pa-3">
          <div>Innovation System &copy; {{ new Date().getFullYear() }}</div>
        </v-footer>
      </v-navigation-drawer>

      <v-app-bar app>
        <v-app-bar-nav-icon @click.stop="toggleDrawer" />
        <v-toolbar-title>{{ title }}</v-toolbar-title>

        <v-spacer></v-spacer>

        <v-tooltip bottom>
          <template v-slot:activator="{ on }">
            <v-icon
              dark
              medium
              style="cursor:default;"
              :color="statusColor || 'primary'"
              v-on="on"
              >swap_horizontal_circle</v-icon
            >
          </template>
          <span>{{ status }}</span>
        </v-tooltip>

        <div v-if="auth">
          <v-menu v-if="$vuetify.breakpoint.xsOnly" bottom left>
            <template v-slot:activator="{ on }">
              <v-btn v-on="on" icon>
                <v-icon>more_vert</v-icon>
              </v-btn>
            </template>

            <v-list>
              <v-list-item
                v-for="(item, i) in menu"
                :key="i"
                @click="item.func"
              >
                <v-list-item-action>
                  <v-icon>{{ item.icon }}</v-icon>
                </v-list-item-action>
                <v-list-item-title>{{ item.tooltip }}</v-list-item-title>
              </v-list-item>
            </v-list>
          </v-menu>

          <div v-else>
            <v-menu v-for="item in menu" :key="item.text" bottom left>
              <template v-slot:activator="{ on }">
                <v-btn v-on="on" icon @click="item.func">
                  <v-tooltip bottom>
                    <template v-slot:activator="{ on }">
                      <v-icon dark color="primary" v-on="on">{{
                        item.icon
                      }}</v-icon>
                    </template>
                    <span>{{ item.tooltip }}</span>
                  </v-tooltip>
                </v-btn>
              </template>

              <v-list v-if="item.menu">
                <v-list-item
                  v-for="(menu, i) in item.menu"
                  :key="i"
                  @click="menu.func"
                >
                  <v-list-item-title>{{ menu.title }}</v-list-item-title>
                </v-list-item>
              </v-list>
            </v-menu>
          </div>
        </div>
      </v-app-bar>
    </div>
    <main style="height:100%">
      <v-main style="height:100%">
        <router-view
          v-if="auth !== undefined"
          @import="importFile"
          @export="exportConfiguration"
          @showConfirm="confirm"
          @apiRequest="apiRequest"
          :socket="socket"
        />
        <v-row style="height:100%" align="center" justify="center" v-else>
          <v-col align="center">
            <div class="text-h2 ma-5">{{ error ? error : 'Loading...' }}</div>
            <v-progress-circular
              v-if="!error"
              size="200"
              indeterminate
            ></v-progress-circular>
            <v-btn text @click="checkAuth" v-else
              >Retry <v-icon right dark>refresh</v-icon></v-btn
            >
          </v-col>
        </v-row>
      </v-main>
    </main>

    <PasswordDialog
      @updatePassword="updatePassword()"
      @close="closePasswordDialog()"
      :show="dialog_password"
      :password="password"
    />

    <Confirm ref="confirm"></Confirm>

    <v-snackbar
      :timeout="3000"
      :bottom="true"
      :multi-line="false"
      :vertical="false"
      v-model="snackbar"
    >
      {{ snackbarText }}
      <v-btn text @click="snackbar = false">Close</v-btn>
    </v-snackbar>
  </v-app>
</template>

<style>
/* Fix Vuetify code style after update to 2.4.0 */
code {
  color: #c62828 !important;
  font-weight: 700 !important;
}
</style>

<script>
// https://github.com/socketio/socket.io-client/blob/master/docs/API.md
import io from 'socket.io-client'
import ConfigApis from '@/apis/ConfigApis'
import Confirm from '@/components/Confirm'
import PasswordDialog from '@/components/dialogs/Password'
import { Settings } from '@/modules/Settings'
import { Routes } from '@/router'

import { mapActions, mapMutations, mapGetters } from 'vuex'

import { socketEvents, inboundEvents as socketActions } from '@/plugins/socket'

export default {
  components: {
    PasswordDialog,
    Confirm
  },
  name: 'app',
  computed: {
    ...mapGetters(['user', 'auth'])
  },
  watch: {
    $route: function (value) {
      this.title = value.name || ''
      this.startSocket()
    },
    dark (v) {
      this.settings.store('dark', this.dark)

      this.$vuetify.theme.dark = v
      this.changeThemeColor()
    }
  },
  data () {
    return {
      socket: null,
      error: false,
      dialog_password: false,
      password: {},
      menu: [
        {
          icon: 'logout',
          func: this.logout,
          tooltip: 'Logout'
        },
        {
          icon: 'lock',
          func: this.showPasswordDialog,
          tooltip: 'Password'
        }
      ],
      pages: [
        { icon: 'widgets', title: 'Control Panel', path: Routes.controlPanel },
        { icon: 'settings', title: 'Settings', path: Routes.settings },
        { icon: 'movie_filter', title: 'Scenes', path: Routes.scenes },
        { icon: 'bug_report', title: 'Debug', path: Routes.debug },
        { icon: 'folder', title: 'Store', path: Routes.store },
        { icon: 'share', title: 'Network graph', path: Routes.mesh }
      ],
      settings: new Settings(localStorage),
      status: '',
      statusColor: '',
      drawer: false,
      mini: false,
      topbar: [],
      title: '',
      snackbar: false,
      snackbarText: '',
      dark: undefined,
      baseURI: ConfigApis.getBasePath()
    }
  },
  methods: {
    ...mapActions(['initNodes', 'setAppInfo', 'updateValue', 'removeValue']),
    ...mapMutations(['setControllerStatus', 'initNode', 'removeNode']),
    async updatePassword () {
      try {
        const response = await ConfigApis.updatePassword(this.password)
        this.showSnackbar(response.message)
        if (response.success) {
          this.closePasswordDialog()
          this.$store.dispatch('setUser', response.user)
        }
      } catch (error) {
        this.showSnackbar(
          'Error while updating password, check console for more info'
        )
        console.log(error)
      }
    },
    closePasswordDialog () {
      this.dialog_password = false
    },
    showPasswordDialog () {
      this.password = {}
      this.dialog_password = true
    },
    toggleDrawer () {
      if (['xs', 'sm', 'md'].indexOf(this.$vuetify.breakpoint.name) >= 0) {
        this.mini = false
        this.drawer = !this.drawer
      } else {
        this.mini = !this.mini
        this.drawer = true
      }
    },
    async confirm (title, text, level, options) {
      options = options || {}

      const levelMap = {
        warning: 'orange',
        alert: 'red'
      }

      options.color = levelMap[level] || 'primary'

      return this.$refs.confirm.open(title, text, options)
    },
    showSnackbar: function (text) {
      this.snackbarText = text
      this.snackbar = true
    },
    apiRequest (apiName, args) {
      if (this.socket.connected) {
        const data = {
          api: apiName,
          args: args
        }
        this.socket.emit(socketActions.zwave, data)
      } else {
        this.showSnackbar('Socket disconnected')
      }
    },
    updateStatus: function (status, color) {
      this.status = status
      this.statusColor = color
    },
    changeThemeColor: function () {
      const metaThemeColor = document.querySelector('meta[name=theme-color]')
      const metaThemeColor2 = document.querySelector(
        'meta[name=msapplication-TileColor]'
      )

      metaThemeColor.setAttribute('content', this.dark ? '#000' : '#fff')
      metaThemeColor2.setAttribute('content', this.dark ? '#000' : '#fff')
    },
    importFile: function (ext) {
      const self = this
      // Check for the various File API support.
      return new Promise(function (resolve, reject) {
        if (
          window.File &&
          window.FileReader &&
          window.FileList &&
          window.Blob
        ) {
          const input = document.createElement('input')
          input.type = 'file'
          input.addEventListener('change', function (event) {
            const files = event.target.files

            if (files && files.length > 0) {
              const file = files[0]
              const reader = new FileReader()

              reader.addEventListener('load', function (fileReaderEvent) {
                let err
                let data = fileReaderEvent.target.result

                if (ext === 'json') {
                  try {
                    data = JSON.parse(data)
                  } catch (e) {
                    self.showSnackbar(
                      'Error while parsing input file, check console for more info'
                    )
                    console.error(e)
                    err = e
                  }
                }

                if (err) {
                  reject(err)
                } else {
                  resolve({ data, file })
                }
              })

              if (ext === 'buffer') {
                reader.readAsArrayBuffer(file)
              } else {
                reader.readAsText(file)
              }
            }
          })

          input.click()
        } else {
          reject(Error('Unable to load file in this browser'))
        }
      })
    },
    exportConfiguration: function (data, fileName, ext) {
      ext = ext || 'json'
      const textMime = ['json', 'jsonl', 'txt', 'log', 'js', 'ts']
      const contentType = textMime.includes(ext)
        ? 'text/plain'
        : 'application/octet-stream'
      const a = document.createElement('a')

      data =
        ext === 'json' && typeof data === 'object' ? JSON.stringify(data) : data

      const blob = new Blob([data], {
        type: contentType
      })

      document.body.appendChild(a)
      a.href = window.URL.createObjectURL(blob)
      a.download = fileName + '.' + (ext || 'json')
      a.target = '_self'
      a.click()
    },
    async startSocket () {
      if (
        this.auth === undefined ||
        this.socket ||
        !this.$route.meta ||
        !this.$route.meta.requiresAuth
      ) {
        return
      }

      if (this.auth && (!this.user || !this.user.token)) {
        await this.logout()
      }

      const query = this.auth ? { token: this.user.token } : undefined

      this.socket = io('/', {
        path: ConfigApis.getSocketPath(),
        query: query
      })

      this.socket.on('connect', () => {
        this.updateStatus('Connected', 'green')
      })

      this.socket.on('disconnect', () => {
        this.updateStatus('Disconnected', 'red')
      })

      this.socket.on('error', () => {
        console.log('Socket error')
      })

      this.socket.on('reconnecting', () => {
        this.updateStatus('Reconnecting', 'yellow')
      })

      this.socket.on(socketEvents.init, data => {
        // convert node values in array
        this.initNodes(data.nodes)
        this.setControllerStatus(data.error ? data.error : data.cntStatus)
        this.setAppInfo(data.info)
      })

      this.socket.on(socketEvents.connected, this.setAppInfo.bind(this))
      this.socket.on(
        socketEvents.controller,
        this.setControllerStatus.bind(this)
      )

      this.socket.on(socketEvents.nodeUpdated, this.initNode.bind(this))
      this.socket.on(socketEvents.nodeRemoved, this.removeNode.bind(this))

      this.socket.on(socketEvents.valueRemoved, this.removeValue.bind(this))
      this.socket.on(socketEvents.valueUpdated, this.updateValue.bind(this))

      this.socket.emit(socketActions.init, true)
    },
    async logout () {
      const user = Object.assign({}, this.user)
      localStorage.setItem('user', JSON.stringify(user))
      localStorage.removeItem('logged')

      if (this.socket) {
        this.socket.close()
        this.socket = null
      }

      if (this.auth) {
        try {
          await ConfigApis.logout()
          this.showSnackbar('Logged out')
        } catch (error) {
          this.showSnackbar('Logout failed')
        }

        if (this.$route.path !== Routes.login) {
          this.$router.push(Routes.login)
        }
      }
    },
    // get config, used to check if gateway is used with auth or not
    async checkAuth () {
      this.error = false
      try {
        const data = await ConfigApis.isAuthEnabled()
        if (!data.success) {
          throw Error(data.message || 'Error while checking authorizations')
        } else {
          const newAuth = data.data === true
          const oldAuth = this.auth

          this.$store.dispatch('setAuth', newAuth)

          if (oldAuth !== undefined && oldAuth !== newAuth) {
            await this.logout()
          }

          if (!newAuth && this.$route.path === Routes.login) {
            this.$router.push(Routes.main)
          }
          this.startSocket()
        }
      } catch (error) {
        setTimeout(() => (this.error = error.message), 1000)
        console.log(error)
      }
    }
  },
  beforeMount () {
    this.title = this.$route.name || ''
    this.checkAuth()
  },
  mounted () {
    if (this.$vuetify.breakpoint.lg || this.$vuetify.breakpoint.xl) {
      this.toggleDrawer()
    }

    this.dark = this.settings.load('dark', false)
    this.changeThemeColor()

    this.$store.subscribe(mutation => {
      if (mutation.type === 'showSnackbar') {
        this.showSnackbar(mutation.payload)
      } else if (mutation.type === 'initSettings') {
        // check if auth is changed in settings
        this.checkAuth()
      }
    })
  },
  beforeDestroy () {
    if (this.socket) this.socket.close()
  }
}
</script>
