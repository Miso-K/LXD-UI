<template>
  <div>
    <!-- Main header -->
    <main-header :current="$route.path"></main-header>

    <!-- Main element -->
    <el-main>
      <h6 class="title is-5">
        Containers
        <div v-if="containers.length > 0" class="is-pulled-right">
          <button
                  class="button is-small is-info" 
                  @click="refresh_containers()" 
                  :class="{ 'is-loading': btn.refresh_containers }" 
                  :disabled="btn.refresh_containers">
            <span class="icon">
              <i class="fa fa-refresh"></i> 
            </span>
            <span>Refresh</span>
          </button>
          <button v-if="!all_stopped"
                  style="margin-left:5px"
                  class="button is-small is-danger" 
                  @click="stop_containers()" 
                  :class="{ 'is-loading': btn.stop_containers }" 
                  :disabled="btn.stop_containers">
            <span class="icon">
              <i class="fa fa-stop"></i> 
            </span>
            <span>Stop All</span>
          </button>
          <button v-else
                  style="margin-left:5px"
                  class="button is-small is-success" 
                  @click="start_containers()" 
                  :class="{ 'is-loading': btn.start_containers }" 
                  :disabled="btn.start_containers">
            <span class="icon">
              <i class="fa fa-stop"></i> 
            </span>
            <span>Start All</span>
          </button>
        </div>
      </h6>
      <div class="box">
        <div class="card-content" v-loading="loading">
          <div v-if="containers.length > 0">
            <table class="table is-fullwidth is-narrow">
              <thead>
                <tr>
                  <th style="width:22.5%">Name</th>
                  <th style="width:1%"></th>
                  <th style="width:12.5%">IP</th>
                  <th style="width:13%">CPU</th>
                  <th style="width:12%">Memory</th>
                  <th style="width:10%">Status</th>
                  <th style="width:1%"></th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(container, index) in containers">
                  <td>
                    <edit-container :name="container.name" 
                                    @on-save="refresh_containers" 
                                    @container-stopped="refresh_containers"
                                    @clicked="manage_dropdown_close_all">
                      {{ container.name }}
                    </edit-container>
                  </td>
                  <td>
                    <div style="display:flex" class="has-text-left">
                      <a v-if="check_started_with_ip(container) && container.services.vnc" @click="openCmd('xterm -e vncviewer -quality 5 -encodings \'copyrect tight hextile zlib corre rre raw\' -compresslevel 5 ' + container.state.network.eth0.addresses[0].address + ':5901')">
                        <span class="icon has-text-info is-pulled-right" title="VNC">
                          <i class="fa fa-desktop"></i>
                        </span>
                      </a>
                      <a v-if="check_started_with_ip(container) && container.services.ssh" @click="open('ssh://root@' + container.state.network.eth0.addresses[0].address)">
                        <span class="icon has-text-info is-pulled-right" title="SSH">
                          <i class="fa fa-folder-open"></i>
                        </span>
                      </a>
                      <a v-if="check_started_with_ip(container) && container.services.http" @click="open('http://' + container.state.network.eth0.addresses[0].address)">
                        <span class="icon has-text-info is-pulled-right" title="HTTP">
                          <i class="fa fa-globe"></i>
                        </span>
                      </a>
                      <a v-if="check_started_with_ip(container) && check_ngrok(container)" @click="open(get_ngrok(container))">
                        <span class="icon has-text-info is-pulled-right" title="HTTP (ngrok)">
                          <i class="fa fa-share-alt"></i>
                        </span>
                      </a>
                    </div>
                  </td>
                  <td>
                    <!-- Running is ip4 -->
                    <a 
                       v-if="check_started_with_ip(container)" 
                       @click="open('http://' + container.state.network.eth0.addresses[0].address)">
                      {{ container.state && container.state.network ? container.state.network.eth0.addresses[0].address : '-' }}
                    </a>
                    <!-- Running not ip4 -->
                    <span 
                          v-if="container.state &&
                                container.status === 'Running' && 
                                (container.state.network.eth0.addresses.length === 0 || isIP4(container.state.network.eth0.addresses[0].address) === false)"
                          @click="refresh_containers()">
                      <i class="fa fa-refresh"></i>
                    </span>
                    <!-- Stopped -->
                    <span v-show="container.status === 'Stopped'">-</span>
                  </td>
                  <td>{{ container.state && container.state.cpu.usage !== 0 ? Number(container.state.cpu.usage/1000000000).toFixed(2) + ' seconds' : '-' }}</td>
                  <td>{{ container.state && container.state.memory.usage !== 0 ? formatBytes(container.state.memory.usage) : '-' }}</td>
                  <td>
                    <strong :class="{'has-text-success': (container.status === 'Running'), 'has-text-danger': (container.status === 'Stopped')}">{{ container.status }}</strong>
                  </td>
                  <td>
                    <div class="dropdown is-small is-right" :class="manage_dropdown_class(container.name)">
                      <div class="dropdown-trigger" @click="manage_dropdown_toggle(container.name)">
                        <button class="button is-info is-small">
                          <span>Manage</span>
                          <span class="icon is-small">
                            <i class="fa fa-angle-down"></i>
                          </span>
                        </button>
                      </div>
                      <div class="dropdown-menu" id="dropdown-menu3" role="menu">
                        <div class="dropdown-content">
                          <router-link class="dropdown-item" v-if="container.status !== 'Error' && container.status === 'Running'" :to="{ path: '/terminal/' + container.name + '/' + container.config['image.os'] }" target="_blank" v-on:click.native="manage_dropdown_close_all()" title="Open terminal">
                            <span class="icon">
                              <i class="fa fa-terminal"></i> 
                            </span>
                            <span>Terminal</span>
                          </router-link>
                          <a @click="start_container(container.name)" class="dropdown-item" v-if="container.status === 'Stopped'">
                            <span class="icon">
                              <i class="fa fa-play"></i> 
                            </span>
                            <span>Start</span>
                          </a>
                          <a @click="stop_container(container.name)" class="dropdown-item" v-if="container.status === 'Running'">
                            <span class="icon">
                              <i class="fa fa-stop"></i> 
                            </span>
                            <span>Stop</span>
                          </a>
                          <a @click="freeze_container(container.name)" class="dropdown-item" v-if="container.status === 'Running'">
                            <span class="icon">
                              <i class="fa fa-pause"></i> 
                            </span>
                            <span>Freeze</span>
                          </a>
                          <a @click="thaw_container(container.name)" class="dropdown-item" v-if="container.status === 'Frozen'">
                            <span class="icon">
                              <i class="fa fa-play"></i> 
                            </span>
                            <span>Thaw</span>
                          </a>
                          <a @click="restart_container(container.name)" class="dropdown-item" v-if="container.status === 'Running'">
                            <span class="icon">
                              <i class="fa fa-refresh"></i> 
                            </span>
                            <span>Restart</span>
                          </a>
                          <a @click="start_ngrok(container.state.network.eth0.addresses[0].address)" class="dropdown-item" v-if="check_started_with_ip(container) && !check_ngrok(container) && container.status === 'Running'">
                            <span class="icon">
                              <i class="fa fa-share-alt"></i> 
                            </span>
                            <span>Ngrok Start</span>
                          </a>
                          <a @click="stop_ngrok()" class="dropdown-item" v-if="check_started_with_ip(container) && check_ngrok(container) && container.status === 'Running'">
                            <span class="icon">
                              <i class="fa fa-share-alt"></i> 
                            </span>
                            <span>Ngrok Stop</span>
                          </a>
                          <a @click="snapshot_container(container.name)" class="dropdown-item" v-if="container.status === 'Stopped'">
                            <span class="icon">
                              <i class="fa fa-cubes"></i> 
                            </span>
                            <span>Snapshot</span>
                          </a>
                          <a @click="image_container(container.name, index)" class="dropdown-item" v-if="container.status === 'Stopped'">
                            <span class="icon">
                              <i class="fa fa-hdd-o"></i> 
                            </span>
                            <span>Image</span>
                          </a>
                          <hr class="dropdown-divider">
                          <a @click="copy_ssh_key(container.name)" 
                             class="dropdown-item" 
                             v-show="check_started_with_ip(container) && container.services.ssh">
                            <span class="icon">
                              <i class="fa fa-key"></i> 
                            </span>
                            <span>Copy SSH Key</span>
                          </a>
                          <a @click="delete_container(container.name)" class="dropdown-item" v-if="container.status === 'Stopped' && !locked_container(container.name)">
                            <span class="icon">
                              <i class="fa fa-times"></i> 
                            </span>
                            <span>Delete</span>
                          </a>
                          <a @click="lock_container(container.name)" class="dropdown-item" v-if="!locked_container(container.name)">
                            <span class="icon">
                              <i class="fa fa-lock"></i> 
                            </span>
                            <span>Lock</span>
                          </a>
                          <a @click="unlock_container(container.name)" class="dropdown-item" v-if="locked_container(container.name)">
                            <span class="icon">
                              <i class="fa fa-lock"></i> 
                            </span>
                            <span>UnLock</span>
                          </a>
                        </div>
                      </div>
                    </div>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
          <div v-else>
            <span v-if="!loading">No containers, you can <router-link :to="{ name: 'images' }">launch containers from images</router-link>.</span>
            <span v-else>&nbsp;</span>
          </div>
        </div>
      </div>
    </el-main>
  </div>
</template>

<script>
  // import _ from 'lodash'
  import moment from 'moment'
  import helpers from '../mixins/helpers.js'
  import lxc from '../mixins/lxc.js'
  import MainHeader from './Layout/MainHeader'
  import EditContainer from './Containers/EditContainer'

  import ElectronStore from 'electron-store'
  const storage = new ElectronStore({
    cwd: 'lxd-ui'
  })

  const checkPort = require('./Containers/CheckPort.js').default
  const hash = require('hash.js')
  const ngrok = require('ngrok')

  export default {
    name: 'containers-page',
    components: { MainHeader, EditContainer },
    mixins: [lxc, helpers],
    data () {
      return {
        loading: true,
        manageButton: [],
        search: null,
        search_result: null,
        containers: [],
        btn: {
          refresh_containers: false
        }
      }
    },
    computed: {
      all_stopped: function () {
        var running = this.containers.filter((row) => {
          if (row.status !== 'Stopped') {
            return row
          }
          return false
        })
        return running.length === 0
      }
    },
    mounted: function () {
      document.title = 'LXDui - Containers'
      this.$nextTick(() => {
        this.get_containers(() => {
          this.loading = false
        })
      })
    },
    methods: {
      /**
       * [Manage] Dropdown button methods
       */
      manage_dropdown_close_all (name) {
        for (var prop in this.manageButton) {
          if (!name || (prop !== name && this.manageButton[prop] === true)) {
            this.manageButton[prop] = false
          }
        }
      },
      manage_dropdown_toggle (name) {
        this.manage_dropdown_close_all(name)
        this.$set(this.manageButton, name, !this.manageButton[name])
      },
      manage_dropdown_class (name) {
        return {
          'is-active': this.manageButton[name]
        }
      },
      /**
       * Checks a containers ip for an open port
       */
      check_http (ip, index) {
        checkPort(80, {host: ip}).then((reachable) => {
          this.containers[index].services.http = reachable
        })
      },
      /**
       * Checks a containers ip for an open port
       */
      check_ssh (ip, index) {
        checkPort(22, {host: ip}).then((reachable) => {
          this.containers[index].services.ssh = reachable
        })
      },
      /**
       * Checks a containers ip for an open port
       */
      check_vnc (ip, index) {
        checkPort(5901, {host: ip}).then((reachable) => {
          this.containers[index].services.vnc = reachable
        })
      },
      /**
       *
       */
      check_ngrok (container) {
        return storage.has('ngrok.' + hash.sha256().update(container.state.network.eth0.addresses[0].address).digest('hex'))
      },
      /**
       *
       */
      get_ngrok (container) {
        return storage.get('ngrok.' + hash.sha256().update(container.state.network.eth0.addresses[0].address).digest('hex'))
      },
      /**
       *
       */
      start_ngrok (ip) {
        //
        this.manage_dropdown_close_all()
        storage.delete('ngrok')
        var self = this
        async function startNgrok () {
          await ngrok.kill()
          const url = await ngrok.connect({
            proto: 'http',
            addr: ip + ':80',
            binPath: function (dir) {
              return dir.replace('app.asar', 'app.asar.unpacked')
            }
          })
          storage.set('ngrok.' + hash.sha256().update(ip).digest('hex'), url)
          //
          self.get_containers()
        }
        startNgrok()
      },
      /**
       *
       */
      stop_ngrok () {
        //
        this.manage_dropdown_close_all()
        storage.delete('ngrok')
        async function stopNgrok () {
          await ngrok.kill()
        }
        stopNgrok()
      },
      /**
       *
       */
      check_started_with_ip (container) {
        return (
          container.state &&
          container.state.network.eth0.addresses.length > 0 &&
          container.status === 'Running' &&
          this.isIP4(container.state.network.eth0.addresses[0].address)
        )
      },
      /**
       *
       */
      start_containers () {
        //
        this.manage_dropdown_close_all()

        this.btn.start_containers = true

        for (var key in this.containers) {
          if (this.containers[key].status !== 'Running') {
            this.start_container(this.containers[key].name)
          }
        }

        this.btn.start_containers = false
      },
      /**
       *
       */
      stop_containers () {
        //
        this.manage_dropdown_close_all()

        this.btn.stop_containers = true

        for (var key in this.containers) {
          if (this.containers[key].status === 'Running') {
            this.stop_container(this.containers[key].name)
          }
        }

        this.btn.stop_containers = false
      },
      /**
       *
       */
      refresh_containers () {
        //
        this.manage_dropdown_close_all()

        this.btn.refresh_containers = true
        this.get_containers(() => {
          this.btn.refresh_containers = false
        })
      },
      /**
       *
       */
      get_containers (callback) {
        this.lxc_list(null, (response) => {
          this.containers = response
          // check http of containers
          for (let key in this.containers) {
            this.$set(this.containers[key], 'services', {http: false, ssh: false, vnc: false})
            if (this.check_started_with_ip(this.containers[key])) {
              this.check_http(this.containers[key].state.network.eth0.addresses[0].address, key)
              this.check_ssh(this.containers[key].state.network.eth0.addresses[0].address, key)
              this.check_vnc(this.containers[key].state.network.eth0.addresses[0].address, key)
            }
          }
          if (typeof callback === 'function') {
            callback()
          }
        })
      },
      /**
       *
       */
      copy_ssh_key (name) {
        //
        this.manage_dropdown_close_all()

        this.lxc_copy_ssh_key(name, process.env['HOME'] + '/.ssh/id_rsa.pub', (response) => {
          this.$notify({
            duration: 2000,
            title: 'Success',
            message: 'SSH key copied to container.',
            type: 'success'
          })
          //
          this.get_containers()
        })
      },
      /**
       *
       */
      start_container (name) {
        //
        this.manage_dropdown_close_all()

        this.lxc_start(name, (response) => {
          this.$notify({
            duration: 2000,
            title: 'Success',
            message: 'Container ' + name + ' started.',
            type: 'success'
          })
          //
          this.get_containers()
        })
      },
      /**
       *
       */
      restart_container (name) {
        //
        this.manage_dropdown_close_all()

        this.lxc_restart(name, (response) => {
          this.$notify({
            duration: 2000,
            title: 'Success',
            message: 'Container ' + name + ' restarted.',
            type: 'success'
          })
          //
          this.get_containers()
        })
      },
      /**
       *
       */
      freeze_container (name) {
        //
        this.manage_dropdown_close_all()

        this.lxc_state(name, 'freeze', (response) => {
          this.$notify({
            duration: 2000,
            title: 'Success',
            message: 'Container ' + name + ' frozen.',
            type: 'success'
          })
          //
          this.get_containers()
        })
      },
      /**
       *
       */
      thaw_container (name) {
        //
        this.manage_dropdown_close_all()

        this.lxc_state(name, 'unfreeze', (response) => {
          this.$notify({
            duration: 2000,
            title: 'Success',
            message: 'Container ' + name + ' unfrozen.',
            type: 'success'
          })
          //
          this.get_containers()
        })
      },
      /**
       *
       */
      stop_container (name) {
        //
        this.manage_dropdown_close_all()

        this.lxc_stop(name, (response) => {
          this.$notify({
            duration: 2000,
            title: 'Success',
            message: 'Container ' + name + ' stopped.',
            type: 'success'
          })
          //
          this.get_containers()
        })
      },
      /**
       *
       */
      snapshot_container (name) {
        //
        this.manage_dropdown_close_all()

        this.lxc_snapshot(name, name + ' (' + moment().format('LLL') + ')', (response) => {
          this.$notify({
            duration: 2000,
            title: 'Success',
            message: 'Snapshot created.',
            type: 'success'
          })
          //
          this.get_containers()
        })
      },
      /**
       *
       */
      image_container (name, index) {
        //
        this.manage_dropdown_close_all()

        let container = this.containers[index]

        let properties = {
          description: name,
          label: (container.config['image.label'] ? container.config['image.label'] : ''),
          architecture: (container.config['image.architecture'] ? container.config['image.architecture'] : ''),
          build: new Date(),
          distribution: (container.config['image.distribution'] ? container.config['image.distribution'] : ''),
          os: (container.config['image.os'] ? container.config['image.os'] : ''),
          release: (container.config['image.release'] ? container.config['image.release'] : ''),
          version: (container.config['image.version'] ? container.config['image.version'] : '')
        }

        this.lxc_image_container(name, properties, (response) => {
          this.$notify({
            duration: 2000,
            title: 'Creating image',
            message: 'This may take a minutes mins.',
            type: 'info'
          })
          // create polling to check operation status
          var poll = true
          var pollOperation = () => {
            if (poll) {
              this.lxc_query('/1.0/operations/' + response.id, 'GET', JSON.stringify({}), (operation) => {
                if (operation.status_code === 103) {
                  setTimeout(pollOperation, 2000)
                } else if (operation.status_code === 200) {
                  this.$notify({
                    duration: 2000,
                    title: 'Success',
                    message: 'Image successfully created.',
                    type: 'success'
                  })
                  poll = false
                  // invalidate local images cache
                  storage.set('images_cached.local', 0)
                } else {
                  this.$notify({
                    duration: 2000,
                    title: 'Error',
                    message: operation.err,
                    type: 'error'
                  })
                  poll = false
                }
              })
            }
          }
          // init poll
          setTimeout(pollOperation, 1000)
          //
          this.get_containers()
        })
      },
      /**
       *
       */
      delete_container (name) {
        //
        this.manage_dropdown_close_all()

        this.$confirm('Are you sure you want to delete ' + name + '?', 'Warning', {
          confirmButtonText: 'OK',
          cancelButtonText: 'Cancel',
          type: 'warning'
        }).then(() => {
          this.lxc_delete(name, (response) => {
            this.$notify({
              duration: 2000,
              title: 'Success',
              message: 'Container ' + name + ' deleted.',
              type: 'success'
            })
            //
            this.get_containers()
          })
        })
      },
      /**
       *
       */
      lock_container (name) {
        this.manage_dropdown_close_all()
        storage.set('locked_containers.' + hash.sha256().update(name).digest('hex'), 1)
      },
      /**
       *
       */
      unlock_container (name) {
        this.manage_dropdown_close_all()
        storage.delete('locked_containers.' + hash.sha256().update(name).digest('hex'))
      },
      /**
       *
       */
      locked_container (name) {
        return storage.has('locked_containers.' + hash.sha256().update(name).digest('hex'))
      },
      /**
       *
       */
      open (link) {
        this.manage_dropdown_close_all()

        this.$electron.shell.openExternal(link)
      },
      /**
       *
       */
      openCmd (link) {
        var childProcess = require('child_process')
        childProcess.exec(link)
      }
    }
  }
</script>

<style>

</style>
