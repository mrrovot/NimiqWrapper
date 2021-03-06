# NimiqWrapper
NimiqWrapper.js is a file containing 3 classes that allow for easier use of the [Nimiq Javascript API](https://github.com/nimiq-network/core).  The 3 classes cover wrapping Nimiq as a whole, wrapping the Nimiq browser miner (pool implementation only), and a final class of utility functions.  My goal when creating this library was to help lower the bar for developers trying to work with Nimiq.  I believe Nimiq has much potential, going far beyond simply mining in the browser, and I can't wait to see what the community creates (hopefully with the help of NimiqWrapper).

#### Let's all work towards a Richer future!

Both wrapping classes were designed in such a way that all interesting events can be listened to using callback functions and both wrapped objects are accessible to the programmer in order to perform advanced functions.  This was done in order to have maximum compatibility with AngularJS but works just as well with NodeJS or any other UI framework.  For UI frameworks, the only thing to remember is that the callbacks may be called outside of the digest cycle / whatever your UI framework calls it.  There's a section in this readme specifically about dealing with this issue in AngularJS and if anyone solves any issues with other frameworks, they can submit a pull request to add a section.

## General Details
### Getting Started
In order to fully connect to the Nimiq Network,, two objects are required.  The first is the `InitInfo` object and the second is the `HandlerFunctions` object.  The `InitInfo` object is meant to contain information on the wallet, mining address, and other information needed to connect to the Nimiq Network.  This object will be passed to the connect(...) function of the NimiqWrapper object.  The `HandlerFunctions` object is meant to contain the callback functions used to obtain information from the wrapper.  This object will be passed to the constructor of the NimiqWrapper object.  Descriptions of the possible fields in each object are given below, and if not present default values will be used.

### InitInfo Object
 * walletSeed
   * Default Value: None, no wallet will be initalized and a message will be logged in the console.
   * Purpose: To define the encrypted serialized form of the wallet to be loaded.
 * walletKey
   * Default Value: None, see above.
   * Purpose: To define the key used to encrypt the walletSeed
 * poolHost
   * Default Value: pool.porkypool.com
   * Purpose: To define the host of the pool to mine with.  Defaults to porkypool.
 * poolPort
   * Default Value: 8444
   * Purpose: To define the port of the pool to mine with.  Defaults to porkypool:8444.
 * mineToAddr
   * Default Value: Nimiq Null Address
   * Purpose: To define the address (in friendly form) to mine to.
### HandlerFunctions Object
 * errorHandler(where, err)
   * This handler function is called when an error occurs in the Nimiq Wrapper.
   * `where` will be a string which references where in the code the error occured.
     * This string will become more detailed over time, and will be useful for debugging.
   * `err` will be the message of the error.
     * This may be replaced by the actual error object or a third parameter will include the traceback.
 * minerChanged(status)
   * This handler function is called when the miner state changes.
   * Possible values of `status` are: "started" and "stopped".
 * connectionState(status)
   * This handler function is called when the connection to the pool changes.
   * Possible values of `status` are: "connected" and "failed".
 * consensus(status)
   * This handler function is called when a major change in consesnsus occurs.
   * Possible values of `status` are: "syncing", "established", and "lost"
 * syncStatus(status)
   * This handler function is called while obtaining consensus and provides updates on which step of syncing the wrapper is at.
   * Possible values of `status` are: "sync-chain-proof", "verify-chain-proof", "sync-accounts-tree", "verify-accounts-tree", and "sync-finalize"
   * These 5 states will be seen in order although sometimes extremely quickly or with some states being skipped.
 * peersChanged()
   * This handler function is called when the number of peers changes.
   * This is one of two functions that accepts no parameter.
   * This handler is useful in reporting any information during the sync as it will be called each time a peer connects or is lost.
     * Do keep in mind though that this handler will be called after the sync and for the remainder of the running time of the program.
     * It is called EVERY time the number of peers changes so don't treat it as a function only called during initialization.
 * headChanged()
   * This handler function is called each time the head of the blockchain changes.
   * This is a good time to update the block height, as well as run any checks for you application.
   * This function should be called once per block as they are produced, although during sync it can be called multiple times as past blocks are retrieved.
   * This will most likely be the most used handler of them all as it's useful for watching the blockchain and reacting to any changes that may have happened in the most recent block.
 * peerJoined(peer)
   * This handler function is called each time a peer joins and provides an object with information on that peer.
### Minimum Necessary Code
```
let handlers = {
    consensus: (status) => console.log("Consensus: " + status),
    headChanged: () => console.log("Now at height: " + wrapper.blockHeight)
};

//Neither of the handlers above or the mineToAddr field are strictly necessary.
//They are however useful for a minimum connection that simply mines for you in place of advertisement.

let inits = {
    mineToAddr: "NQ07 0000 0000 0000 0000 0000 0000 0000 0000"
};

let wrapper = new NimiqWrapper(true, handlers);
let cancelConnect = setInterval(function () {
    let result = wrapper.connect(inits);

    if (result) {
        console.log('Nimiq loaded. Connecting and establishing consensus.');
        clearInterval(cancelConnect);
    }
}, 1000);
```
### Constructor and Connect Functions
The constructor for the NimiqWrapper object takes 3 parameters with the last being optional.
 * `mine` specifies whether the miner should begin once consensus is reached.
 * `handlerFunctions` is the `HandlerFunctions` object which specifies all of the necessary handler functions you'd like to define.
   * Any callbacks not specified in this object use default functions that simply log appropriate console output.
 * `full` specifies whether a full node should be initialized instead of a light node.
   * This parameter is optional and defaults to false if not included.
   
The connect function in the NimiqWrapper object takes 1 parameter being the the `InitInfo` object.

## Implementation Details
### Standard JavaScript
Implementing NimiqWrapper in a site using standard JavaScript and HTML is quite easy.  Simply create whatever handler functions you deem necessary (most likely headHandler will be a minimum requirement) and have those functions update your UI as necessary.  This can be done via changing JavaScript variables and handling updating the UI seperately, or by manually manipulating the DOM to update the UI.  An example of how this can be done can be seen in the demo.html file in this repo, which is hosted live [on my site](https://www.drawpad.org/nimiq/demo.html).
### AngularJS
NimiqWrapper was designed with AngularJS in mind and so using the two together is quite easy.  Similar to the JavaScript section, you will need to create any necessary handler functions and then pass them to the constructor of the NimiqWrapper object.  The important difference with AngularJS is that you must remember to update angular variables within the digest cycle.  If you know nothing of the digest cycle, some introductory reading can be found [here](https://www.sitepoint.com/understanding-angulars-apply-digest/) and [here](https://www.thinkful.com/projects/understanding-the-digest-cycle-528/).  Once you understand where the issue is, solving it is not too hard.  `$apply` and `$timeout` are the two most well known solutions, with $scope.`$evalAsync` being the best I've found.  

`$apply` was my preferred solution however it can run into issues if a digest cycle is already running.  You can use `$apply` pretty reliably in most of the handlers except for headHandler.  The headHandler function has the possibility of being called quite often (as blocks roll in or while you're achieving concensus for the first time) and some of these calls will collide with the current digest cycle.  One solution to colliding with a running digest is to use `$timeout`.  I won't go into much detail on this solution as it's a common workaround documented all over the internet.  I've always tried to avoid it, but unfortunately `$apply` just isn't reliable enough for use with NimiqWrapper.

After being unhappy with using `$timeout` in such a way, I did some research and landed upon `$evalAsync` which is available in later versions of AngularJS.  A very educational overview of this function can be seen [here](https://www.bennadel.com/blog/2605-scope-evalasync-vs-timeout-in-angularjs.htm) and the same author has an side by side comparison of `$evalAsync` and a similar function [here](https://www.bennadel.com/blog/2751-scope-applyasync-vs-scope-evalasync-in-angularjs-1-3.htm).  `$evalAsync` can be used in the same way as `$apply`, except it will always ensure that the function is run at the end of the current digest or start of the next.  This is much faster than `$timeout`, get's the point clearly across, and fixes the issue with trying to digest in the middle of a digest cycle.
### NodeJS
I'm not too experienced with NodeJS although I figured my experience with JavaScript would carry over.  While the code seems like it should work fine, I'm having issues exporting the classes properly as a NodeJS module.  If anyone would like to volunteer to help me set it up, I can be reached on the Nimiq discord (@Chugwig) or a pull request can be made on this repo.
### Other Implementations
If anyone uses NimiqWrapper with a framework not discussed above, please reach out to me or issue a pull request detailing any implementation details of that framework.  I imagine that NimiqWrapper won't work perfectly on at least one framework, but it was designed to work in most environments and I would like to document any exceptions to that.

## Accessing the Wrapped Objects
As a developer, I like to reduce the amount of repeated code across my projects while still maintaining a close to the bone feel.  With NimiqWrapper.js you can do most basic operations while only working with the NimiqWrapper and MinerWrapper objects, but both wrappers give access to the wrapped objects.  Descriptions of the four objects most developers will be working with are below:
 * NimiqWrapper object (referred to as `wrapper` as needed in the bullets)
   * This object is the meat and bones of NimiqWraper.js and initializes the Nimiq engine.
   * Getters are provided to obtain commonly needed values.
     * Global hash rate, block reward, block height, etc...
     * Further getters can (and should) be added in future updates.
 * MinerWrapper object (accessed through `wrapper.miner`)
   * The MinerWrapper object is created by and stored within the NimiqWrapper object.
   * It can be accessed by using the getter function in your NimiqWrapper object.
   * Getters are provided in this object as well, but less information is available.
     * Global hash rate (repeated on purpose), hashrate (current H/s of the miner), and the reward per hour (in NIM)
     * Getters may be added to NimiqWrapper in the future to access these values without needing the MinerWrapper object.
 * `wrapper.wrappedInstance` (`wrapper.nimiqInstance`)
   * This is the officially supported way to access the wrapped Nimiq object.
   * This object has been populated with the following fields:
     * `consensus`, `blockchain`, `accounts`, `mempool`, `network`, `wallet`, and `miner`.
     * `wallet` is initialized with the seed and key provided in the InitInfo object.
   * This object is also saved to `window.nimiq` for use in the console.
     * On official nimiq.com sites, this is equivalent to the `window.$` object.
	 * With the official sushipool wrapper, this is equivalent to the `window.$nimiq` object.
 * `wrapper.wrappedMiner` (`wrapper.miner.wrappedMiner`)
   * This is the officially supported way to access the wrapped SmartPoolMiner object.
   * Most interactions with this object can be done through the MinerWrapper object, but in some cases the direct object may be necessary.

## The Message System
Thinking ahead towards possibly applications I'd like to develop using Nimiq, I've created a system for signing messages with an address.  An example of this system can be seen [here](https://www.drawpad.org/nimiq/) on my website.  This is done by abusing the ExtendedTransaction spec provided in Nimiq and details can be seen below:
 * A message consists of 3 nonces and a data field.
   * The 3 nonces are stored in the transaction's value, fee, and validityStartHeight fields respectively.
     * This means that the first nonce can never be 0, as all transactions (even if never sent) must transfer some value.
   * The nonces are meant to help identify a message, and can be used to require unique messages.
   * The data field is meant to store 8-bit integers, but for the message system stores strings.
     * This string is converted to an array of integers by the Util method.
 * A message is first created (providing a transaction) and then signed (providing a proof).
   * Both the message and the proof should be sent to the recipient of the message for confirmation.
   * Confirmation consists of ensuring that the proof was signed by the sender specified in the message.
     * Nonce values are also checked to be the same before decoding and returning the message data.
     * Transactions that do not pass confirmation return 'null' as the data.
   * For future improvements, verifying that the provided proof is actually the signed message would be useful.
     * Currently I'm using a SignatureProof, but there are different proofs available that may provide the functionality required.
     * Pull requests or reaching out to me with suggestions are both greatly appreciated.

When signing a message, it's confirmed that the signer is the address specified in the message but that is all.  After creating and signing a message, both the message and proof should be serialized using the respective functions in NimiqUtils and sent to the recipient.  Once received, the recipient can unserialize the two objects (using NimiqUtils.unserializeMessage and NimiqUtils.unserializeProof) and then confirm the message to work with the value associated with it.  Possible use cases for such a feature include signing in on any site with a Nimiq wallet, sending messages to a game server that could have only come from you (or someone with access to your private key), and many more.  All necessary functions for using the message system are implemented in the NimiqUtils class and can be seen below:
 * stringToData(s)
   * This function simply takes a string and converts it to an array of 8-bit integers.
   * This function need not be called on the data manually, but could be useful in other cases.
 * dataToString(d)
   * This function does the reverse of stringToData() and also need not be called manually.
 * createMessage(sender, nonceA, nonceB, nonceC, data)
   * This function creates a Nimiq.ExtendedTransaction using the provided data.
   * Both the sender and receiver are set to be normal accounts, with the receiver being the Nimiq Null Address.
     * The sender should be the current wallet controlled by the user, but it's been left as a parameter in case necessary.
   * The three nonces are assigned to the value, fee, and validityStartHeight fields.
     * The only requirement here is that nonceA not be 0 as Nimiq transactions cannot send 0 value.
     * These transactions will never see the actual blockchain (at least as designed) but this requirement must still be met.
   * The provided data should be a string that will be converted to an array of integers representing each character.
 * signMessage(message, signer)
   * This function signs a message with the provided wallet object (signer).
   * The signer must have the same address as the one set in the message.sender field.
   * The Nimiq.SignatureProof provided by Nimiq.Wallet.signTransaction(...) is returned.
 * confirmMessage(message, proof, a, b, c)
   * This function accepts a message, proof, and nonce values and confirms that everything matches.
   * The providied nonce values must match those in the message, and the proof must have been signed by the address specified in message.sender
   * In future updates to this function, I hope to somehow confirm that the data "proof" represents is the provided message.
   * The message.data field is returned if all values match, otherwise the null object is returned.

## Provided Classes
### NimiqWrapper
This class wraps Nimiq as a whole and takes care of initializing the libraries.  To connect to the network and finish up initialization you must call the connect() function.  This function returns a boolean based on whether the wrapper was ready to connect.  This is done to prevent a call to connect() before the Nimiq libraries are fully loaded.  The recommended way to handle this boolean is by using JavaScript's "setInterval" function repeating once per second.  The interval should be cancelled once connect(...) returns 'true'.
### MinerWrapper
This class wraps the Nimiq miner, more specifically the pool miner.  It's set up to mine to the account loaded on the creation of NimiqWrapper, and connects to a hard coded pool.  The decision to hard code the pool was made as most uses of this set of classes, will most likely involve the developer having all users mine to a specific pool.  My pool of choice is porkypool and so that's what is currently hard coded.  Feel free to change it up in your own implementation, and future updates to NimiqWrapper may involve the removal of the hard coded pool.
### NimiqUtils
This horribly named class (as there's already another NimiqUtils class and will surely be countless others) provides some utility functions I found necessary while developing the wrapper.  Some of the more useful ones include:
 * bufferToString(ser)
   * Converts a serialized buffer into a string of hexadecimal values.  Useful for serializing and storing/sharing certain objects within Nimiq.
 * stringToBuffer(str)
   * The reverse of bufferToString, this function will return a Nimiq.SerialBuffer created from the passed in string.  This string is split by spaces, and each part of the string is parsed as a hexadecimal value.
 * serializeEncryptWallet(wall, pass)
   * This function exports the passed in wallet (wall), encrypted with the passed in password, and then converts the returned buffer to a string of hexadecimal values.
   * This leaves you with a more familiar representation of a "private key" although this is only similar in the fact that both let you access a cryptocurrency's account.
 * unserializeEncryptedWallet(seed, key)
   * A wallet can be recovered by using this function where 'seed' is the value returned by serializeEncryptWallet(...), and 'key' is the password used to encrypt the wallet.
 * getNewWallet(pass)
   * Generates a new wallet and encrypts it using serializeEncryptWallet(...) returning that same value.
   * The wallet that was created can be accessed by unserializing and decrypting the returned value.
   * While this is a bit of a hassle, this is done to ensure that the returned value and desired password result in the correct wallet.
   * Anyone wanting to do this without all the extra steps can generate the wallet themselves using "Nimiq.Wallet.generate()".
## Demo information
### Pure JavaScript Demo
This demo is a clone of the one hosted [by the official team](https://demo.nimiq.com) except using the NimiqWrapper files included here to work with the API.  The same wallet on all computers is connected, and it's "Demo Wallet 1" that is used.
### Angular Demo
This demo is custom made and also shows off the messaging system I've created.
### Demo Wallets and Codes
#### Demo Wallet Format
 * Address (friendly)
 * Seed (encrypted serialized seed in hex)
 * Key (the string used to encrypt the seed)
#### Demo Wallet 1
 * NQ47 M5T3 M565 A0G0 UHSP 1H4D 7J2T 5FLP GHGY
 * 01 08 80 CB 6F 9C BA 2A 6A D6 7F 3B BF 1E 2F 7D C0 6F D3 66 9F 1D 19 70 9C 6D 7B 1B 1D 2D FD F4 E5 32 0A 7C F9 24 97 48 76 FF 2D 9F 5B A6 9C D8 5C 46 A9 76 3A 94
 * Key - Password
#### Demo Wallet 2
 * NQ68 G7SF ENKC 1U1V YR15 E808 X05R 92J1 22J6
 * 01 08 3E E7 DA 2C C0 09 68 5E DA 61 4D E1 90 53 00 6D 12 50 55 FD F4 0E FF 34 69 7E B8 36 23 2B 8F 55 AA 66 64 36 5F 61 1E 82 6E 11 3E F7 6D 63 2F 60 81 F4 F7 5A
 * Dracula
#### Demo Message Format
 * Used Parameters
 * Message Key / Code
 * Proof Key / Code
#### Demo Message 1
 * (1, 1, 1, Str)
 * 01 00 03 53 74 72 B5 F8 38 A7 28 FD B2 1D A6 9F A3 A1 56 2A 0D 77 67 87 36 A3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 01 00 00 00 01 00 00 00 00
 * A5 A4 CD C1 1E 37 8B 27 7B D2 35 BF C4 C4 54 BE E4 03 E7 5B 2F 30 17 9A 18 49 0D A6 85 29 C5 AC 00 E0 D1 79 88 77 94 31 9D D6 E9 DC 3B 89 D8 5E 9D 29 9F 17 99 D7 3E A2 25 FC 42 01 D4 9E 9A D6 30 73 FF 11 18 3D 1D 7D 3F FE BA 36 96 40 02 A7 9D 65 E9 AD EE 33 95 8F AA 9A 34 64 67 53 BF D6 07
#### Demo Message 2
 * (1, 1, 1, Strang)
 * 01 00 06 53 74 72 61 6E 67 B5 F8 38 A7 28 FD B2 1D A6 9F A3 A1 56 2A 0D 77 67 87 36 A3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 01 00 00 00 01 00 00 00 00
 * A5 A4 CD C1 1E 37 8B 27 7B D2 35 BF C4 C4 54 BE E4 03 E7 5B 2F 30 17 9A 18 49 0D A6 85 29 C5 AC 00 F9 9F 0F 6D F9 5D 07 E0 BB E3 51 09 0F 62 F4 D8 0A DB 09 DF DC 9F 8C B6 45 03 28 A7 89 DF 36 D2 B1 88 AE 77 A8 BD 1D 8C 39 CF 41 E3 4E 7D AF 20 BC 83 43 A5 2A 08 FF 0D 67 BA B5 8F 0A D0 06 05
#### Demo Message 3
 * (12, 42, 1, This is my data)
 * 01 00 0F 54 68 69 73 20 69 73 20 6D 79 20 64 61 74 61 B5 F8 38 A7 28 FD B2 1D A6 9F A3 A1 56 2A 0D 77 67 87 36 A3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 0C 00 00 00 00 00 00 00 2A 00 00 00 01 00 00 00 00
 * A5 A4 CD C1 1E 37 8B 27 7B D2 35 BF C4 C4 54 BE E4 03 E7 5B 2F 30 17 9A 18 49 0D A6 85 29 C5 AC 00 45 46 4B 47 B2 4D D6 FC 23 78 E9 74 76 27 5A C9 DE 40 DF 35 5C B3 3A 6F 85 0A A9 80 E9 13 AB DE 0A F3 E2 E7 85 A5 4F 5A E3 A0 25 59 22 20 B8 F3 75 68 9D 08 65 53 FC 28 EC 05 88 B5 A6 72 E1 06
## Licensing
NimiqWrapper is licensed under the Apache 2.0 license.  This license was chosen in order to restrict developers as little as possible and anything made using NimiqWrapper has no obligations to release source code, include the Apache 2.0 license, or pay me (the creator of NimiqWrapper) any money.

Any modifications made to either of the 3 classes provided with NimiqWrapper won't require that the modified source be released, however I would appreciate if it would be.  Better yet, please submit pull requests for any modifications made that would benefit the community.

If you'd still like to throw some money my way, I accept donations in the following ways:
 * Send NIM to:
   * NQ89 54KX U06J 5S88 LC2Q V684 EESA KXS2 L984
 * Send ETH or tokens to:
   * 0x844a2Fcbc127980b158a04c054A22545a6f44c50
