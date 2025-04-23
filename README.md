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
