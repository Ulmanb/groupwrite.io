<template>
  <div id="app" >
    <div class="debug" v-if="isAdmin">
      <h1>debug toolbar</h1>
    <p>
      <router-link to="/">Home</router-link>
      <router-link to="/queue">Queue</router-link>
      <router-link to="/game">Game</router-link>
      <router-link to="/admin">Admin</router-link>
    </p>
   </div>
    <router-view></router-view>
  </div>
</template>

<script>
  import store from './components/store'
  import assert from 'assert'

  var request = require('superagent')
  var socketlib = require('socket.io-client')
  var socket = socketlib()
  var VueRouter = require('vue-router')
  var router = new VueRouter()

  function getAdminKey() {
    return window.localStorage.getItem('writeio-admin-key')
  }

  function isAdmin() {
    return !!getAdminKey()
  }

  export default {
    name: 'app',
    components: {
      // Add any custom components here
    },
    data: function () {
      return {
        sharedState: store.state,
        consts: store.consts,
        isAdmin: isAdmin()
      }
    },
    created: function () {
      var self = this

      function updateState() {
        // We only ask for state if we're logged in or admin
        if (self.sharedState.playerId) {
          request.get('/api/state')
            .set('Accept', 'application/json')
            .query({ playerId: self.sharedState.playerId })
            .end((err, res) => {
              assert(!err)
              assert(res.status === 200)

              let state = res.body
              assert(state)

              console.log(`Got server state (updated num players from ${self.sharedState.players.length} to ${state.players.length})`)
              self.sharedState.players = state.players

              assert(state.players.length <= self.consts.maxPlayers)
              // TODO Read current location from VueRouter here. router.history.current shows '/' for some reason
              if (state.players.length === self.consts.maxPlayers && window.location.hash.endsWith('/queue')) {
                router.replace('/game')
              }
            })
        } else {
          if (isAdmin()) {
            let adminKey = getAdminKey()
            console.log(`Admin getting state with key ${adminKey}`)
            request.get('/api/adminState')
              .set('Accept', 'application/json')
              .query({ adminKey })
              .end((err, res) => {
                assert(!err)
                assert(res.status === 200)

                self.sharedState.adminState = res.body
              })
          }
        }
      }

      updateState()

      // Update on change when server state changes
      socket.on('server:state', function () {
        updateState()
      })

      // Routes
      // If we're not on the admin page, reload --> home
      if (!window.location.href.endsWith('admin')) {
        router.replace('/')
      }
    }
  }

</script>

<style>
  #app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
  }

  .debug {
    background-color: lightgray
  }

  a {
    margin-right: 10px;
  }

  .center {
    top: 50%;
    left: 50%;
    transform: translate3d(-50%, -50%, 0);
    position: absolute;
  }
</style>
