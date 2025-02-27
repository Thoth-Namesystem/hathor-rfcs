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
   - A one-time fee is required for registration (for now, but later we plan to introduce renewal anual fees)
   - Names can be between 3-32 characters

2. **Resolution**
   - Registered names can be resolved to their associated addresses
   - Applications can integrate with the service to resolve names to addresses

3. **Management**
   - Name owners can update the resolving address
   - Ownership can be transferred to another address

## Usage Example

Alice wants to register "alice.htr" for her wallet:

1. Alice calls the `create_name` function with the name "alice" for the nano contract with domain "htr"
2. She pays the required registration fee
3. The name is now registered and resolves to her address
4. Later, she can:
   - Change the resolving address to point to a different wallet
   - Transfer ownership to another user if desired

## User Benefits

1. **Simplified Transactions**
   - Send tokens to easy-to-remember names
   - Avoid errors from mistyped addresses

2. **Digital Identity**
   - Establish a recognizable on-chain identity
   - Build reputation around a consistent identifier

3. **Flexibility**
   - Update resolving addresses without changing identity
   - Transfer ownership if needed

# Reference-level explanation

## Contract State Variables

```python
class ThothNamer(Blueprint):
    domain: str                      # Base domain (e.g., "htr")
    names: Dict[str, dict[str, Address]]  # Mapping of names to owner/resolving addresses
    dev_address: Address             # Developer address for receiving fees
    fee: Amount                      # Fee for registering a name
    total_fee: Amount                # Total fees collected
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
def initialize(self, ctx: Context, domain: str, fee: Amount) -> None:
```

Initializes the name service by:
- Setting the base domain name 
- Establishing the registration fee
- Setting the developer address to the contract creator
- Validating inputs (domain not empty, fee positive)

### create_name

```python
@public
def create_name(self, ctx: Context, name: str) -> None:
```

Registers a new name with the following workflow:
- Validates the name format
- Checks that the name doesn't already exist
- Verifies sufficient fee payment
- Maps the name to the caller's address as both owner and resolving address
- Updates total fees collected

### change_fee

```python
@public
def change_fee(self, ctx: Context, fee: Amount) -> None:
```

Allows the developer to update the registration fee:
- Verifies caller is the developer
- Ensures the new fee is a positive value
- Updates the fee amount

### change_dev_address

```python
@public
def change_dev_address(self, ctx: Context, new_dev_address: Address) -> None:
```

Enables the current developer to transfer control:
- Verifies caller is the current developer
- Updates the developer address to the new address

### change_name_owner

```python
@public
def change_name_owner(self, ctx: Context, name: str, new_owner_address: Address) -> None:
```

Transfers name ownership to a new address:
- Verifies the name exists
- Ensures caller is the current owner
- Updates owner information while preserving resolving address

### change_resolving_address

```python
@public
def change_resolving_address(self, ctx: Context, name: str, new_resolving_address: Address) -> None:
```

Updates where a name resolves to:
- Verifies the name exists
- Ensures caller is the owner
- Updates resolving address while preserving ownership information

## View Methods

### resolve_name

```python
@view
def resolve_name(self, name: str) -> str:
```

Retrieves the address associated with a name:
- Verifies the name exists
- Returns the resolving address in Base58 format

### validate_name

```python
@view
def validate_name(self, name: str) -> bool:
```

Implements name validation rules:
- Length between 3-32 characters
- Only lowercase letters, numbers, and hyphens allowed
- No hyphens at beginning or end
- Non-empty string

### check_name_existence

```python
@view
def check_name_existence(self, name: str) -> bool:
```

Checks if a name is already registered:
- Returns true if name exists in registry
- Returns false otherwise

### get_name_owner

```python
@view
def get_name_owner(self, name: str) -> Address:
```

Retrieves the owner of a registered name:
- Verifies the name exists
- Returns owner's address in Base58 format

### get_dev_address

```python
@view
def get_dev_address(self) -> Address:
```

Returns the current developer address in Base58 format.

### get_contract_domain

```python
@view
def get_contract_domain(self) -> str:
```

Returns the base domain for the name service.

## Helper Methods

### _get_action

```python
def _get_action(self, ctx: Context) -> NCAction:
```

Internal utility for processing transactions:
- Ensures only one action is present
- Restricts withdrawals to developer
- Validates token type is HTR
- Returns the action for processing

### _update_resolving_address and _update_owner_address

Internal utilities that handle updates to the names dictionary.

## Name Data Structure

```python
name_entry = {
    "owner_address": Address,      # The address that controls the name
    "resolving_address": Address   # The address that the name resolves to
}
```

# Drawbacks

1. No built-in mechanism for name expiration or renewals
2. Potential for name squatting without a progressive fee structure
3. Limited metadata storage capability

# Rationale and alternatives

The chosen design prioritizes:

1. Simplicity in implementation and user interaction
2. Clear ownership and management rights
3. Minimal state storage requirements
4. Integration with existing Hathor features

Alternatives considered:

1. ENS-style hierarchical domains
   - More complex implementation
   - Higher gas costs
   - More flexible namespace

2. Time-limited registration with renewals
   - Requires more complex time tracking
   - Higher maintenance burden on users
   - Potentially more efficient use of namespace

3. Auction-based name allocation
   - More complex implementation
   - Potentially higher fees for popular names
   - Fairer distribution of valuable names

# Prior art

ThothNamer draws inspiration from several existing name services while adapting for Hathor's specific architecture:

1. Ethereum Name Service (ENS)
   - Simplified structure compared to ENS's hierarchical system
   - Single domain vs multiple TLDs
   - Similar ownership transfer mechanism

2. Handshake
   - Simplified approach compared to Handshake's complete DNS alternative
   - Focus on application-specific names rather than internet naming

3. Namecoin
   - Similar concept of blockchain-based name registration
   - More specific focus on Hathor addresses vs Namecoin's broader scope

# Unresolved questions

1. How should disputes over trademarked names be handled?
2. Is there a need for a name expiration mechanism to prevent dormant addresses?
3. Should fees be adjustable based on name length or desirability?
4. How will applications integrate with the name resolution system?

# Future possibilities

1. Integration with off-chain identity verification
2. Adding metadata support for avatars, descriptions, and social links
3. Implementing name expirations and renewals
4. Creating a marketplace for name trading
5. Adding support for subdomains (e.g., payments.alice.htr)
6. Governance mechanism for contract parameter adjustments
