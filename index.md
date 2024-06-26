---
layout: default
title: Replicated Object Notation
---

# Replicated Object Notation

Replicated Object Notation (RON) is a format for *distributed live data*.
RON's primary mission is continuous data synchronization.
A RON object may naturally have any number of replicas, which may synchronize in real-time or intermittently.
[JSON](htp://json.org), [protobuf](https://developers.google.com/protocol-buffers/),
and many other formats implicitly assume serialization of separate state *snapshots*.
RON has versioning and addressing *metadata*, so state and updates can be always pieced together.
RON handles state and updates all the same: _state is change and change is state_.

Every RON object, every change, every version has a globally unique UUID.
There is no nesting in the RON syntax. Pieces of data reference each other by UUIDs.
RON itself is a [regular language](https://en.wikipedia.org/wiki/Regular_language);
its syntax is as simple as the one of [ini files](https://en.wikipedia.org/wiki/INI_file).
Thanks to the references, RON can express any nesting and, in general, arbitrary graphs of objects.

RON UUIDs are effectively [hybrid timestamps](https://cse.buffalo.edu/tech-reports/2014-04.pdf).
That makes RON very suitable for expressing distributed/partial-order
data structures, especially [CRDTs](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type).
Thanks to that, different data pieces and versions are always mergeable.

Yet another way to look at it: RON is like the [metric system](https://en.wikipedia.org/wiki/Metric_system) but for data.
The [imperial system](https://en.wikipedia.org/wiki/Imperial_units)
employed various usage-based units: foots, lines, furlongs, links, cables, etc.
The metric system defines one unit (the meter), then derives other units from that.
RON defines the immutable [operation](/specs/ops) ("op"), then derives other units from that,
be that data structures (arrays, maps, sets, etc) or storage/transmission units
(snapshots, logs, batches, patches, [etc](/specs/glossary)).

Here is a simple object serialized in RON:

<pre>
<span class="line">  1 </span><span class="id">@1fLDV+biQFvtGV</span> <span class="ref">:lww</span><span class="term">,</span>
<span class="line">  2 </span>    <span class="string">&apos;id&apos;</span>        <span class="string">&apos;20MF000CUS&apos;</span><span class="term">,</span>
<span class="line">  3 </span>    <span class="string">&apos;type&apos;</span>      <span class="string">&apos;laptop&apos;</span><span class="term">,</span>
<span class="line">  4 </span>    <span class="string">&apos;cpu&apos;</span>       <span class="string">&apos;i7-8850H&apos;</span><span class="term">,</span>
<span class="line">  5 </span>    <span class="string">&apos;display&apos;</span>   <span class="string">&apos;15.6” UHD IPS multi-touch, 400nits&apos;</span><span class="term">,</span>
<span class="line">  6 </span>    <span class="string">&apos;RAM&apos;</span>       <span class="string">&apos;16 GB DDR4 2666MHz&apos;</span><span class="term">,</span>
<span class="line">  7 </span>    <span class="string">&apos;storage&apos;</span>   <span class="string">&apos;512 GB SSD, PCIe-NVME M.2&apos;</span><span class="term">,</span>
<span class="line">  8 </span>    <span class="string">&apos;graphics&apos;</span>  <span class="string">&apos;NVIDIA GeForce GTX 1050Ti 4GB&apos;</span><span class="term">,</span>
<span class="line">  9 </span><span class="id">@1fLDk4+biQFvtGV</span>
<span class="line"> 10 </span>    <span class="string">&apos;wlan&apos;</span>      <span class="string">&apos;Intel 9560 802.11AC vPro&apos;</span><span class="term">,</span>
<span class="line"> 11 </span>    <span class="string">&apos;camera&apos;</span>    <span class="string">&apos;IR &amp; 720p HD Camera with microphone&apos;</span><span class="term">,</span>
<span class="line"> 12 </span><span class="comment"><i>@sha3</i></span> <span class="string">&apos;SfiKqD1atGU5xxv1NLp8uZbAcHQDcX~a1HVk5rQFy_nq&apos;</span><span class="term">;</span>
</pre>

Key RON principles are:

- **Immutability** - RON sees data as a collection of immutable timestamped ops.
        In the example above, we have an object state consisting of ten ops
        (object creation op at line #1, seven ops in the initial changeset #2 to #8,
        another changeset of two ops #9/10 and #11).
        An op may be referenced, transmitted, stored, applied or rolled back,
        garbage collected, etc.
        Every RON data structure (array, object, map, set, etc)
        is a collection of immutable ops.
        Similarly, every data storage or transmission unit is made of ops
        (patch, state, chain, chunk, frame, object graph, log, yarn,
        [etc](/specs/glossary)).
- **Addressability** of everything. Changes, versions, objects and every
        piece of data is uniquely identified and globally referenceable.
        Above, the first op has an id `1fLDV+biQFvtGV`, the second one is
        `1fLDV00001+biQFvtGV`, the third is `1fLDV00002+biQFvtGV`
        and so on (the notation skips incremental ids).
        The last two ops (#9-10 and #11) belong to a later changeset, so their
        ids are `1fLDk4+biQFvtGV`, `1fLDk40001+biQFvtGV`. <br/>
        Note: RON has no notational nesting (no brackets).
        Instead, data pieces reference each other by UUIDs, thus forming arbitrary graphs.
- **Causality**. Each RON operation explicitly *references* what other op
        it is based on.
        No matter how and when you get your data, you can always reconstruct
        the correct order and location of data pieces.
        Above, ops form an orderly chain, so references are skipped, except
        for the object creation op at line #1 which references its data type `lww`
        That is last-write-wins, a simple key-value object.
        There are many other [types](/rdts).
- **Efficiency**.
        An op is a very fine-grained unit of change.
        Thus, RON has to optimize per-op metadata overhead in numerous ways.
        Op ids get skipped if they go incrementally.
        References are skipped if they point to the previous op
        (an op chain is a convenient default).
        For example, ops at lines #2-#8 mention neither their own ids
        (the first plus 1) nor their references (always the preceding op).
        The binary variant of RON employs more sophisticated metadata
        compression techniques.
        With no abbreviations, the object would look like a tabular log of ops, two
        metadata UUIDs per op:
<pre style="font-size: 80%;">
<span class="line">  1 </span><span class="id">@1fLDV00000+biQFvtGV</span> <span class="ref"> :lww</span> <span class="term">,</span>
<span class="line">  2 </span><span class="id">@1fLDV00001+biQFvtGV</span>  <span class="ref">:1fLDV00000+biQFvtGV</span> <span class="string">&apos;id&apos;</span>        <span class="string">&apos;20MF000CUS&apos;</span><span class="term">,</span>
<span class="line">  3 </span><span class="id">@1fLDV00002+biQFvtGV</span>  <span class="ref">:1fLDV00001+biQFvtGV</span> <span class="string">&apos;type&apos;</span>      <span class="string">&apos;laptop&apos;</span><span class="term">,</span>
<span class="line">    ...</span>
</pre>
- **Integrity**, as ops form a [Merkle structure](https://en.wikipedia.org/wiki/Merkle_tree).
        If necessary, the data is integrity-checked to the last bit like
        in git, BitTorrent, BitCoin and other such systems.
        In the example, ten ops form a Merkle chain, so the hash of the last op
        (line #12) covers them all.

RON's guiding vision is swarms of mobile devices communicating over unreliable wireless networks in an untrusted environment.

For more in-depth reading, please see:

* an explanation of [RON UUIDs](/uuids/)
* the [protocol glossary](/specs/glossary) and [specification](/specs/), also its serializations: text-based (above), binary, JSON-based, CBOR, nominal
* [RON Replicated Data Types](/rdts/), an ["operational"](http://archagon.net/blog/2018/03/24/data-laced-with-history/)
        variant of [CRDTs](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)
        able to operate in different modes (op-based, state-based, delta-enabled).
* [SwarmDB](/swarm/), a reference implementation of a RON-based syncable key-value store.
