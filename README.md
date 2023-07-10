# Axelar cross-chain gateway protocol solidity implementation

## Protocol overview

Axelar is a decentralized interoperability network connecting all blockchains, assets and apps through a universal set of protocols and APIs.
It is built on top off the Cosmos SDK. Users/Applications can use Axelar network to send tokens between any Cosmos and EVM chains. They can also
send arbitrary messages between EVM chains.

Axelar network's decentralized validators confirm events emitted on EVM chains (such as deposit confirmation and message send),
and sign off on commands submitted (by automated services) to the gateway smart contracts (such as minting token, and approving message on the destination).

See [this doc](./DESIGN.md) for more design info.

## Build

We recommend using the current Node.js [LTS version](https://nodejs.org/en/about/releases/) for satisfying the hardhat compiler 

Run in your terminal
```bash
npm ci

npm run build

npm run test
```

## Example flows

See Axelar [examples](https://github.com/axelarnetwork/axelar-examples) for concrete examples.

### Token transfer

1. Setup: A wrapped version of Token `A` is deployed (`AxelarGateway.deployToken()`)
   on each non-native EVM chain as an ERC-20 token (`BurnableMintableCappedERC20.sol`).
2. Given the destination chain and address, Axelar network generates a deposit address (the address where `DepositHandler.sol` is deployed,
   `BurnableMintableCappedERC20.depositAddress()`) on source EVM chain.
3. User sends their token `A` at that address, and the deposit contract locks the token at the gateway (or burns them for wrapped tokens).
4. Axelar network validators confirm the deposit `Transfer` event using their RPC nodes for the source chain (using majority voting).
5. Axelar network prepares a mint command, and validators sign off on it.
6. Signed command is now submitted (via any external relayer) to the gateway contract on destination chain `AxelarGateway.execute()`.
7. Gateway contract authenticates the command, and `mint`'s the specified amount of the wrapped Token `A` to the destination address.

### Token transfer via AxelarDepositService

1. User wants to send wrapped token like WETH from chain A back to the chain B and to be received in native currency like Ether.
2. The un-wrap deposit address is generated by calling `AxelarDepositService.addressForNativeUnwrap()`.
3. The token transfer deposit address for specific transfer is generated by calling `AxelarDepositService.addressForTokenDeposit()` with using the un-wrap address as a destination.
4. User sends the wrapped token to that address on the source chain A.
5. Axelar microservice detects the token transfer to that address and calls `AxelarDepositService.sendTokenDeposit()`.
6. `AxelarDepositService` deploys `DepositReceiver` to that generated address which will call `AxelarGateway.sendToken()`.
7. Axelar network prepares a mint command, and it gets executed on the destination chain gateway.
8. Wrapped token gets minted to the un-wrap address on the destination chain B.
9. Axelar microservice detects the token transfer to the un-wrap address and calls `AxelarDepositService.nativeUnwrap()`.
10. `AxelarDepositService` deploys `DepositReceiver` which will call `IWETH9.withdraw()` and transfer native currency to the recipient address.

### Cross-chain smart contract call

1. Setup:
    1. Destination contract implements the `IAxelarExecutable.sol` interface to receive the message.
    2. If sending a token, source contract needs to call `ERC20.approve()` beforehand to allow the gateway contract
       to transfer the specified `amount` on behalf of the sender/source contract.
2. Smart contract on source chain calls `AxelarGateway.callContractWithToken()` with the destination chain/address, `payload` and token.
3. An external service stores `payload` in a regular database, keyed by the `hash(payload)`, that anyone can query by.
4. Similar to above, Axelar validators confirm the `ContractCallWithToken` event.
5. Axelar network prepares an `AxelarGateway.approveContractCallWithMint()` command, signed by the validators.
6. This is submitted to the gateway contract on the destination chain,
   which records the approval of the `payload hash` and emits the event `ContractCallApprovedWithMint`.
7. Any external relayer service listens to this event on the gateway contract, and calls the `IAxelarExecutable.executeWithToken()`
   on the destination contract, with the `payload` and other data as params.
8. `executeWithToken` of the destination contract verifies that the contract call was indeed approved by calling `AxelarGateway.validateContractCallAndMint()`
   on the gateway contract.
9. As part of this, the gateway contract records that the destination address has validated the approval, to not allow a replay.
10. The destination contract uses the `payload` for it's own application.

## References

Network resources: https://docs.axelar.dev/resources

Deployed contracts: https://docs.axelar.dev/resources/mainnet

General Message Passing Usage: https://docs.axelar.dev/dev/gmp

Example cross-chain token swap app: https://app.squidrouter.com

EVM module of the Axelar network that prepares commands for the gateway: https://github.com/axelarnetwork/axelar-core/blob/main/x/evm/keeper/msg_server.go
