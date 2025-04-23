# RandomTON Service Documentation

## Overview
RandomTON is a service on the TON Blockchain, enabling smart contracts to request cryptographically secure randomness via Verifiable Random Function (VRF). The system consists of three core components:

1. **Factory Contract**: Deploys and manages verifier contracts.
2. **Verifier Contract**: Generates and verifies randomness (confidential implementation).
3. **Oracle Network**: Off-chain nodes generating cryptographically secure randomness and proofs.

```plaintext
         +----------------+        +-------------------+      +-------------------+
         | Factory        |        |    Verifier       |      | Oracle Network    |
         | (Deploys       +------->   (Validates       <------+ (Generates        |
         |  Verifiers)    |        |       Proofs)     |      |  Randomness &     |
         +------- ^-------+        +---------^---------+      |  Proofs)          |
                  |                          |                +-------------------+
                  |                          |
         +--------+----------+     +---------v---------+
         | Client Contract   <-----> (Handles Requests |
         | (Requests         |     |  & Responses)     |
         |  Randomness)      |     +-------------------+
         +-------------------+
```

```plaintext
Client Contract           Verifier Contract           Oracle Network
     |                             |                        |
     |──1. Request Randomness─────>|                        |
     |                             |──2. Event: New Request─> 
     |                             |<──3. Signed Randomness──
     |<──4. Validated Result───────|                        |
```

---

## Factory contract address.

Right now RandomTON factory contract is deployed at ''.

## Pricing.

For detailed pricing models, transaction fees, and subscription tiers, refer to our dedicated **[RandomTON Pricing Guide](pricing.md)**. Below is a high-level summary:

| Category                  | Description                                                                 |
|---------------------------|-----------------------------------------------------------------------------|
| **Deployment Fees**       | One-time cost for deploying a verifier contract via the Factory.            |
| **Pay-As-You-Go**         | Per-request fee for ad-hoc randomness generation.                           |
| **Subscriptions**         | Tiered plans (Basic/Professional/Enterprise) for high-volume use cases.     |
| **Freemium Programs**     | Free tier with 100 monthly requests for early-stage projects/experiments.   |
| **Grants**                | Sponsorship for public goods, open-source projects, and ecosystem builders. |

[**View Full Pricing Details →**](pricing.md)

---

### Key Variables in Code
The client contract uses these predefined fee constants (adjustable via governance):
```tact
const randomTonDeploymentFee: Int = ton("5");
const randomTonPayAsYouGoFee: Int = ton("0.3");
const randomTonBasicSubscriptionTierFee: Int = ton("27.5");

## Integration Guide

### 1. **Deploy Your Client Contract**
   - Inherit the `RandomTON` trait and initialize critical variables:
     ```tact
     contract TestClientContract with Deployable, Ownable, RandomTON {
         init() {
             self.owner = sender();
             self.randomTonContract = null; // Will be set after registration
             self.randomTonSeed = null;      // Seed is populated later
             self.randomTonIsHandlingRegistration = false;
         }
     }
     ```
   - **Note**: Ensure your contract implements `randomTonRegistrationGuard()` for access control.
     ```tact
     override fun randomTonRegistrationGuard() {
         self.requireOwner();
     }
     ```

### Step 2: Register your own RandomTON Verifier
Deploy a verifier for your contract by sending a registration request to the factory:
Send `RandomTONRegister` message to your contract passing the `randomTonFactoryContract` address via message body.

### Step 3: Request Randomness
Trigger a randomness request. Choose between pay-as-you-go or subscription:
```tact
// Pay-as-you-go
receive( "Request Random Numbers" )
{
    self.requireOwner();
    self.randomTonRequestRandomness(5 /* iterations */, false /* subscription */);
}

// Subscription-based (requires prior subscription purchase)
receive( "Request Subscription Randomness" )
{
    self.requireOwner();
    self.randomTonRequestRandomness(5 /* iterations */, true /* subscription */);
}
```

### Step 4: Handle Randomness Response
Override `randomTonHandleRandomness` to process results:
```tact
override fun randomTonHandleRandomness(randomSeed: Int, iterations: Int)
{
    let numberOfPlayers = 20;
    repeat (iterations)
    {
        let result = self.randomTonRandomize(randomSeed, numberOfPlayers);
        randomSeed = result.newSeed;
        // Use result.randomNumber (e.g., select winners)
    }
}
```

### Step 5: Manage Subscriptions (Optional)
Purchase a subscription tier if needed:
```tact
receive( "Buy Basic Subscription" )
{
    self.requireOwner();
    self.randomTonPurchaseBasicSubscription();
}
```

---

## Key Restrictions
**Authorization**:
   - Only the contract owner can trigger requests, subscriptions, or withdrawals.
   - The `randomTonRegistrationGuard()` must be overridden to enforce access control.

---

## Example Flow
1. **Deploy Client Contract**: Inherit `RandomTON` and set up ownership.
2. **Register with Factory**: Pay the deployment fee to get a dedicated verifier.
3. **Request Randomness**: Choose a payment model (pay-as-you-go or subscription).
4. **Process Results**: Use the callback to handle randomness securely.

---

## Troubleshooting
- **Failed Requests**: Check if:
  - `randomTonContract` is set (completed registration).
  - Correct fees are attached (use `randomTonMinTransactionValue` as a baseline).
  - Subscription is active (if using `isSubscription: true`).
- **Seed Errors**: If randomness repeats, ensure you update `randomTonSeed` with `newSeed` after each use.
