# SBHack Workshop 2

This training session goes over a variety of topics that will prove themselves very useful when building projects that interact with the Casper Network. The topics are as follows:

* Smart contract installation, in which I'll show how to clone, compile, and install an example contract using `casper-client`
* Smart contract entrypoint invocation, where I'll show how to call entrypoints on your smart contracts and pass data to the Network
* Querying global state, which is useful to read data associated with your contract from the network
* Casper JS SDK integration, which can largely simplify the process of the above topics, as well as allow you to create your own dApps that run on Casper
* And lastly we'll go over the CEP-18 and CEP-78 standards. These standards represent Casper's fungible and enhanced non-fungible token standards respectively.

One quick thing before we begin, I'm using macOS Big Sur wish the `zsh` command line. I recommend following along either on macOS or Linux, and if you choose to use Windows, I recommend installing the Windows Subsystem for Linux and using the `bash` command line.

## Smart Contract Installation

Let's go ahead and get started with smart contract installation. For this walkthrough, we will be using the *counter* example contract provided in the *casper-node* repository

Start by opening a terminal and navigating to your preferred root working directory, I'll be using the *home* "`~`" directory

```bash
cd ~
```

Create a project directory, mine will be *sbhack-workshop-2*

```bash
mkdir sbhack
cd sbhack
```

Now clone the *casper-node* repository

```bash
git clone https://github.com/casper-network/casper-node.git
```

and `cd` into the directory

```bash
cd casper-node
```

Let's first take a look at the *counter-installer* contract and see what it does.

To do this open up *smart_contracts/contracts/tutorial/counter-installer/src/main.rs* in the editor of your choice.

Since the `call` function is always executed as soon as the contract is installed, let's see what it does.

On line 55, you can see that a new URef is created reference a 32 bit integer, which will be the counter.

Then on lines 58 through 60, the named key `COUNT_KEY` is initialized and set to a named key within the contract.

Then from lines 63 to 77 the entrypoints `COUNTER_INC` and `COUNTER_GET` are made available publicly

At line 79, the contract is stored in the global state, and towards the bottom, we allow the contract to be accessible both by its version and from the `COUNTER_KEY`

Now let's quickly look at the function `counter_inc` on line 31

We can see that all this function does is get the value at `COUNT_KEY` and increment it by 1.

First we need to prepare our working environment, do so by running

```bash
make setup-rs
```

Now to build the *counter-installer* contract run

```bash
make build-contract-rs/counter-installer
```

This will take a moment to complete

Now that this is complete we will be able to find our *counter_installer.wasm* contract in *target/wasm32-unknown-unknown/release/counter_installer.wasm*

Great, now we can use the casper-client to install this contract to the network

Before we do this though, we'll need an account funded with CSPR. For simplicity's sake, I'll be using Casper's testnet, so I can use the Faucet tool at https://testnet.cspr.live/tools/faucet

In my terminal I'll go back into my project directory and then run `casper-client keygen keys` which will generate me a keypair in the folder keys

Now I'll open up Chrome, or another Chromium based browser, and open the Casper Signer, click **Import Account** and import the new key I just created.

Now I'll go to the faucet and request some test CSPR

While we're waiting on this transaction to execute, let's grab the address of an online node. We can do this by going to https://testnet.cspr.live/tools/peers

I'll copy the address of a node and head over to the terminal

```bash
casper-client put-deploy \
--node-address http://<node_address>:7777/rpc \
--chain-name casper-test \
--secret-key keys/secret_key.pem \
--payment-amount 20000000000 \
--session-path casper-node/target/wasm32-unknown-unknown/release/counter_installer.wasm
```

I'll quickly make sure that the faucet transaction executed, and then I'll submit this deployment

As you can see, we get back a deploy hash, which we can look up using cspr.live.

Once the execution is complete, we can query the count by getting the value at named key `COUNT_KEY`

First, we will need the most up-to-date state root hash. A state root hash is the root at which to query on-chain data; if you use an outdated state root hash, you may not be able to query newer data.

To get the most recent state root hash, run the following command:

```bash
casper-client get-state-root-hash --node-address http://<node_address>:7777/rpc
```

You'll use the value returned to you in the next command.

Now copy your hexadecimal public key from the file *public_key_hex* that was generated from `casper-client keygen keys`. You may also optionally copy your public key from the Casper Signer or from [cspr.live](https://cspr.live).

Now run the following command, where `--state-root-hash` is the most recent state root hash, and `--key` is your account's public key:

```bash
casper-client query-global-state \
--node-address http://<node_address>:7777/rpc \
--state-root-hash 6538adf0d0e4068310edf6fe44e6abc561a481275f8cd40e3865f29f33c158d6 \
--key hash-d95b8625e0557e0aa7b9275252456da0796c0acf11b05b303f6ce13f4653766d \
-q "count"
```

In the response, you should find the "stored_value" which should be "0".

```json
"result": {
    "api_version": "1.4.8",
    "block_header": null,
    "merkle_proof": "[57854 hex chars]",
    "stored_value": {
      "CLValue": {
        "bytes": "00000000",
        "cl_type": "I32",
        "parsed": 0
      }
    }
  }
```

## Casper JS SDK integration

While the casper-client method of contract interaction works, it is not very convenient, especially when writing scripts and building dApps. For this, Casper has a variety of supported SDKs, the most popular of which being the TypeScript SDK. The TypeScript SDK is known canonically as the Casper JS SDK as it compiles down to JavaScript. We will utilize the Casper JS SDK to make the process of installing the contract simpler. Then, we will invoke the `increment` entrypoint from the JS SDK and query the stored value to validate that it worked.

In your terminal, run

```bash
npm init -y
```

Now install the Casper JS SDK

```bash
npm install casper-js-sdk
```

Create the file *index.js*

```bash
touch index.js
```

Start off on the first line by importing some classes from the JS SDK. We'll also need the package `fs` which comes bundled with Node.js in order to read in our WASM contract.

```javascript
const { CasperClient, Contracts, Keys, RuntimeArgs } = require("casper-js-sdk")
const fs = require('fs')
```

Next let's set up some necessary objects. The Casper JS SDK uses the class `CasperClient` as an interface between your program and live nodes on the network. Setting up a `CasperClient` object is easy, simply instantiate the object with a node url like so:

```javascript
const client = new CasperClient("http://<node_address>:7777/rpc")
```

Set up a new `Contract` instance as well, constructing it with the `client` you just created.

```javascript
const contract = new Contracts.Contract(client)
```

Now we need to load in our signing key, we'll use the one we created and funded earlier.

Considering you used `casper-client keygen` with no flags, your key will be of encryption algorithm `Ed25519`. If your key uses Secp256k1 encryption, there is a subclass for that too.

```javascript
const keys = Keys.Ed25519.loadKeyPairFromPrivateFile("./keys/secret_key.pem")
```

The last important constant we'll need is our compiled smart contract as a byte array. We can utilize the `fs` package to do this easily

```javascript
const wasm = new Uint8Array(fs.readFileSync("casper-node/target/wasm32-unknown-unknown/release/counter_installer.wasm").buffer)
```

Now that we've done this we can begin writing some actual logic. To get started I'm going to paste in the functions `pollDeployment` and `iterateTransforms` and just go over what they do. 

```javascript
function pollDeployment(deployHash) {
  return new Promise((resolve, reject) => {
    var poll = setInterval(
      async function (deployHash) {
        try {
          const response = await client.getDeploy(deployHash);
          if (response[1].execution_results.length != 0) {
            //Deploy executed
            if (response[1].execution_results[0].result.Failure != null) {
              clearInterval(poll);
              reject("Deployment failed");
              return;
            }
            clearInterval(poll);
            resolve(response[1].execution_results[0].result.Success);
          }
        } catch (error) {
          console.error(error);
        }
      },
      2000,
      deployHash
    );
  });
}

function iterateTransforms(result) {
  const transforms = result.effect.transforms;
  for (var i = 0; i < transforms.length; i++) {
    if (transforms[i].transform == "WriteContract") {
      return transforms[i].key;
    }
  }
}
```

The `pollDeployment` function makes use of the method `getDeploy` in the `CasperClient` class to check the results of the execution.
If the "execution_results" object in the response is empty, the deploy has yet to be executed. Once it is executed, it will either contain the object "Failure" or "Success", this is how we can check the status of the deploy. This function runs on an interval of 2 seconds until execution is recorded.

The `iterateTransforms` function looks at all of the state changes that occured during the transaction; if the state-change "WriteContract" was made, we know that the associated value is the hash of the contract, which we can used to query the named key "count" and get our counter's value.

Anyway, now that we have these necessary functions, let's begin the fun part and write the `installContract` function.

```javascript
function installContract() {
  const deploy = contract.install(
    wasm,
    RuntimeArgs.fromMap({}),
    "20000000000", //20 CSPR
    keys.publicKey,
    "casper-test",
    [keys]
  );

  client.putDeploy(deploy).then((deployHash) => {
      pollDeployment(deployHash).then((result) => {
        const contractAddress = iterateTransforms(result)
        console.log(contractAddress)
      })
    })
    .catch((error) => {
      console.error(error)
    })
}
```

The `deploy` const is made up of the WASM contract as a byte array, an empty map of runtime arguments, a gas payment amount of 20 billion motes (or 20 CSPR), the public key of the deploying account, the name of the network (in our case, "casper-test"), and an array of the signing keys, which in our case is just one keypair.

Then the `putDeploy` method within the `CasperClient` class and the deploy is sent to the network. Once it's successfully been deployed to the node, a deploy hash is returned. We then use this deploy hash to poll the status of the deploy, and once it has been executed, we iterate through the transformations until we find the "WriteContract" key-value pair, which is printed to the console.

Now that we have a way to install the contract and obtain its address, we can write our getter function which I'll call `getCount`.

```javascript
function getCount(contractAddress) {
  return new Promise((resolve, reject) => {
    contract.setContractHash(contractAddress)
    contract.queryContractData(["count"]).then((response) => {
        resolve(response)
      }).catch((error) => {
        reject(error)
      })
  })
}
```

This function has one parameter `contractAddress` which is tied to the `Contract` instance within the function.

After we set the contract address we can query the contract's named keys, and we are interested in the key named "count".

Finally we can resolve our function with the response of `queryContractData`

Let's test out these functions.

At the bottom of the file, insert the function call `installContract()`

```javascript
...
...
installContract()
```

and in your terminal run

```bash
node index.js
```

Once the deployment is executed, the contract's address should be printed to the console.

Copy this address, and make another function call `getCount(contractAddress)` and comment out the `installContract()` function call

```javascript
...
...
//installContract()
getCount("hash-contractaddress").then((count) => {
  console.log(count)
})
```

Save and rerun `node index.js`

```bash
node index.js
```

Finally, let's write the `increment` function that will call the `counter_inc`rement entrypoint and add 1 to the stored count

```javascript
function increment(contractAddress) {
    return new Promise((resolve, reject) => {
      contract.setContractHash(contractAddress)
      const deploy = contract.callEntrypoint(
        "counter_inc",
        RuntimeArgs.fromMap({}),
        keys.publicKey,
        "casper-test",
        1000000000,
        [keys]
      )

      client.putDeploy(deploy).then((deployHash) => {
        pollDeployment(deployHash).then((result) => {
            resolve(result)
        }).catch((error) => {
            reject(error)
        })
      })
    })
  }
```

Similarly to the `installContract` method, we `put` and `poll` the deployment the same way, but do not iterate the transformations because no contract address is created and we don't need any other transformation data.

Dissimilarly to `installContact`, we instead use the `callEntrypoint` method within the `Contract` class (instead of `install`) as we are creating a deploy that will invoke an entrypoint.

The `callEntrypoint` method in this case takes the following arguments:

* "counter_inc" which is the stored entrypoint name
* An empty RuntimeArgs object as no arguments are required in this case
* The public key of the sender, in this case also the signing account
* The chain name, in our case "casper-test"
* The gas payment, 1 billion motes or 1 CSPR
* An array of the signing keys, in our case only our one account

The result that is printed should just be the raw execution results. We can verify that the count did in fact increment by calling `getCount` after we call `increment`.

Let's try this out by executing the following code, using the same contract address as before:

```javascript
increment("hash-contractaddress").then(() => {
  getCount("hash-contractaddress").then((count) => {
  	console.log(count)
	})
})
```

## CEP-18 and CEP-78 Standard

When programming assets on Casper, like on many other blockchains, there are standards defined for creating assets like fungible and non-fungible tokens. While it's possible to use standards that were devised under the direction of other blockchains or independent researchers, Casper attempts to define its own token standards to best make use of the technologies available on the Casper Network.

### Fungible - CEP-18

First off let's talk about CEP-18 which is Casper's fungible token standard. This can be compared to ERC-20, and we will do so in a moment, but basically think of fungible tokens like USDC or Wrapped ETH or any other token that can be transacted with. So let's just go ahead and read through the standard:
https://github.com/casper-network/ceps/blob/master/text/0018-token-standard.md

### Non-fungible - CEP-78

Now let's move on to Casper's new enhanced NFT standard, CEP-78. In the past Casper used the CEP-47 NFT standard

https://github.com/casper-network/ceps/blob/master/text/0047-casper-nft-protocol.md

however CasperLabs has recently put together its new enhanced NFT standard that adds a lot more functionality to NFTs on Casper, and this is the standard which is to be used on Casper moving forward.

The main idea of CEP-78 is that by introducing a set of selectable options into the actual contract logic, NFT contract installers can build out their project to their own specifications without needing to make adjustments to the contract code itself; These options are known as "modalities".

First, read through the Design Goals, and go through each of the modalities to learn their purpose.



## Conclusion

This training session has been recorded and will be available for you to watch again. A link will be in the Github repository for this session.

https://github.com/casper-ecosystem/sbhack22-workshop-2

Another resource that you may find useful is the repository of Workshop #2 as part of the Smart India Hackathon which took place a couple months ago. There is an associated recorded presentation with this workshop as well, and it is linked in the README. This workshop goes over installing your own custom CEP-78 contract using the Casper JS SDK, and can be used to deploy your own NFT projects, or integrated into your own project. I'll send the link for the Smart India Hackathon Workshop #2 now as well

https://github.com/casper-ecosystem/sih-workshop-2

Lastly I'd like to share with you the repository of Workshop #1 as part of the Ready Player Casper hackathon, which will teach you how to integrate the Casper Signer into your project and sign and send deploys from the front-end. Sending this link in the chat now

https://github.com/casper-ecosystem/rpc-workshop-1
