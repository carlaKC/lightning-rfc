# Local Resource Conservation

## Table of Contents
* [Introduction]()
* [Overview]()
* [Local Reputation]()
  * [Guidelines for Reputation]()
  * [Recommendation for Reputation Scoring]()
* [FAQ]()

## Introduction

Channel jamming is a known denial-of-service attack against the Lightning 
Network. An attacker that controls both ends of a payment route can disrupt 
usage of a channel by sending payments that are destined to fail through a 
target route. This attack can exhaust the resources along the target route in 
two ways, though the difference between the two behaviors is not rigorously 
defined:
* Quick Jamming: sending a continuous stream of payments through a route that 
  are quickly resolved, but block all of the liquidity or HTLC slots along the 
  route.
* Slow Jamming: sending a slow stream of payments through a route and only 
  failing them at the latest possible timeout, blocking all of the liquidity 
  or HTLC slots along the route for the duration.

Fees are currently only paid to routing nodes on successful completion of a 
payment, so this attack is virtually free - the attacker pays only for the 
costs of running a node, funding channels on-chain and the opportunity cost of 
capital committed to the attack.

TODO: this solution for slow jamming / asymmetry

## Overview

Local resource conservation combines the follow elements to provide nodes with 
a mechanism to protect themselves against slow-jamming attacks:

* [HTLC Endorsement](): propagation of a signal along the payment route that 
  indicates whether the node sending [update_add_htlc](TODO) recommends that the 
  HTLC should be granted access to downstream resources (and that it will stake 
  its reputation on the fast resolution of the HTLC).
* [Local Reputation](): local tracking of the historical forwarding behavior 
  of channel peers, used to decide whether to grant incoming HTLCs full access 
  to local resources and whether to propagate endorsement downstream.
* [Resource Bucketing](): reservation of a portion of "protected" liquidity 
  and slots for endorsed HTLCs that have been forwarded by high reputation 
  nodes. 

Sequence: 
* The `update_add_hltc` is sent by an upstream peer.
* If the sending peer has good local reputation (per the receiving peer's view)
  AND the incoming HTLC was sent with `endorsed` set to `1`: 
  * The HTLC will be allowed access to the "protected" bucket, so will have 
    full access to the channel's liquidity and slots, 
  * The corresponding outgoing HTLC (if present) will be forwarded with 
    `endorsed` set to `1`.
* Otherwise: 
  * The HTLC will be limited to the remaining "general" slots and liquidity, 
    and will be failed if there are no resources remaining in this bucket.
  * The corresponding outgoing HTLC (if present) will be forwarded with 
    `endorsed` set to `0`.

In the steady state when Lightning is being used as expected, there should be 
no need for use of the protected slots as payments are resolved quickly. Should 
the network come under attack, honest nodes that have built up reputation over 
time will still be able to utilize protected resources to process payments in 
the network.

## HTLC Endorsement
*See bolt 2* 
Describe the discussions we've had around this

## Local Reputation

### Guidelines for Reputation

Local reputation can be used by forwarding nodes to decide whether to allocate 
resources to and signal endorsement of a HTLC on the outgoing link. Nodes MAY 
use any metric of their choosing to classify a peer as having "good" 
reputation, though a poor choice of reputation scoring metric may affect their 
reputation with their downstream peers. 

General recommendations for a reputation scheme are: 
* It relies on unforegable local metrics, such as fees paid to forward HTLCs, 
  to build reputation. 
* The actions required to build good reputation mimic honest behavior on the 
  network as closely as possible.
* It is slow to build with good behavior, and fast to degrade with bad behavior.
* There is no special action that an attacker can take to build reputation at 
  a faster rate than an honest node.
* It holds forwarding nodes accountable for their endorsement of HTLCs, but 
  does not punish undesirable resolution of unendorsed HTLCs.

### Recommendation for Reputation Scoring

The goal of this reputation metric is to ensure that the cost to obtain 
good reputation is greater than the damage that a node can inflict by abusing 
the access that it is granted. This algorithm uses forwarding fees to measure 
damage, as this value is observable within the protocol.

If granted full access to a node's liquidity and slots, the maximum amount of 
damage that can be inflicted on the targeted node is bounded by the following 
protocol constraints:
* Maximum HLTC hold: the largest cltv delta from the current block height that 
  a node will allow a HTLC to set before failing it with [expiry_too_far](), 
  typically 2016 blocks (~2 weeks) at the time of writing. 
* Maximum route length: the longest route that a valid [onion_packet]() can 
  facilitate, ~25 hops at the time of writing.

It is reasonable to expect an adversary to "about turn": behave perfectly to 
build up reputation, then alter their behavior to abuse it. For this reason, 
in-flight HTLCs have a temporary negative impact on reputation until they are 
resolved. 

When a HTLC is forwarded to a node, it considers the following to determine 
whether the sender of `update_add_htlc` is classified as having good 
reputation: 
* Outgoing link revenue: the total routing fees over the maximum allowed HTLC 
  hold period (~2 weeks) that the outgoing link has earned the local node, 
  considering both incoming and outgoing HTLCs that have used the link.
* Incoming link revenue: the total routing fees that the incoming link has 
  earned the local node, considering only incoming HTLCs on the link.
* In-flight HTLCs: any HLTCs that the incoming link has in-flight negatively 
  affect reputation, assuming that they will be held until their expiry 
  height.

~~~
Peers build reputation by forwarding successful HTLCs that resolve quickly, and
lose reputation if they endorse failing or slow-resolving HTLCs. Reputation is
only _negatively_ affected if an endorsed HTLC resolves undesirably, to hold
nodes accountable for their endorsement signal while still allowing them to 
forward unendorsed HTLCs that they are not certain about.

HTLC resolution time is assessed relative to a threshold that the node 
considers to be a reasonable amount of time for a HTLC to resolve: 
- `resolution_period`: the amount of time a HTLC is allowed to resolve in that 
  is classified as "good" behavior, expressed in seconds (default: 60 seconds).
- `resolution_time`: the time elapsed between `update_add_htlc` and its 
  resolution (`update_fulfill_hltc` / `update_fail_hltc` / 
  `update_fail_malformed_htlc`), expressed in seconds.

Each HTLC's contribution to reputation is expressed by its `effective_fees` 
which capture the fees that it paid and the opportunity cost that holding it 
for `resolution_time` incurred. 
- `fees`: the fees paid by a forwarded HTLC (as described in [BOLT #7](07-routing-gossip.md#htlc-fees), 
  equal to 0 if the HTLC was not fulfilled).
- `opportunity_cost`: `ceil ( (resolution_time - resolution_period) / resolution_period) * fees`

Given that `resolution_time` will be > 0 in practice, `opportunity_cost` is 0 
for HTLCs that resolve within `resolution_period`, and charges the `fees` that 
the HTLC would have earned per period it is held thereafter. This cost accounts 
for the slot and liquidity that could have otherwise been paid for by 
successful, fast resolving HTLCs during the `resolution_time` the HTLC was 
locked in the channel.

For every resolved incoming HLTC a peer has forwarded through a node, its 
`effective_fees` are calculated as follows:
- if `endorsed` = 1 in `update_add_htlc`:
  - `effective_fees` = `fees` - `opportunity_cost` 
- otherwise: 
  - if successfully resolved AND `resolution_time` <= `resolution_period`
    - `effective_fees` = `fees` * 0.5
  - otherwise: 
    - `effective_fees` = 0

Incoming in-flight HTLCs have a negative impact on reputation, as their 
influence is unknown until time of resolution. The `outstanding_risk` of each 
in flight HTLC is calculated as follows: 
- if `endorsed` = 1 in `update_add_htlc`:
  - `outstanding_risk` = `fees` * ( `cltv_expiry` - `height_added` * 10 * 60), 
    where `height_added` is the block height at which the HLTC was irrevocably
    committed to by the receiving node.

The `effective_fees` that a peer has paid our node are compared to our total 
routing revenue to classify a peer's reputation as "good" or "neutral". This
relates the fees that must be paid to earn "good" reputation to the damage that 
can be done by abusing it.

Routing revenue is assessed over a sliding window, so that reputation 
classification is related to the node's current routing activity. Nodes MAY
choose a window per their preferences - shorter windows being more reactive to 
recent routing patterns, and longer windows aggregating trends over time.

- `routing_window`: the period of time in which total routing revenue is 
  assessed (default: 10 minutes * the maximum number of blocks a HTLC may be
  held before the node will send `expiry_too_far`, as outlined in [Failure Messages](#failure-messages)).
- `routing_revenue`: the sum of all fees paid to the node to forward HTLCs
  over the interval [ `now` - `routing_window` ; `now` ].

The total `effective_fees` that an individual peer has generated are assessed
over a longer period of time to relate its reputation classification to the 
routing activity of the node (represented by `routing_revenue`).

- `reputation_window`: the period of time over which the node's `effective_fees`
  are calculated to classify its reputation (recommended default = 10 * `routing_window`).
- `reputation_revenue`: the total `effective_fees` of HTLCs forwarded by the peer
  over the interval [ `now` - `reputation_window` ; `now` ] less the sum of 
  `outstanding_risk` for all incoming, in-flight HLTCs from the peer.

A peer is classified as having "good" local reputation iff `reputation_revenue` >= `routing_revenue`, 
otherwise the peer's reputation is classified as "neutral".

Given the following graph, from Bob's perspective: 
```text
      ----update_add_htlc----> 
+-------+                  +-------+                  +-------+
| Alice |------------------|  Bob  |------------------| Carol |
+-------+                  +-------+                  +-------+
```

Alice will be considered to have good reputation if: 
* Outgoing link revenue: sum of all fees earned for HTLCs that were added by 
  Carol, or used Carol as the outgoing link over ~2 weeks.
* Incoming link revenue: sum of all fees paid for HTLCs that were added by 
  Alice.
* In-flight HTLCs: any HTLCs that Alice currently has in-flight on the 
  Bob -> Carol link.


### Rationale

A jamming attack can only be achieved with good reputation, as the resources 
available to neutral reputation nodes are limited by resource bucketing. If a 
node is attacked, it is guaranteed to have been paid at least its 
`routing_revenue` over the previous `reputation_window`.

Reputation is only negatively affected if a HTLC was endorsed to help protect 
the network's ability to forward "unknown" HTLCs - those from low-activity or 
new entrants that may very well be honest, but have not build up good 
reputation through a history of consistent, good behavior.

Intuitively if a HTLC is endorsed by a peer they have signaled that they expect 
the HTLC to resolve honestly, they should be held accountable for that signal.
By contrast, if a HTLC was not endorsed by the upstream peer, it should not 
have a negative impact on reputation. In the case where one of these 
"unknown" HTLCs succeeds within the reasonable resolution threshold, the 
peer is still credited with fees because the HTLC resolved desirably. This 
allows new, unendorsed entrants to slowly build reputation over time. The fee 
contribution of unendorsed HTLCs is discounted by 50% to incentivise nodes to 
endorse HTLCs.

In flight HLTCs are included in reputation scoring to account for sudden changes
in a peer's behavior. Even when good reputation is obtained, each HTLC choosing 
to take advantage of that good reputation is treated as if it will be used to 
inflict maximum damage. The expiry height of each incoming in flight HLTC is 
considered so that risk is directly related to the amount of time the HTLC 
could be held in the channel, and 10 minute blocks are assumed for simplicity.

## Resource Bucketing

When making the decision to forward a payment on its outgoing channel, a node 
MAY choose to limit its exposure to HTLCs that put it at risk of a denial of 
service attack.
* `unknown_allocation_slots`: defines the number of HTLC slots allocated to 
   unknown traffic (default: 50% of remote peer's `max_accepted_htlcs`).
* `unknown_allocation_liquidity`: defines the amount of the channel balance 
   that is allocated to unknown traffic (default: 50% of remote peer's 
   `max_htlc_value_in_flight_msat`.

A node implementing resource bucketing limits exposure on its outgoing channel:
- MUST choose `unknown_allocation_slots` <= the remote channel peer's 
  `max_accepted_htlcs`.
- MUST choose `unknown_allocation_liquidity` <= the remote channel peer's 
  `max_htlc_value_in_flight_msat`.
- If `endorsed` is set to 1 in the incoming `update_add_htlc` AND the HTLC 
  is from a node that the forwarding node considers to have good local 
  reputation (see [Recommendations for Reputation](#recommendations-for-reputation)):
  - SHOULD proceed with forwarding the HTLC.
  - SHOULD set `endorsed` to 1 in the outgoing `update_add_htlc`.
- Otherwise, the HTLC is classified as `unknown`:
  - If `unknown_allocation_slots` HTLC slots are occupied by other `unknown` HTLCs: 
    - SHOULD return `temporary_channel_failure` as specified in [Failure Messages](#failure-messages).
  - If `unknown_allocation_liquidity`  satoshis of liquidity are locked in 
    other `unknown` HTLCs: 
    - SHOULD return `temporary_channel_failure` as specified in [Failure Messages](#failure-messages).
  - SHOULD set `endorsed` to 0 in the outgoing `update_add_htlc`. 


