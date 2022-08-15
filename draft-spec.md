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
assignment it encounters; the mechanism by which changes in routing or policy
propagate through the network; route table construction; a system for bridging
between networks; a tunneling system; a rudimentary routing and route discovery
system; a set of security models used for spam protection, privacy, peer trust
scores, peer reliability tracking, bandwidth allocation and reciprocity, and
Sybil attack resistance. As an additional benefit of the protocol's use of
Ed25519 digital signatures, each autonomous network will have a built-in public
key directory, and these directories combine to form a distributed hash table.
Peers discover each other through gossip about the DHT and use ephemeral ECDHE
to provide forward secrecy. The exact structure of packets is beyond the scope
of this document, but some general guidelines are proposed. Finally, potential
transport, session, and application layer protocols as well as upgrades to
Mycelium version and network configuration are explored.

## Table of Contents

1. Introduction
2. Terminology
3. Autonomous Networks
    - Topologies and Data Links
    - Configuration
    - Domain Identifier
4. Address Format
    - Mesh Address Elements
    - Bus Address Elements
    - Ring Address Elements
    - Spanning Tree Address Elements
5. Address Assignment/Allocation
    - Bootstrapping a New Network
    - Topological Measurements
    - Leasing of Guest Addresses from Hosts
    - Calculation and Assignment of Address
    - Verification of Address Assignment by Peers
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
    - Distance Heuristic
    - Congestion Detection and Mitigation
    - Tunneling
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
14. Epilogue
15. References


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
- relay: a host that provides tunneling services.
- neighbors: nodes which can communicate with each other directly over a link.
- packet: a Mycelium header plus payload.
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
- hash function: a cryptographic one-way function mapping an arbitary length bit
string to a single, fixed-length output; given the output of a secure hash
function, it is computationally infeasible to determine the input, but it is
easy to verify that an input maps to an output by evaluating the hash function.
- Merkle tree: a binary tree data structure in which a data set is separated
into "leaves", each of which is the hash of the datum; two leaves are hashed
together to create a node; and nodes are combined two at a time to create more
nodes until arriving at a single hash output, called the root. Given a Merkle
root, it is possible to prove that a data point is a leaf without revealing the
entire data set, which is useful for privacy and bandwidth-saving purposes. In
general, a Merkle tree has 2^n number of leaves, sum(2^i for i=1 to n) nodes,
and inclusion proofs with n elements.
- Merklized data structure: a hashed data structure that retains the Merkle tree
properties of compressing a data set into a single cryptographic commitment
which can be used to create efficient inclusion proofs; however, not all leaves
must have the same depth within the hash tree, leading to variable-length
inclusion proofs.
- domain identifier (DID): the unique ID for an autonomous network.
- peer identifier (PID): an identifier created from a DID, an address from that
network, and the public key of the peer with that address.
- content identifier (CID): a hash or Merkle root of some data.
- Hamming distance: the number of different bits between two blocks of data,
i.e. the number of 1s in the output of XORing two equal-length bit strings; if
the bitstrings do not have equal length, a number of least significant bits of
the larger bitstring are dropped equal to the difference in length before the
XOR operation, and that number may added to the result.
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
- Ed-PoW: a Hashcash-esque proof-of-work system for spam protection whereby the
public key of an Ed25519 key pair contains a number of preceding null bits; this
works by generating a seed and XORing it with an incrementing nonce or using the
technique from BIP32 adapted for Ed25519 until a key pair is generated meeting
the pubkey preceding null bit requirement. In the case of MuSig/MuSig2, one of
the participants will have to run the PoW system until the aggregate key meets
the PoW requirement, then sign a message endorsing the new participant key pair.
- Sybil attack: an attack in which a bad actor pretends to be many nodes to
manipulate a network, unfairly distort network activity, or steal a network
service.

Note: it is expected that each host act as a router and relay, but any host may
decide to forward packets only on a subset of its interfaces. A host that leases
an address to a guest is explicitly responsible for forwarding that guest's
packets, but any host may decline to lease addresses to guests. A relay that
leases a tunnel guest address to a node is responsible for forwarding that
tunnel's packets, but each host will have its own independent policies on relay
availability, cost, etc. Each host will also have its own independent policies
for providing routing and bridging services to peers, and policy guidance will
be provided in section 7.


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
- Mesh: a mesh of peers sharing a radio link, variety of radio links, or a
sparsely connected graph of directional links; the topology is described by a
normed vector space and a distance metric.
- Bus: a group of peers sharing a data bus link; effectively a 1-dimensional
mesh network.
- Ring: a group of peers sharing links configured in a ring topology.
- Spanning tree: a group of peers organized in a connected graph structure with
an elected root and an address system that embeds the network structure.

An autonomous network can be configured for a single link or a set of links with
similar topological characteristics, e.g. a network comprised of 802.11 and LoRa
radios for which a mesh topology is most appropriate, or ethernet and PPP links
for which spanning tree is an appropriate topology. Mycelium nodes will attempt
to determine the optimal topologies for their local networks, and the iterative
process should refine the accuracy of the topologies and utility of directories
over time.

### 3.2 Configuration

An autonomous network is configured with a Merklized data structure specifying
the following parameters:

- Sub-tree 1: general parameters
    - Mycelium version: the hash of the specification or reference
    implementation of the Mycelium version.
    - Network topology type: the expected network topology to model, i.e. mesh,
    bus, ring, or spanning tree as described in this document, or a topology
    described in a later version.
    - Data links: a list of descriptions of data links to be included in the
    network.
    - Retransmission limit: the maximum number of times a node should attempt
    to resend a given packet.
    - Forwarding threshold: a distance threshold used for determining whether a
    router will attempt to forward a packet in a network.
    - Bootstrap URIs: (optional) a list of addresses on data links useful for
    bootstrapping.

- Sub-tree 2: topology model parameters
    - Address format: the number and data format of address elements used to
    describe a node's location in the modeled topology; the final element is the
    guest-specific address element.
    - Distance metric: which distance metric to use for gradient descent, i.e.
    L1 norm (taxicab geometry), L2 norm (Euclidean), L3 norm, L-infinity norm
    (Chebyshev), arbitrary Minkowski (tuned p parameter), none, or some other
    metric introduced in a later version. Valid only for mesh and bus models.
    - Measurement thresholds: an ordered list of 1-byte ints corresponding to
    the number of observations that must be made for each type of measurement
    for the specific topology.
    - Gradient threshold: the maximum gradient value for a set of coordinates
    given vectors of neighbor coordinates and distance measurements to those
    neighbors. Valid only for mesh and bus models.
    - Error threshold: the maximum mean squared error value for a set of
    coordinates given vectors of neighbor coordinates and distance measurements
    to those neighbors. Valid only for mesh and bus models.

- Sub-tree 3: security parameters
    - Hashcash threshold: the minimum hashcash proof-of-work difficulty for a
    message to be honored by peers.
    - Ed-PoW threshold: the minimum Ed-PoW difficulty for a public key address
    to be honored by peers; effectively an anti-spam measure for joining nodes.
    - @todo complete the security params

- Sub-tree 4: network administration details
    - Network origin public key: the public key of the network origin; can be
    the public key of an individual node or the aggregate key for several nodes.
    - Network admin key(s): the public key(s) of the network admin(s) used for
    updating the network configuration.

The hash function for constructing the network configuration hash sub-trees will
be sha256, and the roots of each of the sub-trees will be combined into one tree
committing the whole network configuration to a single Merkle root.

### 3.3 Domain Identifier

Each autonomous network is specified by a configuration as outlined above, and
the domain identifier (DID) is the Merkle root of the configuration.


## 4. Address Format

Addresses assigned to hosts will follow a format specified in the network
configuration as used in a topology model. The topological data will be encoded
by XORing the address elements with the network DID. Guest addresses will be the
address of the host XORed with a guest-specific address element encoding either
the latency measured between host and guest or an arbitrary bit string.

Each address format specification must encode no greater than 256 bits of data.
Address elements will be packed into a bit string padded to the length of the
DID bit string. The address element bit string will be XORed with the DID to
produce the address.

### 4.1 Mesh Address Elements

A mesh topology network uses tuples of floats as coordinates to encode the
location of each node in relation to its peers within a normed vecor space. Each
mesh network configuration will include the number of dimensions of the vector
space, the norm (distance metric), and the data types used for each coordinate
and the guest address element. The recommended number of dimensions is 3, and
the recommended data type for each address element is float64.

### 4.2 Bus Address Elements

A bus data link is topologically similar to a one-dimensional mesh. Neighbors
should not need to route packets for each other, but retransmission may be a
necessary or useful feature. Thus, a single float64 will be sufficient for
addressing hosts on a bus link and determining whether or not to retransmit.

### 4.3 Ring Address Elements

Ring networks are characterized by routers that are each connected to exactly
two neighbors. Ring networks can have unidirectional or bidirectional
transmissions.

For rings with unidirectional transmissions, all request-response communications
between peers will have theoretically equal latencies, so latency is useless for
modeling the topology. However, a ring network will have a measurable number of
hops for each transmission. Mapping the entire network topology can be achieved
by each router adding its public key to the end of a list and retransmitting the
list. Once the list is complete, each router can then count the number of hops
required to receive a transmission from the network origin as well as the number
of hops required for the network origin to receive a response. These two numbers
will each be encoded as an integer; the recommended encoding is uint32.

For rings with bidirectional transmissions, two float elements will be used to
indicate distance in the two chiralities (clockwise and anticlockwise). The
recommended encoding is float64.

### 4.4 Spanning Tree Address Elements

A spanning tree network is controlled either by a central router or a hierarchy
stemming from a central router. For a non-hierarchical spanning tree network,
the router will have 256 bits to use in address assignment. For hierarchical
spanning tree networks, there are two possible models: one with manual
assignment of address blocks and one with automatic address block assignment.
For the manually operated network, each address assignment will include an
address block over which the host has the authority to assign addresses or
further delegate address assignment through smaller blocks. For the automatic
network, each level down the hierarchy will reduce the assignable address block
size by a number of bits specified in the network configuration corresponding to
the maximum number of hosts at each level; the suggested value for this
parameter is 10. Thus, each element of an address in a spanning tree network
corresponds to the peer's assigned position within that layer of the hierarchy.
For example, IPv4 and IPv6 could be implemented as manually-operated,
hierarchical spanning tree Mycelium networks.


## 5. Address Assignment/Allocation

The mechanism by which an address is assigned/allocated and the mechanisms for
measuring and bootstrapping a network are specific to the topological model
specified in the network configuration:

- Mesh and bus: each peer estimates its location in the network topology using
the locations of peers and the round-trip latencies for each; each such estimate
will be verified by nodes before adding to the directory.
- Ring: a list is circulated directionally to which each peer appends an entry
with its public key and a set of timestamps; when the list is complete, all
addresses can be computed simultaneously by all peers.
- Spanning tree: addresses are either assigned manually or automatically in a
centralized/hierarchical manner; address assignments are verified by
certificates stemming from the central authority, which may be elected.

### 5.1 Bootstrapping a New Network

Bootstrapping is a process that determines the network configuration once peers
connect on a link, using a unique mechanism for each topology:

- Mesh and bus: when the first two neighbors on a link discover each other, they
exchange network configurations for the link and their public keys. For each
network configuration proposed, the nodes run a tie-breaking function by sorting
and concatenating their public keys, then hashing the concatenation with sha256.
The network configuration with the DID with the least Hamming distance from the
hash becomes the network configuration.
- Ring: when the ring is completed and all peers can send request packets and
receive response packets, the network will be booted by broadcasting the public
keys of all peers. The public keys will be sorted, concatenated, and hashed, and
the node whose public key has the least Hamming distance from the hash becomes
the network origin.
- Spanning tree: each peer will prepare a network configuration with itself as
the origin and transmit the network configuration to its neighbors if it is a
central router for all of its neighbors and the only possible router for them.
Alternately, the network configuration may have a target bitstring, and the
router with the least XOR distance from that target is elected as the origin.

### 5.2 Topological Measurements

The goals of Mycelium nodes are to model the topologies of the networks they can
create over the available links and route traffic using those models. Thus,
peers must take measurements and use those measurements to determine the
topology and addresses of peers on the network. Using an iterative process,
networks of Mycelium nodes gradually improve their topology models, dropping old
models as they become invalidated and adopting new models that better fit the
available data.

To determine which topology models are appropriate for a link, a node will send
beacon packets on its links identified with a UUID and the node's public key.
Each beacon packet will include proof-of-work as described in 11.1 Spam
Protection as well as a hop limit. Any node that receives a beacon packet will
add its public key and digital signature to the list of beacon witnesses and
broadcast the amended beacon packet if it meets the node's policy criteria.

The measurement methods will be specific to each topology model:

#### Mesh and Bus

The appropriate measurement is round-trip latency between two neighbors. Each
packet will include a sequence byte incremented upon each transmission and
any request message or measured latency value. All transmissions will include
a session code for synchronization, and latency will be measured in milliseconds
and encoded with the guest address element data type.

When node A requests to measure the link latency with node B, it includes a
sequence byte seqA and a request to make N measurements. Node B's response will
include sequence byte seqB, an agreement to make N measurements, and a request
to make M measurements of its own. Node A will respond with incremented seqA, an
agreement to make M meaurements, and its first measurement of latency. Node B
will respond with incremented seqB and its first measurement of latency. Node A
and node B continue exchanging incremented sequence bytes and latency
measurements until either they have shared a number of measurements equal to the
greater of N and M or until a transmission error causes the protocol to fail. In
the case of a failure due to transmission error, after a delay equivalent to 5
times the average latency measured to that point, the node which successfully
sent the last packet will attempt to restart the process within the session.
The neighbors then average their latency measurements and sign an attestation
with MuSig.

#### Ring

The appropriate measurement for unidirectional rings is the number of hops in
the ring; latency measurements can be made for the whole ring but are not
particularly helpful. The appropriate measurement for a bidirectional ring, on
the other hand, is the latency of each hop.

Hosts will transmit and retransmit beacon packets containing lists of public
keys, timestamps, and signatures, each appending its own public key, timestamp,
and signature to each list before forwarding. When a given host's beacon
traverses an entire unidirecitonal ring, that host will have a complete topology
of the ring.

For a bidirectional ring created by connecting each host with its neighbors over
unique links,

@todo

#### Spanning Tree

Topological measurements are not necessary in a spanning tree. However,
link reliability may be considered when a node chooses a peer from which it will
lease an address.

### 5.3 Leasing of Guest Addresses from Hosts

After a topology measurement attestation is made between two neighbors, the node
whose public key has the lowest Hamming distance to the DID requests a guest
address lease from the other node. If only one neighbor is a host, the guest
node requests a guest address lease from the host. The host then leases a guest
address if the request met its policy requirements.

In the spanning tree model, hosts have total discretion over how they assign
guest addresses, so topological measurements can be made but are not necessary.
In automatic construction mode, once an origin router is configured or elected,
that origin assigns an address with a single random value as the first address
element to each peer and null for the remaining elements. Those peers in turn
add another random value as the second address element to assign to their peers.
This assignment process propagates through the graph until the full tree has
been formed. This specification uses Virtual Overlays Using Tree Embeddings
(VOUTE) by Stefani Roos et al. and An Algorithm for Distributed Computation of a
Spanning Tree in an Extended LAN by Radia Perlman as references.

### 5.4 Calculation and Assignment of Address

The node with the new guest address and topological measurement attestation then
updates its topological model:

- Mesh and bus: the node runs gradient descent in the n-dimensional address
space with at least n+1 of its most recently leased guest addresses/latency
attestations obtained from unique hosts. That result is then used as the new
host address elements, which is encoded by XORing with the DID, combined with
the attestations and digital signature of the host, and gossiped through the
network.
- Ring, unidirectional: @todo
- Ring, bidirectional: @todo
- Spanning tree: each host with address assignment authority has discretion to
assign address blocks to its guests to make them hosts/routers and should do so
if a guest attests to accessing multiple nodes that the host cannot access and
all its policy requirements are met. Each host address assignment will include
the digital signature chain proving authority delegated from the admin keys set
in the network configuration or a proof of origin election.

### 5.5 Verification of Address Assignment by Peers

- Mesh and bus: when a new host address claim is encountered by a peer, that
peer calculates the gradient and mean squared error using the address elements
and the measurement attestations. If the gradient and MSE are below the
thresholds set in the network configuration, the address is verified.
- Ring, unidirectional: @todo
- Ring, bidirectional: @todo
- Spanning tree: each host checks the digital signature chain attached to each
address assignment. If the digital signatures are valid, the address allocation
rules are followed, no assignment in the signature chain has expired, and the
root digital signature is authorized in the network configuration or belongs to
a node with the lowest XOR distance from the target bitstring, the address is
verified.

Once an address assignment is verified, the gossip is marked for retransmission,
and the address assignment is added to the local copy of the network directory,
replacing any older entries for that public key.

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

For enhanced privacy applications and network bridging, hosts may enable a
tunneling service (i.e. become a relay). Each tunnel will take the form of a
guest address assigned to the requesting peer by the relay and will be
identified with an ephemeral public key used as the guest public key. The guest
will pay any costs for the tunnel specified by relay policy, and the relay will
route packets from the guest address to the peer's original address and vice
versa. Tunnels may include randomized delays to mix traffic and confound
attempts at traffic analysis.

Each packet traversing a tunnel will be encrypted in layers. When a node with a
multihop tunnel sends a packet, it encrypts the last-hop packet first using the
private key for that guest address and the public key of the destination; it
wraps that packet in a request to forward addressed to the last relay encrypted
with the private key for second-to-last-hop guest address and the last relay's
public key; and it continues doing this until all tunnel hops have been included
in the layer-encrypted packet. When the layer-encrypted packet is received by
the first (closest) relay in the tunnel, the outermost layer of encryption is
removed, and the remaining packet is forwarded to the next relay; this occurs
until the last layer of encryption is removed and the final packet is sent to
its destination. When a packet is sent to a tunnel, the relay encrypts the
whole packet before forwarding to the next hop in the tunnel; this continues
until the packet reaches the originator of the tunnel which can then decrypt
each layer.

A 0-hop tunnel is created when a host leases a guest address to itself. It is
recommended that every tunnel start with a 0-hop tunnel.

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

### 9.1 Distance Heuristic

@todo

### 9.2 Congestion Detection and Mitigation

@todo

### 9.3 Tunneling

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


## 14. Epilogue

@todo


## 15. References

@todo
