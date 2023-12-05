# RFC-0000: Metadata for offline signers

|                 |                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------- |
| **Start Date**  | 2023-10-31                                                                                  |
| **Description** | Add SignedExtension to check Metadata Root Hash                                             |
| **Authors**     | Alzymologist Oy, Zondax LLC, Parity GmbH                                                    |

## Summary

Add a metadata digest value (33-byte constant within fixed `spec_version`) to Signed Extensions to supplement signer party with proof of correct extrinsic interpretation. The digest value is generated once before release and is well-known and deterministic. The digest mechanism is designed to be modular and flexible and to support partial metadata transfer as needed by the signing party's extrinsic decoding mechanism and taking into account signing devices potentially limited communication bandwidth and memory capacity.

## Motivation

### Background

While all blockchain systems support (at least in some sense) offline signing used in air-gapped wallets and lightweight embedded devices, only few allow simultaneously complex upgradeable logic and full message decoding on the cold off-line signer side; Substrate is one of these heartening few, and therefore - we should build on this feature to greatly improve transaction security, and thus in general, network resilience.

As a starting point, it is important to recognise that prudence and due care are naturally required. As we build further reliance on this feature we should be very careful to make sure it works correctly every time so as not to create false sense of security.

In order to enable decoding that is small and optimized for chain storage transactions, a metadata entity is used, which is not at all small in itself (on the order of half-MB for most networks). This is a dynamic data chunk which completely describes chain interfaces and properties that could be made into a portable scale-encoded string for any given network version and passed along into an off-chain device to familiarize it with latest network updates. Of course, compromising this metadata anywhere in the path could result in differences between what user sees and signs, thus it is essential that we protect it.

Therefore, we have 2 problems to be solved:

1. Metadata is large, takes long time to be passed into a cold storage device with memory insufficient for its storage; metadata SHOULD be shortened and transmission SHOULD be optimized.
2. Metadata authenticity SHOULD be ensured.

As of now, there is no working solution for (1), as the whole metadata has to be passed to the device. On top of this, the solution for (2) heavily relies on a trusted party managing keys and ensuring metadata is indeed authentic: creating poorly decentralized points of potential failure.

### Solution requirements

#### Include metadata digest into signature

Some cryptographically strong digest of metadata MAY be included into signable blob. There SHALL NOT be storage overhead for this blob, nor computational overhead, on the node side; thus MUST be a constant within given runtime version, deterministically defined by metadata.

 - Metadata information that could be used in signable extrinsic decoding MAY be be included in digest, its inclusion MUST be indicated in signed extensions;
 - Digest MUST be deterministic with respect to metadata;
 - Digest MUST be cryptographically strong against pre-image, both first and second;
 - Extra-metadata information necessary for extrinsic decoding and constant within runtime version MUST be included in digest;
 - Digest format SHOULD be versioned to allow rapid withdrawal of cold signing devices in case severe security vulnerability is found in shortener mechanism;
 - Work necessary for proving metadata authenticity MAY be omitted at discretion of signer device design (to support automation tools).

#### Reduce metadata size

Metadata should be stripped from parts that are not necessary to parse a signable extrinsic, then it should be separated into a finite set of self-descriptive chunks. Thus, a subset of chunks necessary for signable extrinsic decoding and rendering could be sent, possibly in small portions (ultimately - one at a time), to cold device together with proof.

 - Single chunk with proof payload size SHOULD fit within few kB;
 - Chunks handling mechanism SHOULD support chunks being sent in any order without memory utilization overhead;
 - Unused enum variants MUST be stripped (this has great impact on transmitted metadata size; examples: era enum, enum with all calls for call batching).

## Stakeholders

This feature is essential for **all** offline signer tools; many regular signing tools might make use of it. In general, this RFC greatly improves security of any network implementing it, as many governing keys are used with offline signers.

Implementing this RFC would remove requirement to maintain metadata portals manually, as task of metadata verification would be effectively moved to consensus mechanism of the chain.

## Explanation

Detailed description of metadata shortening and digest process is provided in [metadata-shortener](https://github.com/Alzymologist/metadata-shortener) crate (see `cargo doc --open` and examples). Below are presented algorithms of the process.

### Metadata descriptor

Values for metadata shortening protocol version, `ExtrinsicMetadata`, SCALE-encoded `spec_version` and `spec_name` Strings, `u16` base58 prefix, `u8` decimals value, SCALE-encoded token unit String, should be prepared and combined as metadata descriptor.

### Metadata modularization

1. Types registry is stripped from `docs` fields.
2. Types records are separated into chunks, with enum variants being individual chunks differing by variant index; each chunk consisting of `id` (same as in full metadata registry) and SCALE-encoded 'Type' description (reduced to 1-variant enum for enum variants). Enums with 0 variants are treated as regular types.
3. Chunks are sorted by `id` in accending order; chunks with same `id` are sorted by enum vainant index in accending order.

### Merging protocol

`blake3` transformation of concatenated child nodes (`blake3(left + right)`) as merge procedure;

### Complete Binary Merkle Tree construction protocol

1. Leaves are numbered in ascending order. Leaf index is associated with corresponding chunk.
2. Merge is performed using the leaf with highest index as right and node with second to highest index as left children; result is pushed to the end of nodes queue and leaves are discarded.
3. Step (2) is repeated until no leaves or just one leaf remains; in latter case, the last leaf is pushed to the front of the nodes queue.
4. Right node and then left node is popped from the front of the nodes queue and merged; the result is sent to the end of the queue.
5. Step (4) is repeated until only one node remains; this is tree root.

### Digest

1. Blake3 hash is computed for each chunk of modular short metadata registry.
3. Complete Binary Merkle Tree is constructed as described above.
4. Root hash of this tree (left) is merged with metadata descriptor blake3 hash (right); this is metadata digest.

Product of concatenation of protocol version number with resulting metadata digest MUST be included into Signed Extensions.

### Shortening

For shortening, an attempt to decode transaction completely using provided metadata is performed with the same algorithm that would be used on the cold side. All chunks are associated with their leaf indices. An example of this protocol is proposed in [metadata-shortener](https://github.com/Alzymologist/metadata-shortener) that is based on [substrate-parser](https://github.com/Alzymologist/substrate-parser) decoding protocol; any decoding protocol could be used here as long as cold signer's design finds it appropriate for given security model.

### Transmission

Shortened metadata chunks MAY be trasmitted into cold device together with Merkle proof in its entirety or in parts, depending on memory capabilities of the cold device and it ability to reconstruct larger fraction of tree. This document does not specify the manner of transmission. The order of metadata chunks MAY be arbitrary, the only requirement is that indices of leaf nodes in Merkle tree corresponding to chunks MUST be communicated. Community MAY handle proof format standartization independently.

### Offline verification

The transmitted metadata chunks are hashed together with proof lemmas to obtain root that MAY be transmitted along with the rest of payload. Verification that the root transmitted with message matches with calculated root is optional; the transmitted root SHOULD NOT be used in signature, calculated root MUST be used; however, there is no mechanism to enforce this - it should be done during cold signers code audit.

### Chain verification

The root of metadata computed by cold device MAY be included into Signed Extensions; this way the transaction will pass as valid iff hash of metadata as seen by cold storage device is identical to consensus hash of metadata, ensuring fair signing protocol.

The Signed Extension representing metadata digest is a single byte representing both digest vaule inclusion and shortening protocol version; this MUST be included in Signed Extensions set. Depending on its value, a digest value is included as `additionalSigned` to signature computation according to following specification:

| signed extension value | digest value   | comment                            |
|------------------------|----------------|------------------------------------|
| `0x00`                 |                | digest is not included             |
| `0x01`                 | 32-byte digest | this represents protocol version 1 |
| `0x02` - `0xFF`        | *reserved*     | reserved for future use            |

## Drawbacks

### Increased transaction size

A 1-byte increase in transaction size due to signed extension value. Digest is not included in transferred transaction, only in signing process.

### Transition overhead

Some slightly out of spec systems might experience breaking changes as new content of signed extensions is added. It is important to note, that there is no real overhead in processing time nor complexity, as the metadata checking mechanism is voluntary. The only drawbacks are expected for tools that do not implement MetadataV14 self-descripting features.

## Testing, Security, and Privacy

The metadata shortening protocol should be extensively tested on all available examples of metadata before releasing changes to either metadata or shortener. Careful code review should be performed on shortener implementation code to ensure security. The main metadata tree would inevitably be constructed on runtime build which would also ensure correctness.

To be able to recall shortener protocol in case of vulnerability issues, a version byte is included.

## Performance, Ergonomics, and Compatibility

### Performance

This is negligibly short pessimization during build time on the chain side. Cold wallets performance would improve mostly as metadata validity mechanism that was taking most of effort in cold wallet support would become trivial.

### Ergonomics

The proposal was optimized for cold storage wallets usage with minimal impact on all other parts of the ecosystem

### Compatibility

Proposal in this form is not compatible with older toold that do not implement proper MetadataV14 self-descriptive features; those would have to be upgraded to include a new signed extensions field.

## Prior Art and References

This project was developed as Polkadot Treasury grant; relevant development links are located in [metadata-offline-project](https://github.com/Alzymologist/metadata-offline-project) repository.

## Unresolved Questions

1. Should hash inclusion bit be added to signed extensions?
2. How would polkadot-js handle the transition?
3. Where would non-rust tools like Ledger apps get shortened metadata content?

## Future Directions and Related Material

Changes to code of all cold signers to implement this mechanism SHOULD be done when this is enabled; non-cold signers may perform extra metadata check for better security. Ultimately, signing anything without decoding it with verifiable metadata should become discouraged in all situations where a decision-making mechanism is involved (that is, outside of fully automated blind signers like trade bots or staking rewards payout tools).
