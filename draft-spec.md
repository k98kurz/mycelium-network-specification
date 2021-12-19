# Mycelium: A Decentralized Network Layer With Emergent Topology

## Abstract

This document introduces a network protocol which combines novel machine
learning and cryptographic primitives to measure the topology of internetworked
data links and route packets through them. Hosts measure latencies over links to
generate descriptors and addresses that encode a model of the network topology.
Peers also maintain the internetwork by evaluating models. Specifically, this
document specifies the method by which autonomous networks gain their domain
identifiers; the address format; the mechanism used by each node to determine
its address; the mechanism used by each peer to validate each address
self-assignment it encounters; the mechanism by which changes in routing or
policy propagate through the network; route table construction; a system for
bridging between networks; a tunneling system; a rudimentary routing and route
discovery system; a set of security models used for spam protection, privacy,
peer trust scores, peer reliability tracking, bandwidth allocation and
reciprocity, and Sybil attack resistance. As an additional benefit of the
protocol's use of Ed25519 digital signatures, each autonomous network will have
a built-in public key directory, and these directories combine to form a
distributed hash table. Peers discover each other through gossip about the DHT
and use ephemeral ECDHE to provide forward secrecy. The exact structure of
packets is beyond the scope of this document, but some general guidelines are
proposed. Finally, potential transport, session, and application layer protocols
as well as upgrades to Mycelium version and network configuration are explored.

## Table of Contents

1. Introduction
2. Terminology
3. Autonomous Networks
    - Topologies and Data Links
    - Configuration
    - Domain Identifier
4. Address Format
5. Address Assignment/Allocation
    - Bootstrapping a New Network
    - Measurements of Latency
    - Leasing of Guest Addresses from Peers
    - Calculation and Assignment of Address
    - Verification of Address Self-Assignment by Peers
    - PKI/Directory Service
6. Gossip
    - Network Configuration
    - Address Assignments
    - Peer Policy Change
    - Peer Disconnect
7. Peer Services
    - Peer Policies
    - Network Configuration
    - Guest Address
    - Cryptographic/Security Services
    - Routes and Directory
8. Bridges, Tunnels, and Topology Testing
    - Bridges Between Networks
    - Tunnels Through Routers
    - Topology Testing
9. Routing and Route Discovery
10. Route Table Construction
    - Local Networks
    - Bridges to Other Networks
    - Tunnels
11. Network Layer Security
    - Spam Protection
    - Private Transmissions
    - Peer Trust Scores
    - Peer Reliability Tracking
    - Neighbor-to-Neighbor Bandwidth Allocation and Reciprocity
    - Sybil Attack Resistance
12. Implications and Uses of Mycelium for Transport, Session, and Application
Layers
13. Upgrades/Downgrades
    - Mycelium Version
    - Network Configuration
14. References


## 1. Introduction

Previous network layer protocols relied upon a centralized hierarchy for address
assignment. Such a model requires human intervention in organizing and
maintaining networks, and human institutions are relied upon for accurately
measuring network topology and using that topology to make address assignment
and routing decisions. For example, IPv4 and IPv6 use a hierarchy of human
instititutions stemming from IANA for address assignment. Mycelium is the first
network layer protocol that automatically assigns addresses which encode network
topology as measured by hosts, and it includes a network layer security model.
The inspiration for this network system is mushroom mycelia, which are able to
grow autonomous networks from broken fragments and recombine those autonomous
networks into larger mycelial mats, forming symbioses with tree roots, with an
emergent hierarchy and without any central coordination.


## 2. Terminology

- node: a device that implements Mycelium.
- peers: nodes that share a network configuration.
- host: a node that has its own address within an autonomous network.
- guest: a node that has leased an address from a host.
- router: a host that forwards Mycelium packets not addressed to itself.
- bridge: a router that has more than one active network configuration and can
route packets between the autonomous networks.
- upper layer: a protocol layer immediately above Mycelium.
- link: a communication facility or medium over which nodes can communicate at
the link layer, i.e., the layer immediately below Mycelium. Examples are
Ethernets (simple or bridged); 802.11 radios; Long Range (LoRa) radios; PPP; and
internet-layer or higher-layer "tunnels", such as tunnels over IPv4, IPv6, or
Mycelium itself.
- tunnel: a route composed of guest addresses leased to a peer allowing it to
route traffic through the hosts as if it was attached to a link shared with that
host; all hops in a tunnel are encrypted in layers for privacy, and each hop is
delayed up to a maximum allowed in the headers, so traffic is indistinguishable
from ordinary traffic, and the endpoints of a tunnel are hidden from observers.
- neighbors: nodes which can communicate with each other directly over a link.
- autonomous network: a collection of peers that share a network topology, route
packets for each other, and provide peer services to each other. Each autonomous
network has its own identifier used in directory entires.
- internetwork: a group of autonomous networks joined by bridges which share a
distributed directory.
- coordinates: a tuple of floats approximating the location of a node within a
network topology.
- interface: a node's attachment to a link.
- address: a Mycelium-layer identifier encoding the coordinates for an interface
or set of interfaces.
- topology: a model describing how peers are arranged, connected, and addressed
in an autonomous network.
- distance: latency of communication between neighbors over a link.
- distance metric: a mathematical construction for calculating distance between
two points in an N-dimensional network topology.
- squared error: the squared difference between a predicted value and an
observed value.
- mean squared error (MSE): the mean of the squared errors for a set of
predictions matched to observations.
- gradient descent: a simple machine learning technique that allows a computer
to find optimal estimations; in Mycelium, gradient descent minimizing the mean
squared error of a distance metric is used to map network topology by estimating
coordinates given neighbor coordinates and the measured distances to those
neighbors.
- gradient: the scalar sum of the partial derivatives of a function that
calculates the mean squared error given estimated coordinates and vectors
containing known coordinates and distances.
- domain identifier (DID): the Merkle root of the network configuration.
- peer identifier (PID): an identifier created from a DID, an address from that
network, and the public key of the peer with that address.
- content identifier (CID): a hash or Merkle root of some data.
- Hamming distance: the number of different bits between two blocks of data,
i.e. the number of 1s in the output of XORing two equal-length bit strings.
- distributed hash table (DHT): a distributed data structure in which full
replication is not necessary and likelihood for a peer to maintain a copy of a
block of data is proportional to the Hamming distance between its CID and the
peer identifier.
- directory: a mapping of URIs that allows peers to find resources on the
network; in Mycelium, the primary directory is a list mapping public keys to
addresses and networks stored in DHT across all networks.
- gossip: an efficient, asynchronous protocol for omnidirectional communications
between peers.
- peer service: a network service offered to peers by hosts.
- Hashcash: a simple mechanism invented by Adam Beck for spam protection; the
process involves incrementing a nonce until the first bytes of the hash of the
message contain a specified number of preceding null bits.
- proof-of-work (PoW) token: a commitment to some values which meets a Hashcash
difficulty threshold.
- service token: a cryptographic token that can be used to pay for network
services; each token must be signed by a token issuer trusted by a peer for that
peer to accept the token in payment of network services. Each use of a service
token decreases the remaining value and may or may not transfer the difference,
and token value may additionally decay over time. The default specification
service token will not be transferrable.
- reusable proof-of-work (RPoW): an adaptation of Hashcash that allows for a
proof-of-work token to be used as a service token.
- elliptic curve Diffie-Hellman Exchange (ECDHE): a cryptographic mechanism
through which two peers can establish a shared secret for encryption without
exposing any information that would allow an observer or man-in-the-middle to
compute the shared secret.
- ephemeral ECDHE: ECDHE where the sending party uses a new, randomly generated
key pair.
- Curve25519: an ECDHE scheme using the Twisted Edwards Curve 25519.
- Ed25519: a digital signature scheme using a Schnorr group derived from the
Twisted Edwards Curve 25519. Keys and signatures can be aggregated without
increasing the size of the public key or signature. A Curve25519 private key can
be generated from an Ed25519 private key, and a Curve25519 public key can be
generated from an Ed25519, providing for a compact PKI where one public key can
be used for both verifying signatures and deriving shared secrets.
- MuSig/MuSig2: a Schnorr aggregate signature scheme compatible with Ed25519.
- Sybil attack: an attack in which a bad actor pretends to be many nodes to
manipulate a network, unfairly distort network activity, or steal a network
service.

Note: it is expected that each host act as a router, but any router may
decide to forward packets only on a subset of its interfaces. A host that leases
an address to a guest is explicitly responsible for forwarding that guest's
packets, but any host may decline to lease addresses to guests.


## 3. Autonomous Networks

An autonomous network is a collection of peers that share a network topology,
route packets for each other, and provide peer services to each other. Each
autonomous network has its own identifier used in directory entires, and a group
of autonomous networks which are bridged together form an internetwork through
which packets are routed and in which a distributed directory is compiled.

On first boot, each node will generate a set of configurations for boostrapping
autonomous networks with it as the origin. It will then attempt to find
neighbors to determine if compatible autonomous networks already exist for its
links, in which case it will attempt to join those networks; otherwise, it will
wait for neighbors to appear on its links and attempt to bootstrap autonomous
networks with them.

### 3.1 Topologies and Data Links

An autonomous network can span multiple data links so long as all nodes use the
same topology model to derive their addresses. However, not every model is
appropriate for every data link, so each node will share a variety of standard
configurations with its neighbors to boostrap a group of bridged networks for
its links. See section 8.3 for details on bridging and topology testing.

The topologies available in Mycelium will be the following:
- Mesh: a mesh of peers sharing a radio link or variety of radio links described
by a set of unit vectors and a distance metric.
- Bus: a group of peers sharing a data bus link; effectively a 1-dimensional
mesh network.
- Ring: a group of peers sharing links configured in a ring topology.

### 3.2 Configuration

An autonomous network is configured with a Merklized data structure specifying
the following parameters:

- Tree 1:
    - Mycelium version: the hash of the specification or reference
    implementation of the Mycelium version.
    - Network topology type: the expected network topology to model, i.e. mesh,
    bus, ring, or star as described in this document, or a topology described in
    a later version.
    - Network dimensions: the number and data format of coordinates used to
    describe a node's location in a mesh network toplogy.
    - Distance metric: which distance metric to use for gradient descent, i.e.
    L1 norm (taxicab geometry), L2 norm (Euclidean), L3 norm, L-infinity norm
    (Chebyshev), arbitrary Minkowski (tuned p parameter), or some other metric
    introduced in a later version.

- Tree 2:
    - Gradient threshold: the maximum gradient value for a set of coordinates
    given vectors of neighbor coordinates and distance measurements to those
    neighbors.
    - Error threshold: the maximum mean squared error value for a set of
    coordinates given vectors of neighbor coordinates and distance measurements
    to those neighbors.

- Tree 3:
    - Hashcash threshold: the minimum hashcash proof-of-work difficulty for a
    message to be honored by peers.
    - Some Sybil param
    - Some security param
    - Something else

- Tree 4:
    - Network origin public key: the public key of the network origin; can be
    the public key of an individual node or the aggregate key for several nodes.
    - Network admin key(s): the public key of the network admin(s) used for
    updating the network configuration.


## 4. Address Format

@todo


## 5. Address Assignment/Allocation

@todo

### 5.1 Bootstrapping a New Network

@todo

### 5.2 Measurements of Latency

@todo

### 5.3 Leasing of Guest Addresses from Peers

@todo

### 5.4 Calculation and Assignment of Address

@todo

### 5.5 Verification of Address Claim by Peers

@todo

### 5.6 PKI/Diretory Service

@todo


## 6. Network Gossip

Peers will gossip with each other, exchanging messages for mutually-subscribed
topics. Each node subscribes to the gossip for networks in which that node has
an address, whether as a peer or as a guest through a tunnel. This network
gossip will contain any information necessary for the peers in a network to
continue routing packets and interacting with each other. During the gossip
process, two peers exchange lists of tuples of topic identifiers (often a DID)
and CID of the last message seen by the peer. Each peer will maintain a simple
state mapping TIDs to CIDs for global gossip and individual gossip channels. Any
time that a new message or list of messages is received from a peer, the gossip
channel state is updated by pushing the CIDs onto a stack above the previous
CIDs maintaining the order in which the messages were received, which triggers
an event to update the global gossip state; when the global gossip state
changes, it triggers events for all the gossip channels.

Each piece of gossip will include metadata such as message type, timestamp, and
any tokens burned as an anti-spam mechanism. For each message type, each peer
will be configured to assess whether or not the message expired and will ignore
any expired messages.

### 6.1 Network Configuration

@todo

### 6.2 Address Assignments

@todo

### 6.3 Peer Policy Change

@todo

### 6.4 Peer Disconnect

@todo


## 7. Peer Services

@todo

### 7.1 Peer Policies

@todo

### 7.2 Network Configuration

@todo

### 7.3 Guest Address

@todo

### 7.4 Cryptographic/Security Serices

@todo

### 7.5 Routes and Directory

@todo


## 8. Bridges and Tunnels

@todo

### 8.1 Bridges Between Networks

@todo

### 8.2 Tunnels

For enhanced privacy applications and network bridging, routers may enable a
tunneling service. Each tunnel will take the form of a guest address assigned to
the requesting peer by the router and will be identified with an ephemeral
public key used as the guest public key. The guest will pay any costs for the
tunnel speified by router policy

Each packet traversing a tunnel will be encrypted in layers similarly to onion
routing, with each layer containing

### 8.3 Topology Testing

When a node has multiple network configurations for the same data link or subset
of non-neighbor peers that can be reached through non-local bridge(s), it will
test the accuracy of the topologies predicted by each network configuration and
disconnect from the network with the highest mean squared error. This will be
achieved through an audit process that will predict latencies using routing data
and heuristics, measure the actual latencies, and calculate a mean squared error
for each network audited.

Similarly, when a subset of non-neighbor peers as identified by their public
keys is found in the route table to be an intersection of networks, the node
will predict and measure the latencies to each such peer for each network,
then drop the route that had the larger mean squared error.


## 9. Routing and Route Discovery

@todo


## 10. Route Table Construction

@todo

### 10.1 Autonomous Networks

@todo

### 10.2 Bridges to Other Networks

@todo

### 10.3 Tunnels

@todo


## 11. Network Layer Security

@todo

### 11.1 Spam Protection

@todo

### 11.2 Private Transmissions

@todo

### 11.3 Peer Trust Scores

@todo

### 11.4 Peer Reliability Tracking

@todo

### 11.5 Neighbor-to-Neighbor Bandwidth Allocation and Reciprocity

@todo

### 11.6 Sybil Attack Resistance

@todo


## 12. Implications and Uses of Mycelium for Transport, Session, and Application Layers

@todo

Another potential protocol that would be useful for building applications is a
hybrid between gossip, Dandelion, and onion packaging: a message is created,
encrypted with a peer's public key and then encrypted with a second peer's
public key, then the layer-encrypted message is added to the gossip queue; when
the peer with the public key for the outermost layer encounters the message, it
decrypts it and adds the result to its gossip pool; when the mid layer message
is encountered by the second peer, it decrypts it and adds the result to the
gossip queue; the original message at the heart of the onion then propagates
through the network's gossip channels. An alternative to using public keys is to
use a symmetric cipher and include a bit string and its Hamming distance from
the key; when a peer encounters the encrypted message, it flips the Hamming
distance number of bits at random and attempts to decrypt the message, using the
HMAC to verify; upon success, the decrypted message is gossiped out; on failure,
the encrypted message is gossiped out. The former system is traceable through
traffic analysis if the choice of peers is poor, while the latter system is
inherently subject to a very simple brute force attack and non-deterministic
behavior. Using staggered layers of both should make traffic analysis
infeasible.


## 13. Upgrades

Upgrades will be applied with a soft-fork style, maximally preserving backwards
compatibility and the ability to downgrade. Upgrades to a network configuration
can be applied automatically if an admin key is specified in the initial
network configuration.

### 13.1 Mycelium Version

A Mycelium version upgrade would require new software and possibly new hardware
to be installed on a node. A node may include the ability to configure an admin
public key to enable automatic software upgrades through downloading a package
with the new version software signed with the admin public key; otherwise, such
upgrade packages would have to be applied through physically accessing the node.
Unless a security vulnerability is found in an older protocol version, an update
should not deprecate any functionality upon which older nodes may rely.

### 13.2 Network Configuration

If the initial network configuration specified an admin key, an updated network
configuration can be applied automatically to peers on that network. This is
done by signing the Merkle root of the network configuration with the admin
private key and propagating the new configuration and signature to all peers
through the neighbor gossip protocol. Otherwise, a network configuration upgrade
must be applied through physically accessing the node.

When a new configuration is applied, the older configuration is retained, and
the node becomes a bridge between the two networks. Once a node sees that its
peers on the initial network are all bridging to the new network, it will
release its address on the initial network and continue using only the new/
updated network configuration. The initial configuration is retained in an
inactive form for compatibilty to bootstrap joining nodes that do not have the
upgraded network configuration or to downgrade at a later date.

Using neighborhood gossip, a peer can introduce an updated network configuration
for the autonomous network. When an updated configuration is encountered that
modifies only the anti-spam parameters (exclusive of the token issuance spec),
if those anti-spam parameters are closer to the optimal parameters as estimated
by the node than those of the active configuration, the new configuration will
be adopted by the node. In this way, an autonomous network can automatically
adjust its service throttling to maintain link efficiency and mitigate link
congestion.


## 14. References

@todo
