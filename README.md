# RandomTON VRF Service Documentation

## Overview
RandomTON is a Verifiable Random Function (VRF) service on the TON Blockchain, enabling smart contracts to request cryptographically secure randomness. The system consists of three core components:

1. **Factory Contract**: Deploys and manages verifier contracts.
2. **Verifier Contract**: Generates and verifies randomness (confidential implementation).
3. **Client Contract**: Integrates with the VRF service to request and use randomness.

```plaintext
         +----------------+       +-------------------+
         | Factory        |       | Verifier          |
         | (Deploys       +------->+ (Generates        |
         |  Verifiers)     |       |  Randomness)      |
         +-------+--------+       +---------+---------+
                  |                          |
                  |                          |
         +--------v---------+      +---------v---------+
         | Client Contract  <------+ (Handles Requests |
         | (Requests         |      |  & Responses)     |
         |  Randomness)      |      +-------------------+
         +-------------------+
```

---

## Integration Guide

### Step 1: Inherit the `RandomTON` Trait
Include the `RandomTON` trait in your contract to access VRF functionality:
```tact
import "./lib/random_ton";

contract YourContract with Deployable, Ownable, RandomTON {
    // Your logic here
}
```

### Step 2: Register with the VRF Service
Deploy a verifier for your contract by sending a registration request to the factory:
```tact
receive( "Register with VRF" )
{
    self.requireOwner();
    send(
        SendParameters
        {
            to    : factoryAddress, // Replace with factory address
            value : self.randomTonDeploymentFee,
            body  : RandomTONRegister{ randomTonFactoryContract: factoryAddress }.toCell(),
            mode  : SendPayFwdFeesSeparately
        }
    );
}
```

### Step 3: Request Randomness
Trigger a randomness request. Choose between pay-as-you-go or subscription:
```tact
// Pay-as-you-go
receive( "Request Random Numbers" )
{
    self.requireOwner();
    self.randomTonRequestRandomness(iterations = 5, isSubscription = false);
}

// Subscription-based (requires prior subscription purchase)
receive( "Request Subscription Randomness" )
{
    self.requireOwner();
    self.randomTonRequestRandomness(iterations = 5, isSubscription = true);
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
1. **Authorization**:
   - Only the contract owner can trigger requests, subscriptions, or withdrawals.
   - The `randomTonRegistrationGuard()` must be overridden to enforce access control.

2. **Fees**:
   - Ensure correct fees are attached to transactions (e.g., `randomTonDeploymentFee` for registration).
   - Failed payments will revert transactions.

3. **State Management**:
   - Always check `randomTonContract != null` before interacting with the verifier.
   - Use `randomTonIsHandlingRegistration` to avoid reentrancy during registration.

4. **Seed Handling**:
   - The seed is reset after use. Store it persistently if needed for future logic.

---

## Example Flow
1. **Deploy Client Contract**: Inherit `RandomTON` and set up ownership.
2. **Register with Factory**: Pay the deployment fee to get a dedicated verifier.
3. **Request Randomness**: Choose a payment model (pay-as-you-go or subscription).
4. **Process Results**: Use the callback to handle randomness securely.

---

## Example Flow (Detailed)

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

---

### 2. **Register with the Factory Contract**
   - Send a `RandomTONRegister` message to the factory to deploy a dedicated verifier:
     ```tact
     receive( "Start Registration" ) {
         self.requireOwner();
         send(
             SendParameters {
                 to: factoryAddress, // Replace with actual factory address
                 value: self.randomTonDeploymentFee,
                 body: RandomTONRegister{ 
                     randomTonFactoryContract: factoryAddress 
                 }.toCell(),
                 mode: SendPayFwdFeesSeparately
             }
         );
     }
     ```
   - **Callback**: The factory responds with `RandomTONHandleRegistration`, setting `randomTonContract` to the new verifier's address.

---

### 3. **Purchase a Subscription (Optional)**
   - If using a subscription model, purchase a tier first:
     ```tact
     receive( "Buy Basic Subscription" ) {
         self.requireOwner();
         self.randomTonPurchaseBasicSubscription();
     }
     ```
   - **Restriction**: Subscription must be active before using `isSubscription: true` in requests.

---

### 4. **Request Randomness**
   - **Pay-as-You-Go**:
     ```tact
     receive( "Request Pay-As-You-Go" ) {
         self.requireOwner();
         self.randomTonRequestRandomness(
             iterations = 5, // Number of random values needed
             isSubscription = false
         );
     }
     ```
   - **Subscription**:
     ```tact
     receive( "Request Subscription Randomness" ) {
         self.requireOwner();
         self.randomTonRequestRandomness(5, true);
     }
     ```
   - **Key Point**: The verifier contract asynchronously processes the request and returns results via `RandomTONHandleRandomness`.

---

### 5. **Process Randomness Response**
   - Override `randomTonHandleRandomness` to handle the verifier's response:
     ```tact
     override fun randomTonHandleRandomness(randomSeed: Int, iterations: Int) {
         let numberOfPlayers = 20;
         repeat (iterations) {
             let result = self.randomTonRandomize(randomSeed, numberOfPlayers);
             randomSeed = result.newSeed; // Update seed for future use
             
             // Example: Notify winners
             sendWinnerNotification(result.randomNumber);
         }
     }
     ```
   - **Seed Management**: The seed is automatically stored via `randomTonStoreSeed`, but you must manually reset it with `randomTonResetSeed` if needed.

---

### 6. **Use Randomness in Your Logic**
   - Example: Distribute prizes to winners using the generated `randomNumber`:
     ```tact
     fun sendWinnerNotification(winnerId: Int) {
         send(
             SendParameters {
                 to: self.owner,
                 value: self.randomTonMinTransactionValue,
                 body: beginComment("Winner: ${winnerId}").toCell(),
                 mode: SendIgnoreErrors
             }
         );
     }
     ```
   - **Note**: Ensure your contract handles edge cases (e.g., duplicate winners, seed exhaustion).

---

### 7. **Maintenance & Updates**
   - **Withdraw Funds**:
     ```tact
     receive( "Withdraw" ) {
         self.requireOwner();
         sendWithdrawRequest(self.owner);
     }
     ```
   - **Upgrade Verifier**: Re-register with the factory to deploy a new verifier contract if needed.

---

## Troubleshooting
- **Failed Requests**: Check if:
  - `randomTonContract` is set (completed registration).
  - Correct fees are attached (use `randomTonMinTransactionValue` as a baseline).
  - Subscription is active (if using `isSubscription: true`).
- **Seed Errors**: If randomness repeats, ensure you update `randomTonSeed` with `newSeed` after each use.
