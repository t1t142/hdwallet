# Build an Ethereum Wallet with Angular
Let's see how to build a simple Ethereum wallet with Angular using a BIP32 Hierarchical Deterministic Wallet (HDWallet).

> Warning: this is an educational purpose tutorial. Please do not use this method to store crypto assets from the mainnet.

## Setup
We will use Angular with `@angular/material` for the front-end and `ethers.js` for the cryptographic part. We use `ethers.js` instead of `web3` because it supports out-of-the-box HDWallet.

### Angular
First install `@angular/cli`: 
```bash
npm install -g @angular/cli
```

Then in the directory where you want to scaffold your project runs: 
```bash
ng new hdwallet --style scss --routing true --prefix hdwallet
cd hdwallet
ng add @angular/material 
```

Create a `ui` module where we'll export all `@angular/material` modules : 
```bash
ng generate module ui
```

Then imports the `UiModule` into your `AppModule`.

### Ethers
[Ethers.js](https://docs.ethers.io/ethers.js/html/getting-started.html) is a library written in Typescript that provides high and low level API to build Decentralized Applications (Dapps) on top of Ethereum.

Run : 
```bash
$> npm install ethers --save
```

##  Create an Ethereum Wallet
Here is how we will create the Ethereum wallet :
1. Generate a random 12 words long mnemonic based on [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
2. Use the mnemonic string to create an [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) HDWallet.
3. Derive an Ethereum private key from the HDWallet with a [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) path.
4. Encode the private key with a password. The encoded result is called `keystore`.
5. Store the keystore into the localstorage.

`ethers` handles 1, 2 and 3 with the method `Wallet.getRandom()`.

> BIP stands for "Bitcoin Improvement Proposals", like the EIP for Ethereum, but some cryptographic based improvements can be used for Ethereum.

### UI
We'll use the `@angular/material` [stepper](https://material.angular.io/components/stepper/overview) to handle this steps. 

In your `ui.module.ts` imports the `MatStepperModule` : 
```typescript
import { NgModule } from '@angular/core';
import { MatStepperModule } from '@angular/material/stepper';

@NgModule({
  exports: [MatStepperModule]
})
export class UiModule { }
```

Then create a component `generate`: 
```bash
ng generate component generate
```

And inside the `generate.component.html` add the stepper like in [this example](https://material.angular.io/components/stepper/examples).

### Mnemonic

#### BIP39
A BIP39 mnemonic is a group of easy to remember words to generate deterministic wallets. It's easier for human to handle words than hexadecimal values. The mnemonic can be written on paper making it a better, typo tolerant, backup for your private key.

#### Generate a Random Mnemonic
Inside `generate.component.ts` add a `mnemonic` property and a method that will create a random mnemonic with the right entropy.

```typescript
import { Component, OnInit } from '@angular/core';
import { Wallet } from 'ethers';

@Component({
  selector: 'hdwallet-generate',
  templateUrl: './generate.component.html',
  styleUrls: ['./generate.component.scss']
})
export class GenerateComponent implements OnInit {
  public wallet: Wallet;
  public mnemonic: string[];

  ngOnInit() {
    this.randomMnemonic();
  }

  /** Generate a random Mnemonic with the right entropy */
  public randomMnemonic() {
    this.wallet = Wallet.createRandom();
    this.mnemonic = this.wallet.mnemonic.split(' ');
  }
}

```

We want to show the mnemonic to the user so he can write them down. We use an Array for mnemonic to get the index of each word.

> This will generate 12 english words. To use other language you can use this [more complexe method](https://docs.ethers.io/ethers.js/html/cookbook-accounts.html#random-mnemonic).

#### Display the 12 Words

In the HTML display the words: 
```html
<!-- First Step -->
<mat-step>
  <ng-template matStepLabel>Write those words down</ng-template>
  <button mat-button (click)="randomMnemonic()">
    <mat-icon>autorenew</mat-icon>
    Generate new words
  </button>
  <mat-list>
    <mat-list-item *ngFor="let word of mnemonic; let i = index">
      {{ i + 1 }} - {{ word }}
    </mat-list-item>
  </mat-list>
</mat-step>
```

#### UX - Check the user has written down the words
Most of the users won't bother writting the mnemonic, but if they loose it, they won't be able to recover their wallet.

In the next step, let's add a form to test 3 random words.

```typescript
public createRandomWords() {
  const mnemonic = [...this.mnemonic];
  this.randomWords = Array(3).fill('').map(_ => {
    const rand = Math.floor(Math.random() * mnemonic.length);
    return mnemonic.splice(rand)[0];
  });
}
```

> This method should be called each time the user goes to the second step.

### Get an Ethereum keystore

#### HDWallet
The method `Wallet.createRandom()` generates a random mnemonic and creates a HDWallet under-the-hood.

A BIP32 HDWallet consists of a chain of keypairs. There is a lot of interesting cryptography here that we won't cover. You can find more [here](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).

#### Get a Private Key
To access a specific keypair in this tree, we'll need a derivation path. [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) defines a 5 levels path for handling multi-coins addresses, amongst other things. It looks like that : 
```
m / purpose' / coin_type' / account' / change / address_index
```
The path used by default by `Wallet.createRandom()` method is : `m/44'/60'/0'/0/0`.

> If you want a low-level access to the HDWallet you can use the [HDNode](https://docs.ethers.io/ethers.js/html/api-advanced.html#hdnode) class instead.

#### Encrypt the private key
On step 3 we should ask the user the enter and confirm this password.

```html
<!-- Third Step: Ask for a password -->
<mat-step>
  <ng-template matStepLabel>Enter password</ng-template>
  <form (ngSubmit)="createWallet()" [formGroup]="passwordForm" fxLayout="column">
    <ng-template matStepLabel>Fill out your name</ng-template>
    <mat-form-field>
      <input type="password" matInput placeholder="Password" formControlName="pwd" required />
    </mat-form-field>
    <mat-form-field>
      <input type="password" matInput placeholder="Confirm" formControlName="confirm" required />
    </mat-form-field>
    <button type="submit" mat-button>Create Wallet</button>
  </form>
</mat-step>
```

With this password we can now encrypt the private key and save the result into the `localstore`.

```typescript
/** Check password and confirm are the same */
public createWallet() {
  if (this.passwordForm.valid) {
    const pwd = this.passwordForm.get('pwd').value;
    this.encryptPrivatekey(pwd);
  }
}

/** Get a private key and encrypt it with a password */
private async encryptPrivatekey(password: string) {
  const keystore = await this.wallet.encrypt(password);
  localStorage.setItem('keystore', keystore);
}
```

## Display the Account
Now that we've stored the keystore, we should display the address and the Ether balance to the user.

### Display Component
// TODO

#### Routes and Guards
// TODO

#### Get the balance
// TODO

### Sign a Transaction
// TODO


## IPFS and Iframe
// TODO

### Host with IPFS
// TODO

### Use inside an Iframe 
// TODO

