- Feature Name: thoth_namer
- Start Date: 2025-01-27
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Authors: Blaze Karma blaze.karma01@gmail.com, Simon Laplace s1m0n.l4pl4c3@proton.me and Guyana thothguyana@gmail.com
# Summary

ThothNamer is a name service blueprint for the Hathor Network that enables the registration and management of domain names. The service allows users to register unique names under a specified domain, associate them with resolving addresses, and transfer ownership. It implements a fee-based registration system with domain name validation and provides a decentralized alternative to traditional DNS.

# Motivation

Blockchain addresses are cryptographically secure but difficult for humans to remember and use. This creates several challenges:

1. Poor user experience when sending tokens to lengthy hexadecimal addresses
2. High risk of errors when manually entering addresses
3. Difficulty in building recognizable on-chain identities
4. Limited accessibility for non-technical users

ThothNamer addresses these challenges by:

- Providing human-readable name aliases for blockchain addresses
- Creating a consistent naming standard across the Hathor ecosystem
- Enabling ownership and management of digital identities
- Simplifying the process of sending assets to known recipients

The expected outcome is improved usability across Hathor applications, reduced transaction errors, and enhanced identity management within the ecosystem.

# Guide-level explanation

## Service Overview

ThothNamer is a name service that allows users to register and manage human-readable names on the Hathor Network. Users can register a name (e.g., "alice") under a specific domain (e.g., "htr"), creating an address like "alice.htr" that resolves to their Hathor wallet address.

## Core Functionality

1. **Name Registration**
   - Users can register unique names (subject to validation rules)
   - Registration fees vary depending on name length and duration (years)
   - Names with 1 or more characters are valid, but short names (1–4 chars) are more expensive

2. **Resolution**
   - Registered names can be resolved to their associated addresses
   - Applications can integrate with the service to resolve names to addresses

3. **Management**
   - Name owners can update the resolving address
   - Ownership can be transferred to another address
   - TTL (time to live) is tracked as an expiration date (YYYY-MM-DD)
   - Names can be renewed for additional years

## Usage Example

Alice wants to register "alice.htr" for her wallet for 2 years:

1. Alice calls the `create_name` function with the name "alice" and duration `2`
2. The contract calculates the price based on duration and name length
3. She pays the required registration fee
4. The name is now registered and resolves to her address, expiring in 2 years
5. Later, she can:
   - Change the resolving address to a different wallet
   - Renew the name by paying for additional years
   - Transfer ownership to another user

## User Benefits

1. **Simplified Transactions**
   - Send tokens to easy-to-remember names
   - Avoid errors from mistyped addresses

2. **Digital Identity**
   - Establish a recognizable on-chain identity
   - Build reputation around a consistent identifier

3. **Flexibility**
   - Update resolving addresses without changing identity
   - Renew or transfer ownership as needed

# Reference-level explanation

## Contract State Variables

```python
class ThothNamer(Blueprint):
    domain: str                      # Base domain (e.g., "htr")
    names: Dict[str, dict[str, Any]] # Mapping of names to owner/resolving/ttl
    dev_address: Address             # Developer address for receiving fees
    fee: Amount                      # Base fee per year
    total_fee: Amount                # Total fees collected
    short_name_multiplier: int      # Fee multiplier for short names (1–4 chars)
```

## Custom Exceptions

```python
class NameNotFound(NCFail): pass
class NameAlreadyExists(NCFail): pass
class NotAuthorized(NCFail): pass
class InvalidNameFormat(NCFail): pass
class WithdrawalNotAllowed(NCFail): pass
class DepositNotAllowed(NCFail): pass
class InsufficientBalance(NCFail): pass
class InvalidFee(NCFail): pass
class InvalidDomain(NCFail): pass
class TooManyActions(NCFail): pass
class InvalidToken(NCFail): pass
```

## Public Methods

### initialize

```python
@public
def initialize(self, ctx: Context, domain: str, fee: Amount, short_name_multiplier: int) -> None:
```

Initializes the name service:
- Sets the base domain name and developer address
- Establishes the base registration fee
- Sets the short name price multiplier

### create_name

```python
@public
def create_name(self, ctx: Context, name: str, bought_time: int) -> None:
```

Registers a new name:
- Validates the name format
- Checks if name already exists
- Calculates expiration date from `ctx.now` and `bought_time`
- Calculates fee based on name length and duration
- Stores owner, resolver, and TTL (`datetime.date`)
- Updates total fees collected

### renew_name

```python
@public
def renew_name(self, ctx: Context, name: str, extra_years: int) -> None:
```

Renews the TTL of an existing name:
- Validates ownership
- Adds extra years to expiration
- Applies the appropriate fee again

### change_fee

```python
@public
def change_fee(self, ctx: Context, fee: Amount) -> None:
```

Allows the developer to update the base registration fee.

### change_short_name_multiplier

```python
@public
def change_short_name_multiplier(self, ctx: Context, multiplier: int) -> None:
```

Allows the developer to update the price multiplier for short names.

### change_dev_address

```python
@public
def change_dev_address(self, ctx: Context, new_dev_address: Address) -> None:
```

Allows developer to transfer admin rights to another address.

### change_name_owner

```python
@public
def change_name_owner(self, ctx: Context, name: str, new_owner_address: Address) -> None:
```

Transfers ownership of a name to another address.

### change_resolving_address

```python
@public
def change_resolving_address(self, ctx: Context, name: str, new_resolving_address: Address) -> None:
```

Updates the resolving address for a registered name.

## View Methods

### resolve_name

```python
@view
def resolve_name(self, name: str) -> str:
```

Returns the resolving address for a given name.

### validate_name

```python
@view
def validate_name(self, name: str) -> bool:
```

Validates name format:
- Must be non-empty
- Cannot start or end with hyphen `-`
- Only lowercase letters, numbers, and hyphens are allowed

### check_name_existence

```python
@view
def check_name_existence(self, name: str) -> bool:
```

Returns whether the name exists and is not expired.

### check_name_expired

```python
@view
def check_name_expired(self, name: str) -> bool:
```

Returns whether the name’s TTL has passed.

### get_name_owner

```python
@view
def get_name_owner(self, name: str) -> Address:
```

Returns the current owner of the name.

### get_name_ttl

```python
@view
def get_name_ttl(self, name: str) -> str:
```

Returns the name’s expiration date in ISO format (`YYYY-MM-DD`).

### get_dev_address

```python
@view
def get_dev_address(self) -> Address:
```

Returns the developer address.

### get_contract_domain

```python
@view
def get_contract_domain(self) -> str:
```

Returns the current contract domain.

## Helper Methods

### _get_action

```python
def _get_action(self, ctx: Context) -> NCAction:
```

Validates and returns the payment action.

### _calculate_fee

```python
def _calculate_fee(self, name: str, years: int) -> Amount:
```

Calculates total price:
- `fee * years`
- If name has 1–4 characters, apply `short_name_multiplier`

### _calculate_expiration_date

```python
def _calculate_expiration_date(self, timestamp: int, years: int) -> date:
```

Converts timestamp and years into a `datetime.date` expiration.

## Name Data Structure

```python
name_entry = {
    "owner_address": Address,
    "resolving_address": Address,
    "ttl": date  # Expiration date
}
```

# Drawbacks

1. No off-chain dispute resolution system for trademark conflicts
2. Name squatting still possible despite pricing model
3. No automatic purging of expired names (manual renewal only)

# Rationale and alternatives

The design prioritizes:

1. Simplicity and minimal gas usage
2. Full ownership control and upgradability
3. Customizability of pricing and expiration terms

Alternatives considered:

- ENS-style subdomain hierarchies (more complex, higher cost)
- Auction-based name bidding
- Flat-fee renewals with grace periods

# Prior art

- **Ethereum Name Service (ENS)** – inspired flexible ownership/resolver separation
- **Namecoin** – early on-chain identity concept
- **Handshake** – focus on full DNS decentralization (not needed here)

# Unresolved questions

1. Should expired names be automatically removed or remain inert?
2. Should there be a public grace period before full expiration?
3. Could additional metadata be stored with names (e.g., social links)?

# Future possibilities

1. Name marketplace and trading support
2. Optional off-chain identity linkage and proof of personhood
3. Expiration notifications or renewals via oracles
4. Integration into wallets and dApps for native name resolution
5. DAO governance for fee and multiplier adjustments
6. Metadata fields for profile pictures, links, and reputation
