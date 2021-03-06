<script>
  import Api from '../../api/api'
  import Headerblock from '../partials/Header.vue'
  import Sidebar from './partials/Sidebar.vue'

  import keythereum from 'keythereum'
  import sjcl from 'sjcl'
  import Tx from 'ethereumjs-tx';

  import VueQRCodeComponent from 'vue-qrcode-component'

  export default {
    data: function () {
      return {
        allow_is_loading: false,
        withdraw_is_loading: false,
        popupVisible: false,
        account: [],
        balance_eth: 0,
        balance_cl: 0,
        allowance: 0,
        current_role: null,
        accepted_currency: ['CL', 'ETH'],
        min_withdraw: 0.001,
        min_allow: 1,
      }
    },
    methods: {
      showPopup: function (modal, e) {
        let vm = this;
        e.preventDefault();
        vm.$modal.show(modal);
      },

      withdraw: function (e) {
        let vm = this;
        e.preventDefault();

        vm.withdraw_is_loading = true;
        let amount = parseFloat(e.target.amount.value);
        let password = e.target.password.value;
        let currency = e.target.currency.value;
        let address = e.target.address.value;

        if (!amount || amount < vm.min_withdraw) {
          vm.withdraw_is_loading = false;
          return vm.$helpers.errorMsg('Bad amount');
        }

        if (!vm.$config.web3.isAddress(address)) {
          vm.withdraw_is_loading = false;
          return vm.$helpers.errorMsg('Bad address');
        }

        if (vm.accepted_currency.indexOf(currency) == -1) {
          vm.withdraw_is_loading = false;
          return vm.$helpers.errorMsg('Select currency');
        }

        let crypto_pair = atob(vm.account.acc_crypt_pair);
        let decrypted_data = null;
        try {
          decrypted_data = JSON.parse(sjcl.json.decrypt(password, crypto_pair));
        } catch (err) {
          console.error(err);
          vm.withdraw_is_loading = false;
          return vm.$helpers.errorMsg('Invalid password');
        }

        var count = vm.$config.web3.eth.getTransactionCount(vm.account.acc_crypt_address);
        var gasPrice = 21000000000;
        var gasLimit = 100000;

        var tx = null;

        if (currency == 'CL') {
          tx = new Tx({
            "from": vm.account.acc_crypt_address,
            "nonce": vm.$config.web3.toHex(count),
            "gasPrice": vm.$config.web3.toHex(gasPrice),
            "gasLimit": vm.$config.web3.toHex(gasLimit),
            "to": vm.$config.cl_contract_address,
            "value": 0,
            "data": vm.$config.contract_cl.transfer.getData(address, vm.$config.web3.toWei(amount)),
            "chainId": vm.$config.chainId
          });
        } else if (currency == 'ETH') {
          console.log(amount);
          console.log(vm.$config.web3.toWei(amount, 'ether'));
          tx = new Tx({
            "nonce": vm.$config.web3.toHex(count),
            "gasPrice": vm.$config.web3.toHex(gasPrice),
            "gasLimit": vm.$config.web3.toHex(gasLimit),
            "to": address,
            "value": vm.$config.web3.toHex(vm.$config.web3.toWei(amount, 'ether')),
            "chainId": vm.$config.chainId
          });
        }

        tx.sign(new Buffer(decrypted_data.privateKey, 'hex'));

        var serializedTx = tx.serialize();

        vm.$config.web3.eth.sendRawTransaction('0x' + serializedTx.toString('hex'), function(err, hash) {
          if (err) {
            console.log(err);
            vm.withdraw_is_loading = false;
            vm.$modal.hide('withdraw');
            vm.$helpers.errorMsg('Error while trying to withdraw')
          } else {
            console.log(hash);
            vm.withdraw_is_loading = false;
            vm.$modal.hide('withdraw');
            vm.$helpers.successMsg('Withdraw success')
          }
        });


      },

      allowForEscrow: function (e) {
        let vm = this;
        e.preventDefault();

        vm.allow_is_loading = true;
        let amount = e.target.amount.value;
        let password = e.target.password.value;

        if (!amount || amount < vm.min_allow) {
          vm.allow_is_loading = false;
          return vm.$helpers.errorMsg('Bad amount');
        }

        let crypto_pair = atob(vm.account.acc_crypt_pair);
        let decrypted_data = null;
        try {
          decrypted_data = JSON.parse(sjcl.json.decrypt(password, crypto_pair));
        } catch (err) {
          console.error(err);
          vm.allow_is_loading = false;
          return vm.$helpers.errorMsg('Invalid password');
        }

        var count = vm.$config.web3.eth.getTransactionCount(vm.account.acc_crypt_address);
        var gasPrice = 21000000000;
        var gasLimit = 100000;

        var tx = new Tx({
          "from": vm.account.acc_crypt_address,
          "nonce": vm.$config.web3.toHex(count),
          "gasPrice": vm.$config.web3.toHex(gasPrice),
          "gasLimit": vm.$config.web3.toHex(gasLimit),
          "to": vm.$config.cl_contract_address,
          "data": vm.$config.contract_cl.approve.getData(vm.$config.escrow_contract_address, vm.$config.web3.toWei(amount), {from: vm.account.acc_crypt_address}),
          "chainId": vm.$config.chainId
        });

        tx.sign(new Buffer(decrypted_data.privateKey, 'hex'));

        var serializedTx = tx.serialize();

        vm.$config.web3.eth.sendRawTransaction('0x' + serializedTx.toString('hex'), function(err, hash) {
          if (err) {
            console.log(err);
            vm.allow_is_loading = false;
            vm.$modal.hide('allowance');
            vm.$helpers.errorMsg('Error while trying to add allowance')
          } else {
            console.log(hash);
            vm.allow_is_loading = false;
            vm.$modal.hide('allowance');
            vm.$helpers.successMsg('Allowance will be added after mining')
          }
        });

      }
    },
    created() {
      let vm = this;
      vm.account = vm.$store.getters.accountData;
      vm.current_role = vm.$helpers.getCurrentRole(vm.account);
      vm.balance_eth = vm.$config.web3.fromWei(vm.$config.web3.eth.getBalance(vm.account.acc_crypt_address).toNumber());
      vm.balance_cl = vm.$config.web3.fromWei(vm.$config.contract_cl.balanceOf(vm.account.acc_crypt_address).toNumber());
      vm.allowance = vm.$config.web3.fromWei(vm.$config.contract_cl.allowance(vm.account.acc_crypt_address, vm.$config.escrow_contract_address).toNumber());
    },
    mounted () {
      this.$helpers.externalPluginsExecute();
    },
    components: {
      Headerblock,
      Sidebar,
      'qrcode': VueQRCodeComponent
    }
  }
</script>

<template>
  <div>
    <headerblock fullwidth="true"></headerblock>
    <modal width="296" name="deposit" :adaptive="true" :scrollable="true" height="auto">
      <div class="row" style="padding:20px;">
        <div class="col-lg-12 text-center">
          <qrcode :text="account.acc_crypt_address"></qrcode>
        </div>
      </div>
    </modal>
    <modal name="allowance" :adaptive="true" :scrollable="true" height="auto">
      <form style="padding: 50px;" class="tab-pane fade active in" @submit="allowForEscrow">
        <div class="form-group">
          <input type="number" :min="min_allow" class="form-control" name="amount" value="" placeholder="Amount of allowance">
          <i class="form-group__bar"></i>
        </div>

        <div class="form-group">
          <input type="password" class="form-control" name="password" value="" placeholder="Password">
          <i class="form-group__bar"></i>
        </div>

        <button-spinner
            :isLoading="allow_is_loading"
            :disabled="allow_is_loading"
            class="btn btn-primary btn-block m-t-10 m-b-10"
        >
          <span>Allow</span>
        </button-spinner>
      </form>
    </modal>
    <modal name="withdraw" :adaptive="true" :scrollable="true" height="auto">
      <form style="padding: 50px;" class="tab-pane fade active in" @submit="withdraw">

        <div class="form-group">
          <input type="number" :min="min_withdraw" :step="min_withdraw" class="form-control" name="amount" value="" placeholder="Amount">
          <i class="form-group__bar"></i>
        </div>

        <div class="form-group">
          <input type="text" class="form-control" name="address" value="" placeholder="Address">
          <i class="form-group__bar"></i>
        </div>

        <div class="form-group">
          <select class="form-control" name="currency">
            <option value="0">Select currency</option>
            <option v-for="cur in accepted_currency" :value="cur">{{cur}}</option>
          </select>
          <i class="form-group__bar"></i>
        </div>

        <div class="form-group">
          <input type="password" class="form-control" name="password" value="" placeholder="Password">
          <i class="form-group__bar"></i>
        </div>

        <button-spinner
            :isLoading="withdraw_is_loading"
            :disabled="withdraw_is_loading"
            class="btn btn-primary btn-block m-t-10 m-b-10"
        >
          <span>Withdraw</span>
        </button-spinner>
      </form>
    </modal>
    <main id="main">
      <sidebar :role="current_role"></sidebar>
      <section id="main__content">
        <div class="main__container">
          <header class="main__title">
            <h2>Balance</h2>
          </header>
          <div class="list-group list-group--block">
            <div class="list-group-item media">
              <div class="row">
                <div class="col-md-6">
                  <div class="balance-block">
                    <h2>Address</h2>
                    <h4>Your address is <span>{{account.acc_crypt_address}}</span></h4>
                  </div>
                </div>
                <div class="col-md-6">
                  <div class="balance-button">
                    <button @click="showPopup('deposit', $event)">Deposit</button>
                  </div>
                </div>
              </div>
              <div class="row">
                <div class="col-md-6">
                  <div class="balance-block">
                    <h2>Balances</h2>
                    <h4>CL: <span>{{balance_cl}}</span></h4>
                    <h4>ETH: <span>{{balance_eth}}</span></h4>
                  </div>
                </div>
                <div class="col-md-6">
                  <div class="balance-button">
                    <button @click="showPopup('withdraw', $event)">Withdraw</button>
                  </div>
                </div>
              </div>
              <div class="row">
                <div class="col-md-6">
                  <div class="balance-block">
                    <h2>Allowance for Coinlancer Escrow</h2>
                    <h4>CL: <span>{{allowance}}</span></h4>
                  </div>
                </div>
                <div class="col-md-6">
                  <div class="balance-button">
                    <button @click="showPopup('allowance', $event)">Allow</button>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>

    </main>
  </div>
</template>

<style scoped>
  .list-group--block {
    box-shadow: none;
    margin-bottom: 30px;
  }

  .list-group--block .list-group-item {
    border: 2px solid #e0e0e0;
    padding: 35px 60px;
    overflow-x: auto;
  }

  .balance-block h2 {
    color: #494949;
  }

  .balance-block h4 {
    color: #494949;
    margin-top: 20px;
  }

  .list-group--block .list-group-item button {
    padding: 12px 35px;
    font-size: 18px;
    border: 0;
    margin-top: 26px;
    color: #fff;
    background-color: #01bf86;
  }

  .list-group--block .list-group-item button:hover, .list-group--block .list-group-item button:active, .list-group--block .list-group-item button:visited, .list-group--block .list-group-item button:focus {
    color: #fff;
    background-color: #039368;
  }

  .list-group--block .list-group-item .balance-btn {
    float: right;
  }


  .balance-button {
    float: right;
  }

  .balance-button button {
    margin-left: 10px;
  }


  .balance-details {
    position: relative;
    margin-bottom: 25px;
  }

  .balance-details:after {
    content: '';
    position: absolute;
    width: 100%;
    height: 1px;
    background-color: #e0e0e0;
    bottom: 0;
    left: 0;
  }

  .balance-details h2 {
    margin-bottom: 35px;
  }

  .balance-details h4 {
    margin-bottom: 15px;
  }

  .balance-details p {
    font-size: 14px;
    padding-bottom: 25px;
  }

  .payment-items:last-child .balance-details:after {
    display: none;
  }

  #modal-form .modal-header {
    padding: 20px;
  }
  #modal-form .img-wrap {
    text-align: center;
  }

  .modal-close {
    background-color: transparent;
    border: none;
    float: right;
  }

  .modal-close span {
    font-size: 18px;
  }

  .overflowed {
    overflow-y: auto;
  }
</style>