# Sifchain token importing and exporting

- sifethusdt -> bnbethusdt / USDT on Ethereum -> USDT on Sifchain
  - Scenario: Freya burns coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6 (holder for USDT) on sifchain
    and ends up with double-pegged USDT on Binance
    - Actions
      - On Sifchain
        - Freya calls the sifnodecli lock command, specifying
          - coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6 as the denom
          - 1000000 as the amount
          - A fee in rowan (the amount is different for locks and burns)
          - Gas in Rowan
          - the destination network descriptor
        - lock checks to see if the token is on the list of available tokens for the network descriptor.
          - If not, the lock fails
          - See other scenario for how to create the corresponding token the first time
        - The fee is transfered to the receiver account (set at runtime by ```sifnodecli tx ethbridge update_ceth_receiver_account```)
        - Relayers call the smart contracts with the lock data
        - Once the threshold for consensus has been met on Ethereum, send 1000000 of the 
  - Scenario: Freya wants to burn coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6 (a token from 
    Ethereum mainnet, network descriptor 1) to Binance Smart Chain (network descriptor 2) and this is the first
    time anyone has burned that token to BSC.  Create that holding token.
      - This requires deploying a new ERC20 token to BSC, and that operation is expensive.  Instead of
        rolling that operation into the burn itself, we split it off.  Burns that don't have a matching token
        already created will fail.
    - Actions
      - Freya runs the sifnodecli command to create a destination ERC20 token for
        coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6
        - These fields are set to the values held on sifchain for coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6
          - name
          - symbol
          - decimals
        - Note that no decisions need to be made about normalizing these fields; 
          the exact values stored on sifchain will be used
        - Since ERC20 tokens are free to change the values of any of those fields, we will
          use the value from the very first 
      - Relayers call the smart contract action to deploy an ERC20 token for
        coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6
          - Need to reach the consensus threshold before this executes
          - Sifchain needs to store the ERC20 data (name, symbol) for all sifchain denoms
              - if it's not stored, every relayer would need to be able to connect to every supported EVM chain
      - when the token is deployed, tokens should be minted and sent to BridgeBank
- ethusdt -> sifusdt / USDT on Ethereum -> USDT on Sifchain
  - Scenario: Freya locks USDT on Ethereum and ends up with USDT on Sifchain
    - Ethereum token data for USDT on Ethereum 
      - address: 0xdac17f958d2ee523a2206206994597c13d831ec7
      - name: 'Tether USD'
      - symbol: 'USDT'
  - Actions
    - On Ethereum
      - Freya calls #approve on the USDT token contract with the address of the BridgeBank as a parameter
      - Freya calls #lock on the BridgeBank, specifying 
        - sif1pvnu2kh826vn8r0ttlgt82hsmfknvcnf7qmpvk, a sifchain account address to receive tokens
        - 0xdac17f958d2ee523a2206206994597c13d831ec7, the address of USDT
        - 1000000, one USDT (decimals() returns 6 for USDT)
      - BridgeBank does the following:
        - Call decimals() on the contract at 0xdac17f958d2ee523a2206206994597c13d831ec7
          - if decimals() returns an integer in the range 0 to 18 inclusive, use it
          - if decimals() has a exception, use 18
          - if decimals() is outside the range 0 to 18 inclusive, revert the transaction
            - Sifchain is unable to use these tokens
        - Transfers 1000000 from Freya's account to itself
      - The BridgeBank contract emits a log message with the following fields:
        - 1: the Sifchain network descriptor for Ethereum mainnet
          - network descriptors are assigned by Sifchain
        - sif1pvnu2kh826vn8r0ttlgt82hsmfknvcnf7qmpvk, a sifchain account address to receive tokens
        - 0xdac17f958d2ee523a2206206994597c13d831ec7, the address of USDT
        - 1000000, one USDT (decimals() returns 6 for USDT)
      - Notes
        - BridgeBank does not send the symbol or the name of the token at 0xdac17f958d2ee523a2206206994597c13d831ec7.
          Information about #name and #symbol is handled elsewhere.
    - In the Sifchain relayers
      - ebrelayer chooses the most recent unprocessed block to process
        - Note that ebrelayer delays processing blocks to ensure that a block will be a permanent part of the Etherum consensus
      - For each log message, ebrelayer does:
        - Computes the sifchain denom
          - coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6
            - where 1 is the network descriptor and 6 is decimals
        - Send sif1pvnu2kh826vn8r0ttlgt82hsmfknvcnf7qmpvk 1000000 of
          coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6, and include the ethereum transaction hash in the sending message
            - including the ethereum hash ensures we can make sure we never process a transaction twice
        - Once all log messages in a block have been processed, record in sifchain that the ethereum block has
          been processed
            - reduces startup time for ebrelayer, since it can just start processing ethereum blocks at the most recent block number
    - End state
      - ```sifnodecli q auth account sif1pvnu2kh826vn8r0ttlgt82hsmfknvcnf7qmpvk``` returns
        ```{"coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6": 1000000}```
      - Freya has 1000000 fewer USDT
      - ebrelayer records on sifchain:
        - the latest ethereum block number that has all transactions processed
        - the ethereum transaction and block number for each transfer
- ethusdt -> sifethusdt / USDT on Ethereum to USDT on Sifchain
  - Scenario: Freya wants to export USDT from Ethereum to Sifchain
    - Actions
      - On Ethereum
        - Freya authorizes BridgeBank to spend USDT
        - Freya sends a lock command to BridgeBank with the following:
          - amount
          - 0xdac17f958d2ee523a2206206994597c13d831ec7 - the address of USDT
          - A sifchain destination address
        - BridgeBank recevies the tokens and emits an event with those same parameters
          - Note that BridgeBank doesn't need to specify its own network descriptor; that's
            handled by the relayers
      - Relayers
        - Send the following to sifchain:
          - 
- sifethusdt -> ethusdt / USDT on Sifchain to USDT on Ethereum
  - Scenario: Freya wants to export coin/1/0xdac17f958d2ee523a2206206994597c13d831ec7/6 (a token from
    Ethereum mainnet, network descriptor 1) to Ethereum.
    - Look up 


- Rowan on Ethereum -> Rowan on Sifchain
- Rowan on Sifchain -> Rowan on Ethereum
1. Rowan exported to Ethereum
- A user sends a lock tx with the sifnodecli specifying rowan as the token to lock up, the amount of tokens to send, and the desired ethereum address recipient. This TX emits a cosmos event with all of these data fields.
- A relayer has subscribed to these cosmos events and hears them. Upon hearing the event, the relayer takes all of those data fields of eth address, token amount and token type, and packages it into an ethereum transaction. This transaction gets submitted to the CosmosBridge contract by calling ```newProphecyClaim```. Other relayers do the same.
- Once enough relayers have signed off on the prophecyClaim in the CosmosBridge, the CosmosBridge calls the BridgeBank to mint new tokens for the intended recipient in the BridgeToken contract.


2. Ethereum Native Pegged Asset being transferred to ethereum.
- A user sends a burn tx with the sifnodecli specifying ceth as the token to burn, the amount of tokens to send, and the desired ethereum address recipient. This TX emits a cosmos event with all of these data fields.
- A relayer has subscribed to these cosmos events and hears them. Upon hearing the event, the relayer takes all of those data fields of eth address, token amount and token type, and packages it into an ethereum transaction. This transaction gets submitted to the CosmosBridge contract by calling ```newProphecyClaim```. Other relayers do the same.
- Once enough relayers have signed off on the prophecyClaim in the CosmosBridge, the CosmosBridge calls the BridgeBank to unlock the eth for the intended recipient in the BridgeToken contract.

Both scenarios 1 and 2 can be summarized with this image.
![image info](images/peggy-flow.png)



Note that the next type of transactions require approving the BridgeBank contract to spend your tokens before calling either the lock or burn function. The only exception to this rule is ethereum, you do not need to call approve on ethereum, because it is not an ERC20 token.


3. Ethereum Native Asset on Ethereum Being Transferred to Sifchain
- A user sends a lock tx on ethereum to the BridgeBank contract specifying the token address, the amount of tokens to send, and the desired address of the sifchain recipient. This TX emits an ethereum event with all of these data fields.

```
event LogLock(
    address _from,
    bytes _to,
    address _token,
    string _symbol,
    uint256 _value,
    uint256 _nonce
);
```
- Relayers have subscribed to the BridgeBank smart contract, hear this transaction and package it up into a new Oracle Claim and submit the transaction to cosmos.
- Once enough relayers have signed off on this Oracle Claim on cosmos, then pegged assets are minted on cosmos to the recipient.


4. Cosmos Native Pegged Asset on Ethereum Being Transferred Back to Sifchain
- A user sends a burn tx on ethereum to the BridgeBank contract specifying the token address, the amount of tokens to send, and the desired address of the sifchain recipient. This TX emits an ethereum event with all of these data fields.
```
event LogBurn(
    address _from,
    bytes _to,
    address _token,
    string _symbol,
    uint256 _value,
    uint256 _nonce
);
```
- Relayers have subscribed to the BridgeBank smart contract, hear this transaction and package it up into a new Oracle Claim and submit the transaction to cosmos.
- Once enough relayers have signed off on this Oracle Claim on cosmos, then pegged assets are unlocked on cosmos and sent to the recipient.


Definitions:
- LokiBux
  - an ERC20 that does every random thing it can and still be an ERC20 token.  Calls to decimals() for example
    return a random number on every call.