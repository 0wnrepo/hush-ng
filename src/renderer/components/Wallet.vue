<template>
  <div id="wrapper" style="-webkit-app-region: drag">
    <side-menu></side-menu>
    <wallet-menu></wallet-menu>
    <div class="inner-content">
      <addresses></addresses>
      <div class="bottom-row">
        <div class="box">
          <ul id="texts">
            <li>T:</li>
            <li>Z:</li>
            <li>TOTAL:</li>
          </ul>
          <ul id="balances">
            <li>{{ tBalance }} HUSH</li>
            <li>{{ zBalance }} HUSH</li>
            <li>{{ fullBalance }} HUSH</li>
          </ul>
        </div>
        <div class="box alt">
          <p>For more on Z and T addresses, visit the following links:</p>
          <div class="links">
            <a @click="open('https://discord.gg/VfaZjyR')">MyHush.org</a>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  import WalletMenu from './Wallet/WalletMenu'
  import SideMenu from './shared/Menu'
  import Addresses from './Wallet/Addresses'

  const Repeat = require('repeat')
  var request = require('request')
  var store = require('store')
  var cmd = require('node-cmd')
  const bitcoin = require('bitcoin')

  export default {
    name: 'wallet',
    components: { WalletMenu, SideMenu, Addresses },
    data () {
      return {
        fullBalance: 0,
        tBalance: 0,
        zBalance: 0,
        tAddresses: [],
        zAddresses: [],
        blockHeight: 0,
      }
    },
    methods: {
      open (link) {
        this.$electron.shell.openExternal(link)
      },
      startPolling (interval) {
        var self = this
        var execPath = store.get('execPath')

        // Start Hushd
        var exec = ''
        if (require('os').platform() == 'linux') {
          exec = 'cd ' + execPath + '&& ./hushd'
        } else if (require('os').platform() == 'win32') {
          exec = execPath + '\\hushd.exe'
          console.log(exec)
        }
        cmd.get(
          exec,
          function(err, data, stderr){
              if (!err) {
                 console.log('HushNG: Could not start hushd!')
              } else {
                 console.log('HushNG: Started hushd')
              }

          }
        )

        var client = new bitcoin.Client({
          port: 8822,
          user: store.get('connection').rpcuser,
          pass: store.get('connection').rpcpassword,
          timeout: 60000
        });

        // Get T-Addresses
        client.getAddressesByAccount('', function(err, data, resHeaders) {
          if (err) return console.log(err);
          var tAddr = []
          for (var i = 0; i < data.length; i++) {
            tAddr.push({"address": data[i], "balance": 0})
          }
          console.log('Scanning for T-Addrs... Found: ' + tAddr.length + '\n' + JSON.stringify(tAddr))
          store.set('tAddresses', tAddr)
        });

        // Get Z-Addresses
        client.cmd('z_listaddresses', function(err, data, resHeaders){
          if (err) return console.log(err);
          var zAddr = []
          for (var i = 0; i < data.length; i++) {
            zAddr.push({"address": data[i], "balance": 0})
          }
          console.log('Scanning for Z-Addrs... Found: ' + zAddr.length + '\n' + JSON.stringify(zAddr))
          store.set('zAddresses', zAddr)
        });

        Repeat(function() {
          var hushData = {}
          // Get Baseline Info
          client.getInfo(function(err, data, resHeaders) {
            if (err) {
              var getInfo = {}
              getInfo.connections = 0
              getInfo.blocks = 'Scanning...'
              store.set('getInfo', getInfo)
              console.log(err);
            } else {
              store.set('getInfo', data)
              console.log('Blockchain data:',  JSON.stringify(data));
            }
          });

          // Get Wallet Info
          client.getWalletInfo(function(err, data, resHeaders) {
            if (err) return console.log(err);
            store.set('getWalletInfo', data)
            console.log('Wallet data:',  JSON.stringify(data));
          });

          //Get Z Balance
          client.cmd('z_gettotalbalance', function(err, balance, resHeaders){
            if (err) return console.log(err);
            store.set('z_balance', balance)
            self.fullBalance = store.get('z_balance').total
            self.tBalance = store.get('z_balance').transparent
            self.zBalance = store.get('z_balance').private
          });

          client.getWalletInfo(function(err, data, resHeaders) {
            if (err) return console.log(err);
            store.set('getWalletInfo', data)
            self.fullBalance = store.get('getInfo').balance
          });

          // Refresh T-Balances
          var taddr = store.get('tAddresses')
          for (let i = 0; i < taddr.length; i++) {
            client.getReceivedByAddress(taddr[i].address, function(err, data, resHeaders) {
              taddr[i].balance = data
              store.set('tAddresses', taddr)
            });
          }

          // Refresh Z-Balances
          var zaddr = store.get('zAddresses')
          for (let i = 0; i < zaddr.length; i++) {
            client.cmd('z_getbalance', zaddr[i].address, function(err, data, resHeaders) {
              zaddr[i].balance = data
              store.set('zAddresses', zaddr)
            });
          }
        }).every(interval, 'ms').start.now();
      }
    },
    mounted: function() {
      this.startPolling(2500)
    }
  }
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Poppins:300,400,500,700');

  * {
    font-family: 'Poppins', sans-serif;
    color: #2d2d2d;
  }

  #wrapper {
    background:
      radial-gradient(
        ellipse at top left,
        rgba(255, 255, 255, 1) 40%,
        rgba(229, 229, 229, .9) 100%
      );
    height: 100vh;
    width: 100vw;
    padding: 0px;
  }

  #logo {
    float: left;
    height: 100px;
    margin-bottom: 40px;
    width: auto;
  }

  #logo-text {
    float: left;
    margin-left: 20px;
    line-height: 100px;
    font-size: 18pt;
    font-weight: bold;
  }

  main {
    clear: left;
    display: flex;
    justify-content: space-between;
  }

  main > div { flex-basis: 50%; }

  .left-side {
    display: flex;
    flex-direction: column;
  }

  .welcome {
    color: #555;
    font-size: 23px;
    margin-bottom: 10px;
  }

  .title {
    color: #2c3e50;
    font-size: 20px;
    font-weight: bold;
    margin-bottom: 6px;
  }

  .title.alt {
    font-size: 18px;
    margin-bottom: 10px;
  }

  .install-list {
    color: #555;
    font-weight: initial;
    list-style-type: none;
    margin-bottom: 50px;
  }

  .intall-list li {
    color: #000;
    height: 20px;
  }

  .install-list li code {
    color: #3e3e3e;
    font-weight: bold;
  }

  .progress {
    float: left;
    width: 10px;
    height: 10px;
    margin-top: 6px;
    margin-right: 5px;
    border-radius: 50%;
    background-color: #d1d1d1;
  }

  .pending {
    border: 2px solid #7ed35f;
    background-color: transparent;
  }

  .success {
    background-color: #7ed35f;
  }

  .error {
    background-color: #ef4049;
  }

  .doc .button {
    font-size: .9em;
    cursor: pointer;
    outline: none;
    padding: 0.75em 2em;
    border-radius: 2em;
    display: inline-block;
    color: #fff;
    background-color: #2F77F7;
    transition: all 0.15s ease;
    box-sizing: border-box;
    border: 1px solid #2F77F7;
    margin-top: 10px;
    text-decoration: none;
    -webkit-app-region: no-drag;
  }

  .doc .button:hover {
    background-color: #2262d6;
  }

  .doc .button-alt {
    color: #3e3e3e;
    margin-right: 5px;
    background-color: transparent;
  }

  .doc .button-alt:hover {
    background-color: #e2e2e2;
  }

  .inner-content {
    position: absolute;
    width: 100%;
    height: 100%;
    padding: 77px 20px 20px 88.3px;
  }

  .bottom-row {
    clear: both;
    position: fixed;
    bottom: 15px;
  }

  .bottom-row .box {
    float: left;
    width: 300px;
    height: 95px;
    margin-right: 15px;
    padding: 10px 15px 10px 15px;
    background-color: #3e3e3e;
  }

  .bottom-row .alt {
    font-weight: 300;
    font-size: 11pt;
    background-color: #cacaca;
  }

  .bottom-row .box .links {
    position: absolute;
    bottom: 10px;
  }

  .bottom-row .box .links a {
    text-decoration: none;
    cursor: pointer;
    -webkit-app-region: no-drag;
  }

  .bottom-row .box #texts {
    float: left;
    list-style-type: none;
  }

  .bottom-row .box #balances {
    float: right;
    text-align: right;
    list-style-type: none;
  }

  .bottom-row .box li {
    font-weight: 500;
    color: #fff;
  }
</style>
