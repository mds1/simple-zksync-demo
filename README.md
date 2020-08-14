# zksync-demo-vue

## Architecture

This is the initial, simplest architecture that removes the need for multiple signatures for each transaction.

### On Page Load

1. User connects wallet
2. From the web3 provider, get an `ethersProvider` and an ethers signer instance, `signer`
3. Save off checksummed address, `userAddress`
4. Import `zksync`
5. A zkSync provider, `syncProvider` is defined based on the connected network (mainnet or Rinkeby)

Everything is now initialized and ready to go. When user initializes checkout process, we'll move to the next section

### On Checkout

Everything in this section is triggered when the user clicks the checkout button. Note that code snippets may not work exactly as shown below.

#### Sign In

1. Call `const syncWallet = await zksync.Wallet.fromEthSigner(signer, syncProvider)` to prompt login
2. Check `await this.syncWallet.isSigningKeySet()`
   1. If true, user has used zkSync and we do not need to do anything else here
   2. If false, user has never used zkSync and must register their signing key.
      ```javascript
      // Set the signing key
      const changePubkeyTx = await this.syncWallet.setSigningKey();
      // Wait for the zkSync transaction to be committed
      await changePubkeyTx.awaitReceipt();
      ```
3. Use `const accountState = await this.syncWallet.getAccountState()` to get a list of all tokens/balances the user has on zkSync

At this point the user can now deposit funds into zkSync.

### Deposit Funds

1. Look in local storage for an ephemeral wallet

   1. If one is there, TODO. Throw error?
   2. If one is not there, generate an ephemeral wallet and save it to local storage

   ```javascript
   // Generate random wallet and save it to local storage
   const ephemeralWallet = new ethers.Wallet.createRandom();
   const ephemeralWalletAddress = ephemeralWallet.address;
   localStorage.setItem('ephemeral-mnemonic', ephemeralWallet.mnemonic.phrase);
   ```

2. Handle token approvals the same way they are currently handled
3. After the final one call deposit to zkSync (sample code only handles case when 1 currency is deposited)
   ```javascript
   const deposit = await syncWallet.depositToSyncFromEthereum({
     depositTo: ephemeralWalletAddress, // if user only has 1 donation in cart, enter grant address here
     token: 'ETH',
     amount: totalAmountThatWasApproved,
   });
   const txHash = deposit.ethTx.hash; // L1 tx hash, i.e. view on etherscan.io
   ```
4. At this point we wait for the deposit receipt. Afterwards, we can set the signing key
   ```javascript
   // Wait for receipt
   const depositReceipt = await deposit.awaitReceipt();
   // Initialize corresponding zkSync account
   const syncEphemeralWallet = await zksync.Wallet.fromEthSigner(ephemeralWallet, syncProvider);
   // Set the signing key. Because this is a new wallet, this step is always required
   const changeEphemeralPubkeyTx = await this.syncWallet.setSigningKey();
   await changeEphemeralPubkeyTx.awaitReceipt();
   ```
5. Send off each transaction based on the user's cart

   ```javascript
   // Loop through each donation and transfer
   // Because there are n donations and n transaction hashes, we may want to link
   // users to their account page on zkscan, instead of linking to a specific tx
   for (let i = 0; i < donations.length; i += 1 {
     // Get amount and fee
     const amount = this.zksync.utils.closestPackableTransactionAmount(donations[i].donationAmount);
     const fee = await syncProvider.getTransactionFee({
       txType: 'Transfer',
       address: donations[i].recipentAddress,
       tokenLike: 'ETH'
     });

     // Send transfer
     const transfer = await this.syncWallet.syncTransfer({
        to: donations[i].recipentAddress,
        token: 'ETH',
        amount: amount - fee,
        fee,
      });
      const txHash = transfer.txHash; // L2 tx hash, i.e. view on zkscan.io

      // Check transfer status
      const transferReceipt = await transfer.awaitReceipt();
   }

   // Remove the ephemeral wallet from storage
   localStorage.removeItem('ephemeral-mnemonic');
   ```

Done!

In this deposit flow, it is important to note that Gitcoin (more specifically,
the frontend) temporarily has full control over where funds go after once
they are transferred to the ephemeral wallet

## Development

```
yarn install
```

### Compiles and hot-reloads for development

```
yarn serve
```

### Compiles and minifies for production

```
yarn build
```

### Lints and fixes files

```
yarn lint
```

### Customize configuration

See [Configuration Reference](https://cli.vuejs.org/config/).
