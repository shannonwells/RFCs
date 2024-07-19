_This is an amended version of our original Design Document for the MSA Delegation feature that we implemented on the Frequency Polkadot parachain.
I have edited it for the purposes of this technical interview. I selected this particular Design Document because it most closely fits the prompt.
This was a 1-3 month project on which I was lead and there were several of us working on the MSA pallet implementing these features. I wrote much of the original content of this document which edited further as the features evolved.  Please also note the format of this Design Document was taken directly from examples from when I worked on the Filecoin project._

# Delegations

This document describes provable, revocable, permissioned delegation of specific actions to support the [Decentralized Social Network Protocol (DSNP)](https://spec.dsnp.org) on the Frequency Polkadot Parachain, referred to from now on as [DSNP/Frequency (DSNP over Frequency)](https://spec.dsnp.org/Frequency/Overview.html).
These actions are largely related to DSNP Id creation, DSNP Profile management, DSNP (Social) Graph, and DSNP Message announcements by the owner of the DSNP Id.

On the Frequency chain, a DSNP Id is mapped to a Message Source Account Id, or MSA Id, which is a pseudonymous id that is an integer, and the actions it can perform are authorized using a Polkadot `AccountId` cryptographic keypair.
This control can be purely wallet-based, so it does not require a token balance to exist on chain.
Non-token control of delegation actions uses cryptographic signatures of a payload, which is validated during the delegation transaction to prove authorization by both parties.  A failure to validate causes the transaction to fail.

The authorized actions occur on the Frequency chain and are performed by one or more Providers on the DNSP Id holder's behalf.
A Provider is a type of MSA Id holder who has registered as a Provider on the Frequency chain and has been authorized by Frequency's chain governance process.

## Table of Contents

- [Context and Scope](#context-and-scope)
- [Problem Statement](#problem-statement)
- [Goals and Non-Goals](#goals-and-non-goals)
- [Proposal](#proposal)
- [Benefits and Risks](#benefits-and-risks)
- [Alternatives and Rationale](#alternatives-and-rationale)
- [Glossary](#glossary)

## Context and Scope

This document describes how a delegation relationship between an MSA (identified by their MSA Id) and a Provider is created and validated on the Frequency chain, and outlines an API for its implementation.
Delegation to a Provider MSA may be used to perform tasks on behalf of another MSA.
Some examples of delegated actions and delegated permissions are given.
It's expected that the actions and permissions that are implemented for delegation will evolve as needed.

## Problem Statement

The primary motivation for delegation is to support end users of DSNP/Frequency, however, it is expected that delegation will be used in other ways.

Market research makes it clear that people are extremely reluctant to pay to use applications, particularly social networks.
Secondly, as we are attempting to build a decentralized social networking platform that does not privilege wealth, asking people to pay to use it is contradictory to our purpose.
This means there needs to be some way to onboard end users of social applications and relay their activity via DSNP/Frequency without charging them.
We are also building a platform that brings people's "switching cost" to zero so that people are not forced to use a platform that does not serve their needs, whatever they may be.  

Requiring the switching cost to be zero means that switching from one Provider to another must be free, easy, and transparent.
Next, the platform must also provide these delegations in a way that makes it extremely difficult to perform delegated actions without the person's permission.
Cryptography primitives and consensus mechanisms that are inherent to blockchains are a natural solution to this problem.

The use of authorized delegates, that is, Providers, enables the creation of accounts as well as processing and storing user messages and other data for the end users, paid for by a Provider, who can recoup these costs by other means (outside the scope of this Design Document).
The vast majority of the end user's actual activity will not reside on chain, however, Frequency needs to be able to coordinate the exchange of data, provide for discoverability, and to securely allow an account holder to manage their Delegates.
The Delegator --> Provider delegation is managed by assigning each account, called a Message Source Account or MSA, an ID number, called an Msa Id.

## Technical Goals and Non-Goals

Delegation, roughly speaking, must allow all Create, Read, Update and Delete (CRUD) operations by a Provider MSA to fulfill the purpose of performing actions on behalf of their Delegators.
Put another way, delegation must have the following properties:

- **Authorizable** - delegations can be authorized with specific permissions by MSAs.
- **Verifiable** - there is a way to check that Providers are doing things only when authorized and only what they are authorized to do.
- **Transparent** - delegations can be readable by anyone, in order to maximize opportunities to police Provider actions.
- **Changeable** - a Delegator can change Provider permissions to give MSAs control over what tasks are permitted to the Provider. https://github.com/frequency-chain/frequency/blob/main/designdocs/provider_permissions.md
- **Revocable** - a Delegator can withdraw permissions completely from the Provider.

### Non-Goals

- Doesn't cover handling the retirement of an MSA Id. _NB: Although retiring an MSA Id was not part of this original Design Document, it has been implemented._
- Delegated removal would allow removing any other Provider without substituting itself as the new Provider. Such an endpoint presents serious enough issues that it should be discussed and designed separately, if it's to be implemented at all.
- Does not specify what the permissions are nor the permissions data type.
- Does not specify a value for pallet constants, only when there should be one. These values should be determined by such factors as storage costs and performance.
- Does not include a "block/ban" feature for delegation, which is under discussion; the belief is that a Provider also ought to be able to permanently refuse service to a given MSA id, which further supports the idea of a mutually agreed upon relationship.

## Proposal

The proposed solution is to give someone the ability to create an on-chain MSA Id through an authorized Provider. One could also transparently authorize and manage their own Providers and permissions, either directly using a native token or through an explicitly authorized Provider. Additionally, we allow MSA ids` to be directly created using the native FRQCY token.

The method for creating and managing these delegations without requiring the MSA account holder to have a token balance is by cryptographically signed payloads that are validated on chain. The delegator signs a payload using their own, typically wallet-based keys, however the delegator key is a Polkadot `AccountId` and may have a token balance.

### API (extrinsics)
_NB: "extrinsics" is the Polkadot term for on-chain transactions._

- All names are placeholders and may be changed.
- All extrinsics must emit a unique event with all parameters for the call, unless otherwise specified.
- Errors in the extrinsics must have unique, reasonably-named error enums for each type of error for ease of debugging.
- "Owner only" means the caller must own the delegator MSA id.
- Events are not deposited for read-only extrinsic calls.  In addition to readable state storage, Events help fulfill the requirement of transparency.

#### create_sponsored_account_with_delegation

Creates a new MSA on behalf of a delegator and adds the MSA Id associated with the Origin (transaction signing public key) as its Provider.  

- Parameters:

    1. `add_provider_payload` - this is what the holder of delegator_key must sign and provide to the Provider beforehand.
        - `authorized_msa_id` - the Provider, of type `MessageSourceId`
    2. `delegator_key` - The authorizing cryptographic public key of the keypair used to create `proof`, belonging to the one doing the delegating
    3. `proof` - The signature of the hash of `add_provider_payload` by the delegator

- Events emitted on success:
    1. `MsaCreated`
        - `new_msa_id` - id of the newly created MSA
        - `key` - the delegator_key, a
    2. `DelegationGranted`
        - `delegator` - id of the newly created MSA
        - `Provider` - id of the MSA help by the Provider

#### revoke_delegation_by_provider

Supports the revocability requirement.  Provider revokes its relationship from the specified `delegator` in the parameters. This function allows a Provider to control access to its services.

- Parameters:

    1. `delegator` - the MSA Id of the delegator (generally an end user)
- Restrictions: **Owner only** - this transaction is restricted by default to the owner of the Provider MSA Id, as the relationship to be revoked is looked up based on the transaction signer's public key.
- Events on success:
    1. `ProviderRevokedDelegation`
        - `provider` - MSA Id held by the Provider
        - `delegator` - MSA Id held by the delegator 

#### create

Directly creates an MSA for the origin (caller) without a Provider.
This is a signed call directly from the caller, so the owner of the new MSA pays the fees for its creation.

- Events on success:
    1. `DelegatorRevokedDelegation`
        - `msa_id` - id of the newly created MSA

#### revoke_delegation_by_delegator

Supports the revocability requirement. A delegator removes its relationship from a Provider.
This is a signed call directly from the delegator's MSA.
This call is exempted from transaction fees.

- Parameters:

    1. `provider_msa_id` - id of the MSA held by the Provider

- Restrictions: **Owner only** - this transaction is restricted to the owner of the Delegator MSA Id, since the relationship to be revoked is looked up based on the transaction signer's public key.

- Event: `DelegateRemoved`
    1. `delegator` - id of the MSA held by the delegator
    2. `Provider` - id of the MSA held by the Provider

### Custom RPC endpoints
Custom RPC endpoints on a Polkadot parachain are excecuted off chain and perform read-only operations.

#### get_msa_keys(msa_id)

Supports the verifiability requirement. Retrieve a list of public keys of up to `MaxPublicKeysPerMsa` size for the provided MSA id, or an empty list if the MSA id does not exist.

- Parameters:
    1. `msa_id`: the MSA id of which associated keys are to be retrieved

#### check_delegations

Supports the verifiability requirement.
Validate that a Provider can delegate for a list of MSA ids. 
This call is intended for validating messages referenced by a message pallet `add_message` transaction for a [DSNP Batch Announcement](https://spec.dsnp.org/DSNP/BatchPublications.html), so this function would be an all-or-nothing check.
If the permission stored for a given MSA id exceeds the parameter, the check for that MSA id passes.
For example, if a Provider has _all_ permissions set, then querying for a subset of permissions will pass.
Verify that the provided Provider `provider_msa_id` is a Provider of the delegator, and has the given permission value.
Returns `Ok(true)` if Provider is valid, `Ok(false)` if not.
Throws an Error enum indicating if either Provider or delegator does not exist.

- Parameters:
    1. `delegator_msa_ids`: a list of Delegator ids possible delegators
    2. `provider_msa_id`: the ProviderId to verify

### Storage
This section describes any necessary on-chain state storage for this feature. State storage of Delegations supports the changeability requirement. State storage is also readable for free and by anyone with access to a node endpoint, which supports the transparency requirement.

- Delegations are stored as a double-key map of Delegator MSA Id + Provider MSA Id. The data stored is a bounded BTree of `SchemaId` mapped to a revocation block for that relationship.  A revocation block of 0 means the delegation is active for that `SchemaId`.
- The entire delegation can be revoked by setting `blocked_at` to the block number when it is revoked.  Alternatively, individual schemas can be revoked by doing the same for the Schema Id entry in `schema_permissions`

```rust
pub struct Delegation<SchemaId, BlockNumber, MaxSchemaGrantsPerDelegation>
where
	MaxSchemaGrantsPerDelegation: Get<u32>,
{
	/// Block number the grant will be revoked.
	pub revoked_at: BlockNumber,
	/// Schemas that the provider is allowed to use for a delegated message.
	pub schema_permissions: BoundedBTreeMap<SchemaId, BlockNumber, MaxSchemaGrantsPerDelegation>,
}

pub(super) type DelegatorAndProviderToDelegation<T: Config> = StorageDoubleMap<
    _,
    Twox64Concat,
    Delegator,
    Twox64Concat,
    Provider,
    Delegation<SchemaId, BlockNumberFor::<T>, T::MaxSchemaGrantsPerDelegation>,
    OptionQuery,
>;
```

## Benefits and Risks

As stated earlier, one of the primary intended benefits of delegation is to allow feeless account creation and messaging.

There is a risk of abuse with delegation of messages, since this makes it possible for a Provider to, for example, modify the a user's messages before batching them. The message sender would have to be caught and the person must react after the fact, instead of the message sender being technologically prevented from this type of dishonesty.

There is another risk of abuse for any other type of delegated call if the wallet that provides the signing capability does not make it very clear to the user what they're signing.

## Alternatives and Rationale

### MSA holder pays for existential deposit

We briefly discussed the possibility of requiring a small token deposit to create their account. We decided against this option because:

1. As mentioned above, people don't expect and won't pay to use social media.
2. Onboarding would be a problem; even if they did want to pay even a small amount, getting people access to a token is tremendously difficult at this time, requiring unacceptable tradeoffs.
3. We would be unable to serve people who are unbanked and/or don't have access to crypto trading platforms.

### dApp Developer pays for existential deposit

One alternative to allow for account creation at no cost to the end user was to have the dApp developer MSA send an existential deposit to the account to create it.
We decided against this option for a number of reasons:

1. It could create a potential for abuse and token loss by those creating numerous fake accounts and then removing the delegation (which is a free transaction).
2. This creates unpredictable capital expenses just for user acquisition.  
3. Assuming typical user dropoff rates, creating a user account with an existential deposit could have a dramatic effect on token inflation as well as putting a burden on node storage.  While unused accounts could theoretically be reaped and the token burned or returned to storage, such solutions could undermine trust in the Frequency network.

### MSA pays to send messages, with no possibility of delegating

An alternative for delegating messaging capabilities was to have each MSA pay for their own messages.
This was ruled out as the sole solution because:

1. The average person can't or won't pay to use social media.  See reasons above for existential deposit.
2. Making end users pay to send messages would require people to sign transactions every time they make any updates — all posts, all reactions, all replies, all profile changes, all follows/unfollows, etc. This would create much friction and annoyance, likely resulting in loss of users.

The proposed design includes direct pay endpoints, so even if an MSA did not want to trust a Provider, they could still pay for all of their messages, assuming they have some way of obtaining FRQCY token.

### Permissioned delegation is an industry standard

Furthermore, permissioned delegation via verifiable strong cryptographic signature is a well-known and tested feature in smart contracts of distributed blockchain-based applications.

## Deferred features

### An "effective block range" for providers

Including an effective block range in the Provider storage data would allow providers to be expired, not just removed. A block range could better support features like Tombstone, blocking, and retiring an MSA id. Effective block range is deferred because those features have not been fully defined. (_Tombstone messages are now defined as DSNP Announcements, and MSA Id retirement on Frequency is now implemented_)

### add_provider(delegator, Provider, permissions)
_This still cannot be done; we've adopted the philosophy that all relationships should be mutual and opt-in, so adding a provider still requires both signatures._

Directly adding a Provider, with or without a Provider's permission, is not to be implemented at this time. The original use case was for a potential wallet app to support browsing and adding providers. Adding/replacing a Provider for an existing account with an MSA id could still be done using the delegated methods, `add_self_as_delegate` or `replace_delegate_with_self`. A direct add brought up concerns about potential risks of adding a Provider without the Provider's knowledge. For example, if the Provider has removed the delegator for legitimate reasons, such as if the end user violated the Provider's Terms of Service, then the Provider ought to be able to prevent them from adding the Provider again just by paying for it.

## Glossary

- **Provider**: An MSA that has been granted specific permissions by its Delegator. A company or individual operating an on-chain Provider MSA in order to post Frequency transactions on behalf of other MSAs.  A Provider must be registered and approved via Frequency chain proposal before it can receive any delegations.
- **Delegator**: An MSA that has granted specific permissions to a Provider.
- **MSA**: Message Source Account refers to the DSNP Id that lives on the Frequency Chain as an integer MSA Id, plus one or more cryptographic key pairs that are associated with it.  The keys may or may not have a token account.
- **Public Key**: A 32-byte (u256) number that is used to refer to an on-chain MSA and verify signatures. It is one of the keys of an MSA key pair
- **MsaId**: An 8-byte (u64) number used as a lookup and storage key for delegations, among other things
- **End user**: Also referred to as MSA holder. Groups or individuals that control an MSA that is not a Provider MSA.

You may see our [Design Documents in the Frequency repository](https://github.com/frequency-chain/frequency/tree/main/designdocs).  For a more recent example, please see the [Provider Boost Implementation](https://github.com/frequency-chain/frequency/blob/feat/capacity-staking-rewards-impl/designdocs/provider_boosting_implementation.md) Design Document which I also wrote (both the implementation and [Economic Model docs](https://github.com/frequency-chain/frequency/blob/feat/capacity-staking-rewards-impl/designdocs/provider_boosting_economic_model.md)). However I did not select these, because I am the only person doing the implementation and it has been longer than a 3 month project.

