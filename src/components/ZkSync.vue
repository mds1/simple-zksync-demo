<template>
  <div>
    <!-- ALERTS -->
    <!-- Success alert -->
    <div
      v-if="showAlert"
      class="alert alert-dismissible fade show container"
      :class="alertClass"
      role="alert"
      id="alert"
    >
      {{ alertMessage }}
      <button type="button" class="close" aria-label="Close" @click="closeAlert">
        <span aria-hidden="true">&times;</span>
      </button>
    </div>

    <!-- CONTENT -->
    <h1>
      zkSync Demo
    </h1>
    <p class="figure-caption">
      You are connected to <span class="font-weight-bold">{{ network }}</span> with
      <span class="font-weight-bold">{{ userAddress }}</span>
    </p>
    <h3 class="mt-4">Instructions</h3>
    <p style="font-weight: bold; color: red;">
      This will work on mainnet and Rinkeby, so make sure to select the desired network
    </p>
    <p class="container text-left">
      The Cart section below mimics a simple Gitcoin Grants cart with two items. All amounts are
      currently hardcoded for simplicity. All you need to do is enter two different addresses in the
      right-most column and click the Checkout button. This will start the zkSync checkout flow.
      <br /><br />
      It is recommended to enter other addresses from your MetaMask wallet as the recipient
      addresses. This will let you test the withdrawal flow as well.
    </p>

    <h3 class="mt-5">Cart</h3>

    <!-- CART -->
    <!-- Item 2 -->
    <div class="row justify-content-center align-items-center my-3">
      <div class="col-2 text-left font-weight-bold">
        Grant Name
      </div>
      <div class="col-2 text-left font-weight-bold">
        Amount
      </div>
      <div class="col-2 text-left font-weight-bold">
        Grant Recipient Address
      </div>
    </div>
    <!-- Item 1 -->
    <div class="row justify-content-center align-items-center my-3">
      <div class="col-2 text-left">
        Test Grant 1
      </div>
      <div class="col-2 text-left">{{ donations[0].amount }} {{ donations[0].tokenName }}</div>
      <div class="col-2 text-left">
        <input
          v-model="donations[0].recipientAddress"
          type="text"
          class="form-control"
          placeholder="Enter recipient address"
        />
      </div>
    </div>
    <!-- Item 2 -->
    <div class="row justify-content-center align-items-center my-3">
      <div class="col-2 text-left">
        Test Grant 2
      </div>
      <div class="col-2 text-left">{{ donations[1].amount }} {{ donations[1].tokenName }}</div>
      <div class="col-2 text-left">
        <input
          v-model="donations[1].recipientAddress"
          type="text"
          class="form-control"
          placeholder="Enter recipient address"
        />
      </div>
    </div>

    <button class="btn btn-primary mt-3" @click="checkout">Checkout</button>
  </div>
</template>

<script>
import { ethers } from 'ethers';
const { utils } = ethers;

export default {
  name: 'ZkSync',

  data() {
    return {
      // donor's web3 wallet
      ethersProvider: undefined,
      signer: undefined,
      userAddress: undefined,
      network: undefined,

      // zkSync
      zksync: undefined, // zksync package
      syncProvider: undefined, // donor's zksync provider instance

      // donation details
      donations: [
        {
          tokenName: 'ETH',
          amount: '0.002',
          recipientAddress: undefined,
        },
        {
          tokenName: 'ETH',
          amount: '0.003',
          recipientAddress: undefined,
        },
      ],

      showAlert: false,
      alertClass: undefined,
      alertMessage: undefined,
    };
  },

  async mounted() {
    // Connect wallet
    await window.ethereum.enable();

    // Get details
    this.ethersProvider = new ethers.providers.Web3Provider(window.ethereum);
    this.signer = this.ethersProvider.getSigner();
    this.userAddress = await this.signer.getAddress();
    const network = await this.ethersProvider.getNetwork();
    // ethers identifies mainnet as 'homestead' so we rename for clarity
    this.network = network.chainId === 1 ? 'mainnet' : network.name;

    // Get zkSync provider
    this.zksync = await import('zksync');
    this.syncProvider = await this.zksync.getDefaultProvider('rinkeby');

    // Generat ephemeral wallet and save to local storage, unless one already exists
    if (!this.getEphemeralWallet()) {
      // None exits, create one
      const ephemeralWallet = new ethers.Wallet.createRandom();
      window.localStorage.setItem('ephemeral-mnemonic', ephemeralWallet.mnemonic.phrase);
      console.log('generated ephemeralWallet: ', ephemeralWallet);
    } else {
      console.log('ephemeral wallet found: ', this.getEphemeralWallet());
    }
  },

  methods: {
    handleError(e) {
      console.error(e);
      this.alertMessage = e.message;
      this.alertClass = 'alert-danger';
      this.showAlert = true;
    },

    closeAlert() {
      this.showAlert = false;
    },

    async checkout() {
      try {
        // Hide alert if visibile
        this.showAlert = false;

        // Make sure recipient addresses are valid
        const donations = this.donations;
        donations[0].recipientAddress = await utils.getAddress(donations[0].recipientAddress);
        donations[1].recipientAddress = await utils.getAddress(donations[1].recipientAddress);

        this.alertMessage = 'Your donation was successfully completed!';
        this.alertClass = 'alert-success';
        this.showAlert = true;
      } catch (e) {
        this.handleError(e);
      }
    },
  },
};
</script>
