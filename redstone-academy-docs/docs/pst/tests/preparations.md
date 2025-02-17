# Preparations

It is more than important to carefully test your contract before deploying it to the blockchain. We will write few tests and execute them in server environment as well as in browser one using Jest. Throughout this process you will get to know couple of important **ArweaveJS** and **RedStone Smartweave SDK** methods.

## 🔤 Declaring variables

Head to [redstone-academy-pst/challenge/tests/contract.test.ts](https://github.com/redstone-finance/redstone-academy/tree/main/redstone-academy-pst/challenge/tests/contract.test.ts) and start by declaring all necessary variables in `beforeAll` - a callback which will be executed before all the tests. Remeber to import all the interfaces!

```js
let contractSrc: string;
let wallet: JWKInterface;
let walletAddress: string;
let initialState: PstState;
let arweave: Arweave;
let arlocal: ArLocal;
let smartweave: SmartWeave;
let pst: PstContract;
```

## 🅰️ Setting up ArLocal and instantiating Arweave

```js
arlocal = new ArLocal(1820);
await arlocal.start();

arweave = Arweave.init({
  host: 'localhost',
  port: 1820,
  protocol: 'http',
});
```

We initialize ArLocal - a local server resembling real Arweave network - on port 1820 (the default one is 1984) and start the instance. Then, we initialize Arweave by pointing to the running local testnet.

## 🎛️ LoggerFactory

```js
LoggerFactory.INST.logLevel('error');
```

Logging is helpful not only during development process but also when interacting with the contract as it gives some advices of what is happening and what errors have occured. RedStone Smartweave SDK has seven levels of logging, you can view all logging implementations in the SDK [https://github.com/redstone-finance/redstone-smartcontracts/tree/main/src/logging](https://github.com/redstone-finance/redstone-smartcontracts/tree/main/src/logging). It is adviced to use only `fatal` or `error` logging levels in production as using other ones may slow down the evaluation. You can also use different logging levels for each module. Names of the modules are derived from Typescript classes in the SDK, example of such implementation:

```js
LoggerFactory.INST.logLevel('fatal');
LoggerFactory.INST.logLevel('debug', 'ArweaveGatewayInteractionsLoader');
```

For the tutorial purposes we will set `error` logging level.

## 🪡 Setting up SmartWeave

```js
smartweave = SmartWeaveNodeFactory.memCached(arweave);
```

SmartWeave class in SDK is a base class that supplies the implementation of SmartWeave protocol. Check it out in SDK [https://github.com/redstone-finance/redstone-smartcontracts/blob/main/src/core/SmartWeave.ts](https://github.com/redstone-finance/redstone-smartcontracts/blob/main/src/core/SmartWeave.ts). SmartWeave allows to plug-in different module implementations (like interactions loader or state evaluator) but as it is just the first tutorial, we will go with the most basic implementation. We will create SmartWeave instance by using `SmartWeaveNodeFactory` which is designed to be used in node environment. We will also use `MemCache` which a simple in-memory cache. Later in the Academy schedule you will be introduced to other types of chache (like `fileCached` or `knexCached`).

## 👛 Generating wallet and adding funds

```js
wallet = await arweave.wallets.generate();
walletAddress = await arweave.wallets.jwkToAddress(wallet);
```

In order for tests to be working we need to generate a wallet which will be connected to the contract and therefore responsible for signing the transactions. We also need to get its address. We do it using ArweaveJS SDK. We advise you to read [this documentation of wallets](https://github.com/ArweaveTeam/arweave-js#wallets-and-keys).

We also need to fund the wallet with some tokens. Head to [redstone-academy-pst/challenge/utils/\_helpers.ts](https://github.com/redstone-finance/redstone-academy/tree/main/redstone-academy-pst/challenge/utils/_helpers.ts) and write this asynchronous helper function.

```js
export async function addFunds(arweave: Arweave, wallet: JWKInterface) {
  const walletAddress = await arweave.wallets.getAddress(wallet);
  await arweave.api.get(`/mint/${walletAddress}/1000000000000000`);
}
```

It takes two arguments - arweave instance and generated wallet. It declares wallet address using ArweaveJs and hit the `mint` Arweave endpoint which is responsible for minting tokens. Keep in mind that above value is an amount of winstons.

:::info
1 Winston = 0.000000000001 AR
:::

No go back to the test and call the `addFunds` function:

```js
await addFunds(arweave, wallet);
```

## 📰 Reading contract source and initial state files

Ok, back to the test. Now, we need to find a way to read files with contract source and initial state we've prepared in the last section. In order to do that we will use NodeJS method `readFileSync`. Remember to import `fs` and `path` modules. We are pointing to the files we've prepared in [the last section](../writing-pst-contract/contract-source#-bundling-contract) using esbuild and prepared scripts.

```js
contractSrc = fs.readFileSync(
  path.join(__dirname, '../dist/contract.js'),
  'utf8'
);
const stateFromFile: PstState = JSON.parse(
  fs.readFileSync(
    path.join(__dirname, '../dist/contracts/initial-state.json'),
    'utf8'
  )
);
```

## ✍🏻 Updating initial state

```js
initialState = {
  ...stateFromFile,
  ...{
    owner: walletAddress,
  },
};
```

We need to update our initial state and set previously generated wallet address as the owner of the contract. Please remember that this step is only needed in case of dynamically generated wallet (usually when we are using testnets).

## 🪗 Deploying contract

Now, the moment we were all waiting for!

```js
const contractTxId = await smartweave.createContract.deploy({
  wallet,
  initState: JSON.stringify(initialState),
  src: contractSrc,
});
```

We are using RedStone Smartweave SDK's `deploy` method to deploy the contract. You can view the implementation here [https://github.com/redstone-finance/redstone-smartcontracts/blob/main/src/core/modules/impl/DefaultCreateContract.ts](https://github.com/redstone-finance/redstone-smartcontracts/blob/main/src/core/modules/impl/DefaultCreateContract.ts#L12).

What it does is take object with **wallet**, **initial state** and **contract source** as contract data, create transaction using ArweaveJS SDK, add tags specific for SmartWeave contract (we will discuss tags in the next paragraph), sign transaction using generated Arweave wallet and post it to the Arweave blockchain. If your are not familiar with creating transactions on Arweave we strongly recommend reading ArweaveJS documentation, it's the key part to understand transactions flow, you can read more about it [here](https://github.com/ArweaveTeam/arweave-js#transactions).

## 🏷️ Tags

Tags are used to add metadata to the transaction, it helps documenting the data related to the contract. RedStone SmartWeave add following tags to the contract

```js
{
  name: 'App-Name';
  value: 'SmartWeaveContractSource';
}
{
  name: 'App-Version';
  value: '0.3.0';
}
{
  name: 'Content-Type';
  value: 'application/javascript';
}
```

You can of course override them or add some more tags by passing property with `tags` key to the contract data passed to `deploy` function.

## 🔌 Connecting to the pst contract

```js
pst = smartweave.pst(contractTxId).connect(wallet);
```

In order to perform any operation on the contract we need to connect to it. You can connect to any contract using SDK's `contract` method. But in case of pst contract it is recommended to connect to contract by using `pst` method which allows to use all the functions which are specific for PST Contract implementation. You can view `connect` and `pst` methods in [`SmartWeave` class](https://github.com/redstone-finance/redstone-smartcontracts/blob/main/src/core/SmartWeave.ts#L47). All methods specific for PST contracts can be viewed in [PstContract interface](https://github.com/redstone-finance/redstone-smartcontracts/blob/main/src/contract/PstContract.ts#L73).

We then connect our wallet to the pst contract. Please remember that connecting a wallet MAY be done before `viewState` (depending on contract implementation, ie. whether called contract's function required `caller` info). Connecting a wallet MUST be done before `writeInteraction`. And therefore, it is not required when we just want to read the state.

## 🚧 Mining blocks

As you may recall from the Elementary section, blockchain mining means adding transactions to the blockchain ledger of transactions. On mainnet in order to mine a block it is required for nodes to validate a transaction, when using ArLocal we need to mine a block manually. We will add another helper function to the `_helpers.ts` file:

```js
export async function mineBlock(arweave: Arweave) {
  await arweave.api.get('mine');
}
```

We need to call this function so the transaction will be available on Arweave instance. As we deployed a contract in previous step, let's finish our `beforeAll` callback with calling `mineBlock` function:

```js
await mineBlock(arweave);
```

## 🛑 Stopping ArLocal

After all tests will be executed, we will need to stop ArLocal instance. Add following code to the `afterAll` function:

```js
await arlocal.stop();
```

## 📜 Test scripts

It is good to test contract in different environments. We will test it in server environment as well as browser one. Jest executes tests on server so we need to add some additional files in order for the browser tests to work. It's already prepared but if you want to get familiar with how it works see these files: [challenge/jest.browser.config.js](https://github.com/redstone-finance/redstone-academy/blob/main/redstone-academy-pst/challenge/jest.browser.config.js) and [challenge/browser-jest-env.js](https://github.com/redstone-finance/redstone-academy/blob/main/redstone-academy-pst/challenge/browser-jest-env.js). One last bit is to add scripts to `package.json` files which will give us possibility to run node tests, run browser tests or run them both.

```json
    "test": "yarn test:node && yarn test:browser",
    "test:node": "jest tests",
    "test:browser": "jest tests --config ./jest.browser.config.js"
```

### Conclusion

Ok, a ton of work done! Let's write some tests!
