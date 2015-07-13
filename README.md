Muttr: Secure Distributed Messaging
===================================

Abstract
--------

The problem...

Mechanics
---------

The Muttr network is comprised of two distinct types of participants: *seeds* 
and *pods*. Seeds are nodes which store PGP encrypted messages on behalf other 
seeds in the network, forming a DHT (distributed hash table). Each seed 
possesses a PGP key pair which is used to sign messages to other seeds and 
decrypt messages in the DHT that are intended for them. 

Pods also participate in the storage of messages but, more importantly, form a 
"meta-network" that acts as a federated routing system. Each pod allows seeds 
to register *aliases* to their public PGP key, which allows other seeds to 
address them by `<some_alias>@<pod_hostname>`. In addition, seeds may use pods 
to help deliver messages to other seeds by providing the location (or key) of a 
given message in the DHT, which the pod will relay to the recipient directly or 
if the recipient is offline, store for "playback" at a later time.

In practice, to join the network a seed will connect to a pod of it's choice on 
two channels. The first channel is to start discovering other seeds in the DHT 
and begin participating in the cooperative storage of messages. The other is a 
private channel between the seed and the pod exclusively, for the purpose of 
notifying the seed of received message keys that the seed can then go look up 
and decrypt.

The typical flow can be described below.

1. Unidentified **seed** connects to **pod** muttr.me and asks for nearest seeds
2. **Pod** responds with nearest seeds and **seed** builds routing table
3. **Seed** identifies itself by sending signed message to **pod** privately
4. **Seed** registers an alias *Alice* with **pod**
5. **Pod** begins pushing message keys addressed to *Alice* over private channel
6. *Alice* asks **pod** for the public key for *Bob* and **pod** replies
7. *Alice* signs and encrypts a message to *Bob*
8. *Alice* stores the message at the appropriate seeds and it propagates the DHT
9. *Alice* informs **pod** that a message for *Bob* is stored at the given key
10. *Bob* gets notified by *pod* of the message
11. *Bob* fetches the message from the DHT
12. *Bob* decrypts and verifies the message from *Alice*

Protocol
--------

The protocol...

Implementation
--------------

Code samples and reference implementation...

Comments
--------

Describe improvement proposal process...

---

This work is licensed under the Creative Commons Attribution-ShareAlike 4.0 
International License. To view a copy of this license, visit 
http://creativecommons.org/licenses/by-sa/4.0/.
