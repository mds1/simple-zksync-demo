<template>
  <div class="hello">
    <h1>
      zkSync Demo
    </h1>
    <p style="font-weight: bold; color: red;">
      This will work on mainnet and Rinkeby, so make sure to select the desired network
    </p>
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
      <button @click="depositEth">Deposit 0.01 ETH</button>
      <button @click="transferEth">Transfer 0.005 ETH</button>
      <button @click="withdrawEth">Withdraw 0.0045 ETH (some is lost as fees)</button>

      <div style="margin-top: 1rem;">
        <div>
          Enter an address in the field below, and it will be used for whichever action you select.
          So you can deposit to, transfer to, or withdraw to the address entered
        </div>
        <input v-model="inputAddress" />
      </div>
    </div>

    <div v-if="txHash" style="margin-top: 1rem;">
      <div>Transaction hash: {{ txHash }}</div>
      <div style="font-style: italic;">View on <a :href="zkScanUrl" target="_blank">zkScan</a></div>
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
      inputAddress: undefined,
      zksync: undefined,
      syncProvider: undefined,
      ethersProvider: undefined,
      signer: undefined,
      syncWallet: undefined,
      balances: {
        committedEth: undefined,
        verifiedEth: undefined,
      },
      txHash: undefined,
    };
  },

  computed: {
    zkScanUrl() {
      if (!this.txHash) return undefined;
      if (window.ethereum.chainId === '0x4' || window.ethereum.chainId === '0x04') {
        // rinkeby
        return `https://rinkeby.zkscan.io/explorer/transactions/${this.txHash.split(':')[1]}`;
      }
      return `https://zkscan.io/explorer/transactions/${this.txHash.split(':')[1]}`;
    },
  },

  async mounted() {
    console.log('------------------------ Start app setup ------------------------');
    await window.ethereum.enable();
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
     * @notice Throws if an invalid Ethereum address is provided, otherwise returns
     * checksummed address
     */
    async getChecksummedAddress() {
      return ethers.utils.getAddress(this.inputAddress);
    },

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
      // const userZksyncAddress = this.syncWallet.address();
      const deposit = await this.syncWallet.depositToSyncFromEthereum({
        depositTo: await this.getChecksummedAddress(),
        token: 'ETH',
        amount: ethers.utils.parseEther('0.01'),
      });
      console.log('deposit: ', deposit);

      // Await confirmation from the zkSync operator
      // Completes when a promise is issued to process the tx, i.e. zkSync state is updated
      const depositReceipt = await deposit.awaitReceipt();
      console.log(new Date(), 'depositReceipt: ', depositReceipt);
      console.log('This means we have received a promise to process the transaction');

      // Await verification
      // Completes when the state is finalized on Ethereum with a zk-proof
      const depositReceiptVerify = await deposit.awaitVerifyReceipt();
      console.log(new Date(), 'depositReceiptVerify: ', depositReceiptVerify);
      console.log('This means the transaction has been finalized on Ethereum');
      console.log('------------------------ End deposit ------------------------');
    },

    /**
     * @notice Transfer ETH to another user
     */
    async transferEth() {
      console.log('------------------------ Start transfer ------------------------');
      const recipient = '0xD2553382a60F121d9b1e35cFC9EBF4870FbCC96F';

      // We are sending 0.005 ETH with a 0.0001 fee
      const amount = this.zksync.utils.closestPackableTransactionAmount(
        ethers.utils.parseEther('0.005')
      );
      const fee = this.zksync.utils.closestPackableTransactionFee(
        ethers.utils.parseEther('0.0001')
      );
      console.log('transfer recipient: ', recipient);
      console.log('transfer amount: ', amount);
      console.log('transfer fee: ', fee);

      // Send the transaction
      console.log('this.getChecksummedAddress(): ', this.getChecksummedAddress());
      const transfer = await this.syncWallet.syncTransfer({
        to: await this.getChecksummedAddress(),
        token: 'ETH',
        amount,
        fee,
      });
      console.log('transfer: ', transfer);
      this.txHash = transfer.txHash;

      // TODO: transfer to the above address, then make sure they can withdraw it
      const transferReceipt = await transfer.awaitReceipt();
      console.log('transferReceipt: ', transferReceipt);

      console.log('------------------------ End transfer ------------------------');
    },

    /**
     * @notice Withdraws 0.005 ETH back to Ethereum
     */
    async withdrawEth() {
      console.log('------------------------ Start withdraw ------------------------');
      const amount = ethers.utils.parseEther('0.0045');
      const fee = this.zksync.utils.closestPackableTransactionFee(
        ethers.utils.parseEther('0.0005')
      );
      const mainnetAddress = await this.signer.getAddress();

      console.log('transfer recipient: ', mainnetAddress);
      console.log('transfer amount: ', amount);
      console.log('transfer fee: ', fee);

      const withdraw = await this.syncWallet.withdrawFromSyncToEthereum({
        ethAddress: await this.getChecksummedAddress(),
        token: 'ETH',
        amount,
        fee,
      });
      console.log('withdraw: ', withdraw);
      this.txHash = withdraw.txHash;

      // Wait for ZKP verification to complete
      await withdraw.awaitVerifyReceipt();
      console.log('------------------------ End withdraw ------------------------');
    },
  },
};
</script>
