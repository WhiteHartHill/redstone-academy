# Contract deployment

## 🅰️ Setting up Arweave

As mentioned earlier, we will deploy our contract to RedStone public testnet. You will notice that is very similair to how we deployed contract to ArLocal. We just need to write a NodeJS script which will generate and fund Arweave wallet, read, contract source and initial state files and deploy contract to the testnet. At the end of this chapter, I'll show you how to repeat all these steps in order to deploy contract to Arweave mainnet.

Like the last time - firstly, we will need to declare variables that will be needed in the script. Head to [challenge/src/tools/deploy-test-contract.ts](https://github.com/redstone-finance/redstone-academy/blob/main/redstone-academy-pst/challenge/src/tools/deploy-test-contract.ts) and add following code:

```js
let contractSrc: string;

let wallet: JWKInterface;
let walletAddress: string;

let initialState: PstState;

let arweave: Arweave;
let smartweave: SmartWeave;
let pst: PstContract;
```

Then, we need to initialize Arweave, this time we will point to the RedStone testnet. Write following code in the asynchronous callback

```js
arweave = Arweave.init({
  host: 'testnet.redstone.tools',
  port: 443,
  protocol: 'https',
});
```

Exactly like the last time, we will set logging level to `error`:

```js
LoggerFactory.INST.logLevel('error');
```

Initialize Smartweave:

```js
smartweave = SmartWeaveNodeFactory.memCached(arweave);
```

Generate wallet and add some funds using our helper functions:

```js
wallet = await arweave.wallets.generate();
walletAddress = await arweave.wallets.jwkToAddress(wallet);
await addFunds(arweave, wallet);
```

Read contract source and initial state files:

```js
contractSrc = fs.readFileSync(
  path.join(__dirname, '../../dist/contract.js'),
  'utf8'
);
const stateFromFile: PstState = JSON.parse(
  fs.readFileSync(
    path.join(__dirname, '../../dist/contracts/initial-state.json'),
    'utf8'
  )
);
```

Override contract's owner address with the generated wallet address:

```js
initialState = {
  ...stateFromFile,
  ...{
    owner: walletAddress,
  },
};
```

And deploy contract using exactly the same method that in the tests. We will log contract id to the console, so we can relate to it while writing interactions.

```js
const contractTxId = await smartweave.createContract.deploy({
  wallet,
  initState: JSON.stringify(initialState),
  src: contractSrc,
});

console.log(contractTxId);
```

The script is ready!

## 🏃‍♀️ Running script

All we need to do now is run the prepared script. We will use typescript execution engine for NodeJS - ts-node. Run following script in the root folder:

```bash
ts-node src/tools/deploy-test-contract.ts
```

:::tip
Remember that you need to bundle contract source file before deploying the contract. We've already done it in [**Writing PST contract** chapter](../writing-pst-contract/contract-source.md#-bundling-contract).
:::

Congratulations!
You've just deployed contract to the RedStone testnet. You should see contract id in the console output. You can also visit [SonAR](htttps://sonar.redstone.tools), switch to the testnet and search your contract :)

## ➡️ Deploying to Arweave mainnet

There aren't many differences between deploying to testnet and mainnet. All you need to do is point to the right host:

```js
const arweave = Arweave.init({
  host: 'arweave.net',
  port: 443,
  protocol: 'https',
});
```

...and instead of using dynamically generated wallet you will need to use your real one. One way of doing that is using [ArConnect browser extension](https://www.arconnect.io/). You will receive your wallet's key in json format which you will need to put in your project. Remember to secure it correctly, e.g. by putting in `.secrets` folder and adding the folder to `.gitignore` file.

Now, the only thing you need to do is import the file with wallet key and point to it when deploying your contract:

```js
  const contractTxId = await smartweave.createContract.deploy({
    wallet: jwk as ArWallet,
    initState: initialState,
    src: contractSrc
  });
```

Remember to not make your jwk public!
After deploying the contract to the mainnet you can also view it in SonAR. You just need to wait for blocks to be mined. It can take up to 20 minutes.
