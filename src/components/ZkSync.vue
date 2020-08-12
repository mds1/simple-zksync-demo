<template>
  <div class="hello">
    <h1>
      zkSync Demo
    </h1>
    <button v-if="syncWallet === undefined" @click="signIn">Sign in to zkSync</button>
    <div v-else-if="isLoading">Loading...</div>
    <div v-else>
      <div style="margin-bottom: 1rem;">Logged in!</div>
      <div
        v-if="this.balances.committedEth && this.balances.verifiedEth"
        style="margin-bottom: 1rem;"
      >
        <div>Committed balance: {{ formatEther(this.balances.committedEth) }}</div>
        <div>Verified balance: {{ formatEther(this.balances.verifiedEth) }}</div>
      </div>
      <div>Available actions are below</div>
      <button @click="depositEth" label="sfsdf">Deposit 0.01 ETH</button>
    </div>
  </div>
</template>

<script>
import { ethers } from 'ethers';

export default {
  name: 'ZkSync',

  data() {
    return {
      isLoading: false,
      zksync: undefined,
      syncProvider: undefined,
      ethersProvider: undefined,
      signer: undefined,
      syncWallet: undefined,
      balances: {
        committedEth: undefined,
        verifiedEth: undefined,
      },
    };
  },

  async mounted() {
    console.log('------------------------ Start app setup ------------------------');
    // Get user's wallet
    this.ethersProvider = new ethers.providers.Web3Provider(window.ethereum);
    this.signer = this.ethersProvider.getSigner();

    // Get zkSync provider
    this.zksync = await import('zksync');
    console.log('zksync: ', this.zksync);
    this.syncProvider = await this.zksync.getDefaultProvider('rinkeby');
    console.log('syncProvider: ', this.syncProvider);
    console.log('------------------------ End app setup ------------------------');
  },

  methods: {
    formatEther: ethers.utils.formatEther,

    /**
     * @notice Signs into zkSync by asking the user to sign a message
     */
    async signIn() {
      this.isLoading = true;
      console.log('------------------------ Start login/register ------------------------');
      this.syncWallet = await this.zksync.Wallet.fromEthSigner(this.signer, this.syncProvider);
      console.log("syncWallet (user's zksync wallet): ", this.syncWallet);

      // To control assets in zkSync network, an account must register a separate public key once
      if (!(await this.syncWallet.isSigningKeySet())) {
        console.log(
          'User has not registered signing key (this is a one-time step). Setting it now...'
        );
        if ((await this.syncWallet.getAccountId()) == undefined) {
          throw new Error('Unknwon account');
        }

        const changePubkey = await this.syncWallet.setSigningKey();
        console.log('Waiting for transaction to be confirmed...');

        // Wait until the tx is committed
        await changePubkey.awaitReceipt();
        console.log('Signing key successfully set');
      }
      console.log('------------------------ End login/register ------------------------');

      console.log('------------------------ Start check zkSync balances ------------------------');
      // Committed state is not final yet
      this.balances.committedEth = await this.syncWallet.getBalance('ETH');
      // Verified state is final
      this.balances.verifiedEth = await this.syncWallet.getBalance('ETH', 'verified');

      console.log("Got ETH balances, now let's get account state");
      const accountState = await this.syncWallet.getAccountState();
      console.log('accountState: ', accountState);
      console.log('------------------------ End check zkSync balances ------------------------');
      this.isLoading = false;
    },

    /**
     * @notice Deposit ETH from Ethereum into zkSync
     */
    async depositEth() {
      console.log('------------------------ Start deposit ------------------------');
      // Deposit ETH
      const userZksyncAddress = this.syncWallet.address();
      const deposit = await this.syncWallet.depositToSyncFromEthereum({
        depositTo: userZksyncAddress,
        token: 'ETH',
        amount: ethers.utils.parseEther('0.01'),
      });
      console.log('deposit: ', deposit);

      // Await confirmation from the zkSync operator
      // Completes when a promise is issued to process the tx
      const depositReceipt = await deposit.awaitReceipt();
      console.log(new Date(), 'depositReceipt: ', depositReceipt);
      console.log('This means we have received a promise to process the transaction');

      // Await verification
      // Completes when the tx reaches finality on Ethereum
      const depositReceiptVerify = await deposit.awaitVerifyReceipt();
      console.log(new Date(), 'depositReceiptVerify: ', depositReceiptVerify);
      console.log('This means the transaction has been finalized on Ethereum');
      console.log('------------------------ End deposit ------------------------');
    },
  },
};
</script>
