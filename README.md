# Payment Channel Smart Contract Documentation

![Clarity Language](https://img.shields.io/badge/Language-Clarity-8A2BE2)

## Overview

This smart contract implements a **bi-directional payment channel system** enabling trustless off-chain transactions between two parties with secure on-chain settlement. The contract supports both cooperative closures (mutual agreement) and unilateral closures with dispute resolution mechanisms, providing flexibility while maintaining security.

## Key Features

1. **Off-Chain Efficiency** - Conduct unlimited transactions without blockchain interaction
2. **Dispute Resolution** - 1008-block (~7 day) challenge period for unilateral closures
3. **Signature Verification** - Cryptographic proof enforcement for transaction validity
4. **Fund Security** - Non-custodial design with escrowed balances
5. **Multi-Funding** - Ability to add funds to open channels
6. **State Tracking** - Channel nonce management for transaction sequencing

## Technical Specification

### Data Structure

```clarity
(define-map payment-channels
  {
    channel-id: (buff 32),      // 32-byte channel identifier
    participant-a: principal,   // Initiating party
    participant-b: principal    // Counterparty
  }
  {
    total-deposited: uint,     // Combined channel balance
    balance-a: uint,           // Participant A's current allocation
    balance-b: uint,           // Participant B's current allocation
    is-open: bool,             // Channel status flag
    dispute-deadline: uint,    // Block height for dispute resolution
    nonce: uint                // State version counter
  }
)
```

### Error Codes

| Code   | Description                     |
| ------ | ------------------------------- |
| `u100` | Unauthorized operation          |
| `u101` | Channel already exists          |
| `u102` | Channel not found               |
| `u103` | Insufficient channel funds      |
| `u104` | Invalid cryptographic signature |
| `u105` | Operation on closed channel     |
| `u106` | Dispute period active           |
| `u107` | Invalid input parameters        |

## Core Functions

### 1. Channel Creation

```clarity
(create-channel channel-id participant-b initial-deposit)
```

- **Parameters**

  - `channel-id`: 32-byte unique identifier
  - `participant-b`: STX address of counterparty
  - `initial-deposit`: Initial funding amount (micro-STX)

- **Requirements**

  - Unique channel ID
  - Non-zero deposit
  - Distinct participants

- **Effects**
  - Locks initial funds in escrow
  - Initializes channel state
  - Sets balances (A: deposit, B: 0)

### 2. Channel Funding

```clarity
(fund-channel channel-id participant-b additional-funds)
```

- **Parameters**

  - Existing channel identifiers
  - Additional STX amount

- **Requirements**

  - Open channel status
  - Valid deposit amount

- **Effects**
  - Increases `total-deposited` and `balance-a`

### 3. Cooperative Closure

```clarity
(close-channel-cooperative channel-id participant-b balance-a balance-b signature-a signature-b)
```

- **Parameters**

  - Final balance allocations
  - Dual signatures (both participants)

- **Verification**

  1. Signature validity check
  2. Sum conservation: `balance-a + balance-b == total-deposited`

- **Execution**
  - Immediate fund distribution
  - Channel state closure

### 4. Unilateral Closure

```clarity
(initiate-unilateral-close channel-id participant-b proposed-balance-a proposed-balance-b signature)
```

- **Parameters**

  - Proposed balances
  - Initiator's signature

- **Process**
  1. Signature verification
  2. Balance validation
  3. Sets 1008-block dispute period

### 5. Dispute Resolution

```clarity
(resolve-unilateral-close channel-id participant-b)
```

- **Requirements**

  - Dispute period expiration
  - Valid channel state

- **Execution**
  - Finalizes proposed balances
  - Closes channel

## Usage Examples

### Creating a Channel

```clarity
(create-channel 0x1234abcd... 'SP3BZ... 5000000)
```

- Creates channel with 5 STX deposit

### Cooperative Closure

```clarity
(close-channel-cooperative
  0x1234abcd...
  'SP3BZ...
  3000000
  2000000
  0xabcd...
  0xef01...)
```

- Distributes 3 STX to A, 2 STX to B

### Unilateral Closure

```clarity
(initiate-unilateral-close 0x1234abcd... 'SP3BZ... 4000000 1000000 0xabcd...)
```

- Proposes 4/1 STX split, starts 1-week dispute period

## Security Considerations

### Signature Requirements

- Cooperative closures require dual signatures
- Unilateral initiations need participant's signature
- **Note:** Current implementation uses simplified signature verification - production deployment requires proper cryptographic verification

### Fund Safeguards

- Escrow model prevents direct fund access
- Balance conservation enforced at settlement
- Dispute period prevents premature closure

### Access Control

- Channel operations restricted to participants
- Emergency withdrawal limited to contract owner

## Emergency Procedures

### Owner Withdrawal

```clarity
(emergency-withdraw)
```

- **Restrictions**
  - Contract owner only
  - Full balance withdrawal
- **Use Case**
  - Protocol migration
  - Critical vulnerabilities

## Testing & Audit Recommendations

1. Comprehensive signature verification tests
2. Edge case testing for:
   - Over/underflow scenarios
   - Concurrent operations
   - Expired dispute resolutions
3. Third-party security audit
4. Testnet deployment before mainnet launch

## Limitations

- Current signature verification implementation is simplified
- No built-in transaction history tracking
- Fixed dispute period duration
