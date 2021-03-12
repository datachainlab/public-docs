# Appendix

## Regarding the Privacy of Relayer

A Relayer in Cosmos is initialized with a [configuration](https://github.com/datachainlab/cross/blob/adbf051333acb1f8fbcbae6ff2888d477a4bfeb9/tests/demo/path01.json#L1) and connection information for both blockchains. The Relayer then monitors both chains and submits a packet transaction to the other chain. For this reason, it is assumed that the Relayer has the following privileges.

<!-- The Relayer in Cosmos is configured with [this configuration](https://github.com/datachainlab/cross/blob/adbf051333acb1f8fbcbae6ff2888d477a4bfeb9/tests/demo/path01. json#L1) and the connection information of both Blockchains, a Relayer monitors both chains and submits Packet transactions to the other chain. Therefore, it is assumed that the Relayer has the following privileges: 1.-->

1. the relayer can query an event to the sender blockchain.
2. the relayer can get the state of both blockchains.
3. the relayer can submit a transaction to both blockchains.

However, as HLF is a permissioned blockchain, it is generally difficult for a Relayer to satisfy all of the above. In particular, in order to satisfy 1. and 2., it is necessary to assume the existence of a Relayer operator who can access the two different ledgers, which raises privacy concerns.

Therefore, in Fabric-IBC, it is recommended that the Relayer is operated by (one or more) organizations that can read and write to the Chaincodes of both chains.

If the roles 1 and 2 are performed by a specific role of the organizations operating the chaincode, privacy issues can be avoided. Alternatively, an API gateway can be applied.
As for 3., it is possible to avoid disclosing unnecessary states while granting write permission to state by allowing only functions that provide specific IBC functions to the Relayer.
