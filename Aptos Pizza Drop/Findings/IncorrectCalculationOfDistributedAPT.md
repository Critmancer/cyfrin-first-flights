# Incorrect Unit Code Assumption Between APT and Octa Leads to Underpaying Users on Claim

## Description

When a user signs up, they are allowed to claim `PizzaDrop`. Claiming `PizzaDrop` will provide them a random amount of APT between 100 to 500 as per the documentation.

APT is expressed as Octas on Aptos. 1 APT = 100,000,000 Octas.

In the `pizza_drop::get_random_slice` function, the calculation for the `random_amount` incorrectly assumes the number calculated will be expressed as APT, when its established above that APT is expressed as Octas on Aptos. If we focus on the following 3 lines here

```Solidity
#[randomness]
    entry fun get_random_slice(user_addr: address) acquires ModuleData, State {
        let state = borrow_global_mut<State>(get_resource_address());
--->        let time = timestamp::now_microseconds();
--->        let random_val = time % 401;
--->        let random_amount = 100 + random_val;  // 100-500 APT (in Octas: 10^8 smallest unit)
        table::add(&mut state.users_claimed_amount, user_addr, random_amount);
    }
```

`time` is expressed as EPOCH NOW in MS (Microseconds). The calculation for the above will look like

```markdown
time = 1756544411700
random_val = 1756544411700 % 401 = 96 
random_amount = 100 + 96 = 196 
```

This means the protocol is believed to provide the user with 196 APT when claiming their initial `PizzaDrop`.

However, as expressed previously, Aptos is expressed as Octas (100,000,000 | 10^8) so the user will only receive `0.00000196 APT`, severely being underpaid on their claim.

As PizzaDrop is a core functional requirement of the Pizza Drop Protocol which all new users will utilise after signup, all users will be severely underpaid with "dust" amounts, therefore losing trust and favour in the protocol.

## Risk

**Likelihood**: HIGH

* Every user who signs up and claims PizzaDrop will be effected.

**Impact**: HIGH

* Loss of trust with customers signing up

* Heavy operational load to remediate customer issues

* Immediately incident would be declared on release of Protocol to remediate core affected function

## Recommended Mitigation

Declaration of APT/Octa should be added above the `[event]` declaration section for improved code-quality and to ensure we reduce any "Magic Numbers" in our code.

```diff
+  /// Units: 1 APT = 100,000,000 octas
+  const OCTA: u64 = 100_000_000;
```

And a modification to the `get_random_slice` function

```diff
    #[randomness]
    entry fun get_random_slice(user_addr: address) acquires ModuleData, State {
        let state = borrow_global_mut<State>(get_resource_address());
        let time = timestamp::now_microseconds();
        let random_val = time % 401;
-       let random_amount = 100 + random_val;  // 100-500 APT (in Octas: 10^8 smallest unit)
+       let random_amount = (100 + random_val) * OCTA // 100-500 APT (In Octas: 10^8 smallest unit)
        table::add(&mut state.users_claimed_amount, user_addr, random_amount);
    }
```
## References

References Aptos Tokenomics - https://aptosfoundation.org/currents/aptos-tokenomics-overview 
Aptos Labs Transaction Explorer - https://explorer.aptoslabs.com/?network=mainnet 
Epoch Clock - https://www.epochconverter.com/clock
APT Token - https://explorer.aptoslabs.com/coin/0x1::aptos_coin::AptosCoin/info?network=mainnet
