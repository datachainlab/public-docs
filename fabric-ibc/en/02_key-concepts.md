# Key Concepts

## IBC Compliance

Fabric-IBC supports [IBC](https://github.com/cosmos/ibc), specifications for inter-blockchain communication that is being standardized.
This will enable communication between blockchains, DLTs, and legacy systems that implement the IBC specifications.

## Automated communication

In Fabric-IBC, a packet sent from one blockchain is automatically relayed to the target blockchain through the Relayer service.
In this way, clients organizing a process across multiple blockchains do not need to perform any operations on the target blockchain, and can transparently communicate with other blockchains.

## Without Trusted third-party

<!-- A trusted third party (TTP) is a trusted third party that is required in addition to the trust of each Blockchain that performs IBC. -->
Currently, interoperability solutions often require a Trusted third-party(TTP), which acts as the coordinator of the states of the blockchains involved.
However, in general, it is difficult to form an inter-blockchain network based on TTP.

On the other hand, Fabric-IBC does not assume the existence of a TTP. The operator of the Relayer service in Fabric-IBC, can be anyone who has access to the network. Fabric-IBC enables inter-blockchain communication without any trusted entity other than the consensus execution of each blockchain.

## Ease of integration with existing chaincodes

To develop Fabric application using IBC, developers need to add Module which implements application logic to App provided by Fabric-IBC.

Developers can easily migrate business logic and data models from existing Chaincodes. Fabric-IBC provides a wrapper for the Chaincode shim API for this purpose.

## Extendable Architecture

The core of Fabric-IBC focuses on the ability to send and receive Packets, the means of communication with other blockchains.
Developers can develop their own modules. For example, they can add functions that enhance the security and privacy of Packet's data.
We have already developed several modules, such as Private module, Cross Framework, etc.
