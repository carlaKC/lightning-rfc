# HTLC Endorsement

## Table of Contents
* [Proposal](#proposal)
  * [Introduction](#introduction)
  * [Threat Model](#threat-model)
  * [Mitigation](#mitigation)
    * [Slow Jamming Mitigation](#slow-jamming-mitigation)
    * [Endorsement Mechanism](#ensorsement-mechanism)
    * [Local Reputation](#local-reputation)
    * [Resource Bucketing](#resource-bucketing)
    * [Long Held HTLCS](#long-held-hltcs)
    * [Justification](#justification)
* [Network Upgrade](#network-upgrade)
* [FAQ](#faq)
* [References](#references)

## Proposal

### Introduction
Channel jamming is a known denial-of-service attack against the Lightning 
Network. An attacker that controls both ends of a payment route disrupt usage 
of a channel by sending payments that are destined to fail through a target 
route. This attack can exhaust the resources along the target route in two ways, 
though the difference between the two behaviors is not rigorously defined: 
1. Quick Jamming: sending a continuous stream of payments through a route that
   are quickly resolved, but block all of the liquidity or HTLC slots along 
   the route.
2. Slow Jamming: sending a slow stream of payments through a route and only 
   failing them at the latest possible timeout, blocking all of the liquidity
   or HTLC slots along the route for the duration.

Fees are currently only paid to routing nodes on successful completion of a 
payment, so this attack is virtually free - the attacker pays only for the 
costs of running a node, funding channels on-chain and the opportunity cost
of capital committed to the attack.

### Threat Model
Attackers may aim to disrupt a specific set of channels, or the network as a
whole. We consider the following threat model, adapted from [1]: 
* The attacker can quickly set up a set of seemingly unrelated nodes and open
  channels to any public node in the network.
* The attacker has an up to date view of the network's public topology. 
* The attacker is economically rational, meaning that they will pursue an 
  attack with the lowest cost and highest utility (and "seeing the world burn" 
  classifies as utility).
* The attacker has the ability to send slow or quick jams: 
  * Quick jamming attacks will aim to fill all available htlc slots if
    `max_accepted_htlcs` * `htlc_minimum_msat` < `max_htlc_value_in_flight_msat`,
    or otherwise will aim to deplete channel liqudidity.
  * The attacker has a modified LN node implementation that allows them to hold
    HLTCs up until the point of force closure, then release them.
* The attacker may have access to long-lived channels when the attack begins.

### Mitigation
To comprehensively mitigate jamming attacks against the Lightning Network, a
solution needs to address both quick and slow jamming. This proposal covers
local HTLC endorsement to address slow jamming. A follow up proposal will 
outline the use of upfront fees to address quick jamming [2].

The goal of this proposal it to produce a mitigation that has a trivial impact
on honest nodes (and honest payments), and significant cost to persistent 
attackers. 

### Slow Jamming Mitigation
Slow jamming attacks are easier to identify because they lock liquidity for
an atypically long period of time. We propose local HTLC endorsement and 
resource bucketing to protect against slow jamming attacks:
* HTLC Endorsement: a signal from forwarding nodes that they expect a HTLC to 
  resolve quickly. The absence of endorsement does not classify a HTLC as "bad",
  but rather that it is unknown - the forwarding node makes no promises about
  the manner in which it will resolve.
* Resource Bucketing: division of a node's scarce resources (liquidity and 
  slots) into a portion for traffic that is expected to resolve quickly, and 
  a portion for unknown (and potentially risky) payments. 

#### Endorsement Mechanism
An endorsement TLV field  is added to `update_add_htlc` to allow sending nodes
to signal whether they endorse a HTLC. This field is binary to prevent leaking
the HTLC's location in the route. 

```
1. `tlv_stream`: `update_add_htlc_tlvs`
2. types:
    1. type: 1 (`endorsed`)
    2. data:
        * [`byte`:`endorsed`]
```

A node will signal that it endorses a HTLC if:
* The are the original sender of a payment (see [network upgrade](#network-upgrade)
  for comment on privacy implications).
* The HTLC is endorsed by the upstream node *and* the forwarding node considers
  the upstream node to have good reputation.

#### Local Reputation
Forwarding nodes will be required to locally track the reputation of their peers
to decide how to allocate resources, and whether to endorse the HTLC when they
forward it onwards. A reputation scoring system should consider: 
* The time a HTLC takes to resolve.
* The historical fees earned from forwards originating from the peer.
* Whether its endorsed HTLCs resolved as expected.

Note: specification of a recommended reputation algorithm is ongoing at the
time of writing.

#### Resource Bucketing
When making the decision to forward a payment, nodes need to consider their two 
scarce resources: liquidity and HTLC slots. These resources are divided into 
two buckets (with allocation set per the node's risk tolerance):
* Endorsed Payments: the liquidity and slots in this bucket are reserved for 
  HTLCs that are endorsed by a peer that the forwarding node considers to have
  good reputation.
* Unknown Payments: HTLCs that are not endorsed, or come from a peer that the
  forwarding node considers to have bad reputation may only occupy liquidity 
  and slots in this bucket. Once the liquidity and slots in this bucket have 
  been filled, the peer will not forward any more unknown payments.

We suggest a 50/50 split by default. This division can accommodate the suggested
maximum HTLC that nodes should allow per the Oakland Protocol (which suggests
capping maximum HTLC size at 50% of the channel capacity to prevent balance 
probing). Nodes wishing to disable this protection entirely have the option of
allocating 100% of resources to the unknown payments bucket.

#### Long Held HTLCs
There are some "honest" instances where a HTLC may be held for a longer period 
of time in the network:
1. Submarine Swaps [3]: the exchange of off-chain liquidity for on-chain funds
   (or the reverse) can lock funds for 3-6 blocks in the optimistic case, and 
    50-100 blocks in failure mode.
2. Offline Nodes: if a forwarding node goes offline, payments that are 
   in-flight along its route can get "stuck" until the route's timeout has 
   elapsed.
3. DLC contracts[4]: real-world "bets" that wait for the outcome of an event 
   and oracle signature to resolve.

The protocol will need to be upgraded to serve this type of traffic if it grows
to be a significant portion of activity. For example, the fee structure will 
need to account for longer hold times to compensate forwarding nodes. We 
suggest explicit signaling and appropriate compensation for long-held payments, 
though we leave the specifics to future work.

### Justification
Division of resources into endorsed and unknown buckets allows the network to 
degrade gracefully in the event of an attack. Nodes that have demonstrated good
behavior will have a portion of the network's resources reserved for their 
payments, so will be able to continue with regular - albeit slightly degraded - 
operation. 

Any proposal that defines a threshold above which abuse is penalized is open 
to being gamed by an attacker that targets activity to fall _just_ below the
threshold for "bad behavior". We suggest reputation as a first step to mitigate
the most egregious jamming attack on Lightning (slow-jamming) so that the 
network has some protection against attack, and recommend that it is followed 
by upfront fees as a mechanism to charge nodes that repetitively fall just 
below detection thresholds.

## Network Upgrade
Endorsement signaling is a link-level upgrade to the network - a full route 
does not need to be upgraded for individual nodes to signal endorsement. The
use of an odd TLV field also allows nodes to include the signal even if the
next hop does not support it. Upgraded nodes that receive HTLCs that do not
include the TLV can simply treat the absence as a lack of endorsement. There
is no need to enforce presence of the TLV, because attackers have nothing to
gain by omitting it - their traffic will simply be directed to the unendorsed
bucket.

Quality of implementation note: We suggest that senders fuzz endorsement of 
their own payments as the network upgrades to mitigate the privacy concerns of 
identifying yourself as the sender through your endorsement field.  

## FAQ
1. What if the majority of nodes do not allocate any resources to unknown 
payments?

Profit maximizing nodes will manage the trade-off between fee revenue and risk
of forwarding unendorsed traffic. While some may choose to only forward 
endorsed payments, a healthy routing market will also contain participants who
are willing to take on risk in exchange for higher routing volumes. 

2. Is endorsement a path to payment censorship in Lightning?

HTLC endorsement is local, and provides no mechanism for centralized entities 
to control the flow of traffic through the network. Parties looking to KYC or
censor transactions already have the ability to transmit data with payments 
using custom TLV values or payment metadata - an endorsement field with limited
information expressed does not exacerbate this reality.

## References
[1] [Spamming the Lightning Network](https://github.com/t-bast/lightning-docs/blob/master/spam-prevention.md) - Bastien Teinturier

[2] [Unjamming Lightning: A Systematic Approach](https://eprint.iacr.org/2022/1454.pdf) - Clara Shikhelman and Sergei Tikhomirov

[3] [Understanding Submarine Swaps](https://docs.lightning.engineering/the-lightning-network/multihop-payments/understanding-submarine-swaps) - Leo Weese

[4] [DLCs on Lighting](https://suredbits.com/discreet-log-contracts-on-lightning-network/) - Ben Carman
