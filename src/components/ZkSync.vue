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
      You are connected to
      <span class="font-weight-bold" :class="{ red: network === 'mainnet' }">{{ network }}</span>
      with
      <span class="font-weight-bold">{{ userAddress }}</span>
    </p>
    <h3 class="mt-4">Instructions</h3>
    <div class="container text-left mb-5">
      <p class="text-center" style="font-weight: bold; color: red;">
        This will work on mainnet and Rinkeby, so make sure to select the desired network
      </p>
      <p>
        The Cart section below mimics a simple Gitcoin Grants cart with two items. The Recipient
        Dashboard section below that represents the experience for grant owners to withdraw their
        funds back to L1. All amounts are currently hardcoded for simplicity. All you need to do is
        enter two different addresses in the right-most column and click the Checkout button. This
        will start the zkSync checkout flow.
      </p>

      <p>
        Suggestions for using this app:
      </p>
      <ul>
        <li>
          Enter addresses from your MetaMask wallet as the recipient addresses. This will let you
          test the withdrawal flow as well (once implemented)
        </li>
        <li>Please refresh the page if you change the MetaMask address</li>
        <li>
          Open the console to watch status updates, otherwise it will look like nothing is
          happening. This was done to reduce development time for this POC. In reality, events will
          be reflected in the Gitcoin UI
        </li>
        <li>
          Use <a href="https://github.com/mds1/simple-zksync-demo" target="_blank">zkScan</a> to
          view transactions in the block explorer. You can get transactions hashes by expanding
          variables logged to the console
        </li>
        <li>
          View the source code
          <a href="https://github.com/mds1/simple-zksync-demo" target="_blank">here</a>
        </li>
      </ul>
    </div>

    <div class="container" style="border-top: 1px solid rgba(0, 0, 0, 0.2); height: 1px;" />

    <!-- CART -->
    <h3 class="mt-5">Cart</h3>
    <p class="mb-5">
      This section is intended to mimic the actual Gitcoin Grants cart checkout flow
    </p>

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

    <button class="btn btn-primary mt-3 mb-5" :disabled="isCheckoutInProgress" @click="checkout">
      Checkout
    </button>

    <!-- ACCOUNT SUMMARY -->
    <div class="container" style="border-top: 1px solid rgba(0, 0, 0, 0.2); height: 1px;" />
    <div class="container mt-5">
      <h3>Recipient Dashboard</h3>
      <p class="mb-4">This section contains the aspects important to grant owners</p>

      <div class="row justify-content-center align-items-center my-3">
        <!-- User's current wallet -->
        <div class="col-6 mx-2 my-2 data-card">
          <h5>Selected Web3 Account</h5>
          <p class="address">{{ userAddress }}</p>
          <button v-if="!syncAccountState" class="btn btn-primary mt-3" @click="loginToZkSync">
            Log in to zkSync
          </button>
          <div v-else>
            <div class="row justify-content-center text-left">
              <div class="col-4">
                Committed balance:
              </div>
              <div class="col-auto">
                {{ formatEther(syncAccountState.committed.balances.ETH) }} ETH
              </div>
            </div>
            <div class="row justify-content-center text-left">
              <div class="col-4">
                Verified balance:
              </div>
              <div class="col-auto">
                {{ formatEther(syncAccountState.verified.balances.ETH) }} ETH
              </div>
            </div>
            <button class="btn btn-primary mt-3" @click="withdrawVerifiedBalance">
              Withdraw Full Verified Balance
            </button>
          </div>
        </div>
      </div>

      <div v-if="syncAccountState" class="figure-caption text-left mb-5">
        Some notes and definitions:
        <ul>
          <li>
            Committed balance is the last state known to the zkSync network. This can be ahead of
            the verified state
          </li>
          <li>Verified balance is proved with a zero-knowledge proof on the Ethereum network</li>
          <li>
            Committed and verified balances are not additive. In other words, if committed balance
            is 1 ETH and verified balance is 0.75 ETH, then you have 0.25 ETH that is committed but
            not yet verified
          </li>
        </ul>
      </div>
    </div>
    <!-- END ACCOUNT SUMMARY -->
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
      syncWallet: undefined, // donor's zksync wallet
      syncAccountState: undefined, // token balances and nonce

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

      // UI helpers
      isCheckoutInProgress: false,
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
    this.syncProvider = await this.zksync.getDefaultProvider(this.network);
  },

  methods: {
    formatEther: utils.formatEther,

    handleError(e) {
      console.error(e);
      this.alertMessage = e.message;
      this.alertClass = 'alert-danger';
      this.showAlert = true;
      this.isCheckoutInProgress = false;
    },

    closeAlert() {
      this.showAlert = false;
    },

    async loginToZkSync() {
      // Login
      // OPTION 1
      console.log('Waiting for user to sign the prompt to log in...');
      this.syncWallet = await this.zksync.Wallet.fromEthSigner(this.signer, this.syncProvider);
      console.log(
        '✅ Login complete. Sync wallet generated from web3 account. View wallet:',
        this.syncWallet
      );

      // // OPTION 2
      // const ephemeralSigner = await this.zksync.Wallet.fromEthSigner(
      //   this.getEphemeralWallet(),
      //   this.syncProvider
      // );
      //
      // this.syncWallet = await this.zksync.Wallet.fromEthSigner(
      //   this.signer,
      //   this.syncProvider,
      //   ephemeralSigner,
      //   undefined,
      //   'EthereumSignature'
      // );

      // Get latest zkSync state
      await this.getZkSyncAcountState();
      console.log(
        '✅ Latest sync state fetched for sync wallet. View state:',
        this.syncAccountState
      );
    },

    async getZkSyncAcountState() {
      // Get nonce and all token balances
      this.syncAccountState = await this.syncWallet.getAccountState();
    },

    createEphemeralWallet() {
      const ephemeralWallet = new ethers.Wallet.createRandom();
      window.localStorage.setItem('ephemeral-mnemonic', ephemeralWallet.mnemonic.phrase);
    },

    getEphemeralWallet() {
      const mnemonic = window.localStorage.getItem('ephemeral-mnemonic');
      if (mnemonic === 'undefined') return undefined;
      return new ethers.Wallet.fromMnemonic(mnemonic);
    },

    getEphemeralWalletAddress() {
      const ephemeralWallet = this.getEphemeralWallet();
      if (!ephemeralWallet) return undefined;
      return ephemeralWallet.address;
    },

    async checkAndSetupSigningKey() {
      // To control assets in zkSync network, an account must register a separate public key once.
      // This is a one-time process
      if (!(await this.syncWallet.isSigningKeySet())) {
        if ((await this.syncWallet.getAccountId()) == undefined) {
          throw new Error('Unknown account');
        }

        const changePubkey = await this.syncWallet.setSigningKey();
        console.log('Waiting for transaction to be confirmed...');

        // Wait until the tx is committed
        await changePubkey.awaitReceipt();
        console.log('Signing key successfully set');
      }
    },

    async checkout() {
      try {
        console.log('Begin checkout process');
        // UI setup
        this.showAlert = false;
        this.isCheckoutInProgress = true;

        // Validate user inputs --------------------------------------------------------------------
        // Make sure recipient addresses are valid
        const donations = this.donations;
        donations[0].recipientAddress = await utils.getAddress(donations[0].recipientAddress);
        donations[1].recipientAddress = await utils.getAddress(donations[1].recipientAddress);
        console.log('✅ Validated recipient addresses');

        // Create ephemeral wallet -----------------------------------------------------------------
        await this.createEphemeralWallet();
        console.log(
          "✅ Ephemeral wallet created and saved to local storage. It's address is",
          this.getEphemeralWalletAddress()
        );

        // Log in and update state ----------------------------------------------------------------
        // Log in
        await this.loginToZkSync();

        // Token approvals -------------------------------------------------------------------------
        // None at the moment since we're only using ETH for now

        // Deposit ---------------------------------------------------------------------------------
        const depositAmount = String(
          donations.reduce(function (accumulator, currentValue) {
            return accumulator + Number(currentValue.amount);
          }, 0)
        );
        console.log(`✅ Total deposit amount is valid: ${depositAmount} ETH`);

        console.log(
          'Waiting for user to approve transaction to deposit funds to ephemeral wallet...'
        );
        const deposit = await this.syncWallet.depositToSyncFromEthereum({
          depositTo: this.getEphemeralWalletAddress(),
          token: 'ETH',
          amount: utils.parseEther(depositAmount),
        });
        console.log(
          '✅ Deposit transaction successfully sent. View deposit info and tx hash:',
          deposit
        );

        console.log('Waiting for transaction to be mined...');
        const txHash = deposit.ethTx.hash;
        const txReceipt = await this.ethersProvider.waitForTransaction(txHash);
        console.log('✅ Transaction mined. View receipt:', txReceipt);

        console.log(
          'Waiting for zkSync operator to issue promise that transaction will be processed...'
        );
        // If we do not wait for this receipt, the unlock step below will fail with
        // ":"Error: Failed to Set Signing Key: Account does not exist in the zkSync network"
        const depositReceipt = await deposit.awaitReceipt();
        console.log('✅ Deposit receipt received. View deposit receipt:', depositReceipt);

        // Unlock zkSync account -------------------------------------------------------------------
        // To control assets in zkSync network, an account must register a separate public key
        // once, so we now do that for the ephemeral keypair
        const ephemeralWallet = this.getEphemeralWallet();
        const ephemeralSyncWallet = await this.zksync.Wallet.fromEthSigner(
          ephemeralWallet,
          this.syncProvider
        );
        console.log('Registering ephemeral wallet on zkSync...');
        const changePubkey = await ephemeralSyncWallet.setSigningKey();
        await changePubkey.awaitReceipt(); // wait until the tx is committed
        console.log('✅ Ephemeral wallet is ready to use on zkSync');

        // Check status of deposit (will be committed, not verified) -------------------------------
        const ephemeralWalletState = await ephemeralSyncWallet.getAccountState();
        console.log('✅ State of ephemeral wallet retrieved. View state:', ephemeralWalletState);

        // Do the transfers ------------------------------------------------------------------------
        console.log('Sending all transfers...');
        for (let i = 0; i < donations.length; i += 1) {
          const donation = donations[i];
          console.log(`Begin transfer ${i + 1} of ${donations.length}`);
          const recipient = donation.recipientAddress;
          const amount = this.zksync.utils.closestPackableTransactionAmount(
            ethers.utils.parseEther(donation.amount)
          );
          const fee = await this.syncProvider.getTransactionFee(
            'Transfer',
            recipient,
            donation.tokenName
          );
          console.log('Using fee of:', fee);
          const transfer = await ephemeralSyncWallet.syncTransfer({
            to: recipient,
            token: donation.tokenName,
            amount: amount.sub(fee.totalFee),
            fee: fee.totalFee,
          });
          console.log('Transfer sent. Waiting for receipt. View transfer:', transfer);
          // We await the receipt to guarantee that transactions are sent sequentially,
          // which ensures the nonces are in the proper order. If we do not await the
          // receipt:
          //   1. We would need to track and update the nonce manually
          //   2. We would need to ensure that transactions are sent to/received by zkSync in
          //      order of the nonce, because they execute transactions in the order received,
          //      and do not reorder by nonce (this is ok because there is no penalty for failed
          //      transactions)
          // Source: zkSync Discord
          const transferReceipt = await transfer.awaitReceipt();
          console.log('✅ Transfer complete. View receipt:', transferReceipt);
        }
        console.log(
          'Ephemeral wallet still in storage. We keep it there until transactions are finalized on mainnet'
        );
        console.log('✅✅✅ Donations complete!');

        // Checkout complete -----------------------------------------------------------------------
        this.alertMessage = 'Your donation was successfully completed!';
        this.alertClass = 'alert-success';
        this.showAlert = true;
        this.isCheckoutInProgress = false;
      } catch (e) {
        this.handleError(e);
      }
    },

    async withdrawVerifiedBalance() {
      console.log('Withdraw back to Ethereum started');
      console.log(`Sending funds to ${this.userAddress}`);
      const amount = ethers.BigNumber.from(this.syncAccountState.verified.balances.ETH);
      console.log('amount: ', amount);
      const fee = await this.syncProvider.getTransactionFee('Withdraw', this.userAddress, 'ETH');
      console.log('Using fee of:', fee);
      const withdraw = await this.syncWallet.withdrawFromSyncToEthereum({
        ethAddress: this.userAddress,
        token: 'ETH',
        amount: amount.sub(fee.totalFee),
        fee: fee.totalFee,
      });
      console.log('✅ Withdraw sent to network. View details:', withdraw);
      console.log(
        'Assets will be withdrawn to the specified address after the zero-knowledge proof of the zkSync block with this operation is generated and verified by the mainnet contract. You may check the above transaction hash to see when this is completed. We will optimistically assume this will succeed, and it can take up to 15 minutes.'
      );
      // Commented out since this does not seem to work
      // console.log('Waiting for verification...');
      // await withdraw.awaitVerifyReceipt();
      console.log('✅✅✅ Withdraw complete!');
    },
  },
};
</script>

<style scoped>
.red {
  color: red;
}

.data-card {
  border: 1px solid rgba(0, 0, 0, 0.2);
  padding: 2rem;
  -webkit-box-shadow: 0 3px 3px rgba(0, 0, 0, 0.2); /* Safari 3-4, iOS 4.0.2 - 4.2, Android 2.3+ */
  -moz-box-shadow: 0 3px 3px rgba(0, 0, 0, 0.2); /* Firefox 3.5 - 3.6 */
  box-shadow: 0 3px 3px rgba(0, 0, 0, 0.2); /* Opera 10.5, IE 9, Firefox 4+, Chrome 6+, iOS 5 */
}

.address {
  font-size: 0.8rem;
}
</style>
