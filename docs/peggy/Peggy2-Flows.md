# Sifchain token importing and exporting

- Rowan on Ethereum -> Rowan on Sifchain
- Rowan on Sifchain -> Rowan on Ethereum
- USDT on Ethereum -> USDT on Sifchain
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
      - The BridgeBank contract emits a log message with the following fields:
        - 1: the Sifchain network descriptor for Ethereum mainnet
          - network descriptors are assigned by Sifchain
        - sif1pvnu2kh826vn8r0ttlgt82hsmfknvcnf7qmpvk, a sifchain account address to receive tokens
        - 0xdac17f958d2ee523a2206206994597c13d831ec7, the address of USDT
        - 1000000, one USDT (decimals() returns 6 for USDT)
    - In the Sifchain relayers
      - ebrelayer chooses the most recent unprocessed block to process
        - Note that ebrelayer delays processing blocks to ensure that a block will be a permanent part of the Etherum consensus
      - 
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
