# Delegating Transactions

Most `Registry` and `Resolver` methods have meta-transaction support, which allows you to delegate transactions to another party. Generally, meta-transactions allow users to sign messages to control their domains that are then submitted to the registry by a different party. This enables Unstoppable to submit transactions on behalf of users so that the user can still manage their domains in a self-custodial capacity without any gas.

Meta-transactions work by having users sign function calls along with a nonce. They then send that signed function call over to a different party. That party calls the meta-transaction-enabled function on the `Registry` or `Resolver`. For most management methods, there is a method with meta-transaction support that has a `For` suffix at the end. The meta-transaction method then checks the permission for a domain against the address recovered from the signed message sent to the function, unlike the base method that checks it against the submitter of the transaction e.g. `msg.sender`. Test!

![](../.gitbook/assets/Meta-Transaction.svg)

For example, `resetFor` is the meta-transaction version of `reset`. This method has an additional `signature` argument as the last parameter.

Note that the meta-transaction versions of `Registry` functions are implemented in the [SignatureController.sol](https://github.com/unstoppabledomains/dot-crypto/blob/master/contracts/controllers/SignatureController.sol) contract, not in the registry itself.

The source code for signature validation can be found in [SignatureUtil.sol](https://github.com/unstoppabledomains/dot-crypto/blob/master/contracts/util/SignatureUtil.sol)

## Token nonce

Meta transaction methods are bound to names via their nonce \(instead of [Account nonce](https://ethereum.stackexchange.com/questions/27432/what-is-nonce-in-ethereum-how-does-it-prevent-double-spending) of traditional transactions\). It protects from [Double-spending](https://en.wikipedia.org/wiki/Double-spending) in the same way as an account-based nonce in traditional transactions.

The example below shows how replay attacks can be used to exploit domains:

![](../.gitbook/assets/Without-Nonces.svg)

A nonce is simply a transaction counter for each token. This prevents replay attacks where a transfer of a token from `A` to `B` can be replayed by `B` over and over to continually revert the state of the name back to a previous state. This counter increments by 1 each time a state transition happens to a token. Token-based nonces can be used to prevent misordering of transactions in a more general sense as well. This prevents front running non-fungible assets and enables secure transaction batching.

![](../.gitbook/assets/Nonces.svg)

## Meta transaction signature generation

A meta transaction requires 2 signatures: one passed as a method argument and one classical. A classical signature is generated in a standard way. A meta signature requires a domain owner \(or a person approved by the owner\) to sign a special message formed from:

* A domain based meta-transaction nonce
* A [Function selector](https://solidity.readthedocs.io/en/v0.7.0/abi-spec.html#function-selector) of the original method
* The original method parameters \(the one without signature\)

Example signature generation for a `reset` method call for a domain:

```javascript
const domain = 'example.crypto';
const methodName = 'reset';
const methodParams = ['uint256'];
const contractAddress = '0xb66DcE2DA6afAAa98F2013446dBCB0f4B0ab2842';
// Can be different or the same as contractAddress
const controllerContractAddress = '0xb66DcE2DA6afAAa98F2013446dBCB0f4B0ab2842';
const tokenId = namehash(domain);

function generateMessageToSign(
  contractAddress: string,
  signatureContract: string,
  methodName: string,
  methodParams: string[],
  tokenId: string,
  params: any[],
) {
  return solidityKeccak256(
    ['bytes32', 'address', 'uint256'],
    [
      solidityKeccak256(
        ['bytes'],
        [encodeContractInterface(contractAddress, method, methodParams, params)],
      ),
      controllerContractAddress,
      ethCallRpc(controllerContractAddress, 'nonceOf', tokenId),
    ],
  );
}

const message = generateMessageToSign(
  contractAddress,
  signatureContractAddress,
  methodName,
  methodParams,
  tokenId,
  [tokenId]
);
```

Functions Reference:

* `namehash` — [Namehashing gunction](../domain-registry-essentials/namehashing.md) algorithm implementation
* `ethCallRpc` — Ethereum `eth_call` JSON RPC implementation
* `encodeContractInterface` — [Solidity ABI](https://solidity.readthedocs.io/en/v0.7.0/abi-spec.html#argument-encoding) interface parameters encoder
* `solidityKeccak256` — [Solidity ABI](https://solidity.readthedocs.io/en/v0.7.0/abi-spec.html#argument-encoding) parameters encoder

