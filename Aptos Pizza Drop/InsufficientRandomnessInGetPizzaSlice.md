# Insufficient randomness in get\_random\_slice leads to predictable claim amounts

## Description

The current random implementation of `get_random_slice` uses `let time = timestamp::now_microseconds();` to derive a random amount value based on block timestamp (+100) to determine the amount of APT to award a user on claim.

The current implementation creates predictability as all information is visible and stored on the blockchain and therefore is not truely random and can create a biased reward allocation if abused by the owner, or a block proposer.

```solidity
    #[randomness]
    entry fun get_random_slice(user_addr: address) acquires ModuleData, State {
        let state = borrow_global_mut<State>(get_resource_address());
--->    let time = timestamp::now_microseconds();
        let random_val = time % 401;
        let random_amount = 100 + random_val;  // 100-500 APT (in Octas: 10^8 smallest unit)
        table::add(&mut state.users_claimed_amount, user_addr, random_amount);
    }
```

## Risk

Due to the randomness being predictable, this can enable claim manipulation creating a biased and unfair environment for its user, especially if the owner acts in coordination with a bad-actor.

**Likelihood**: LOW

* The RNG is not random, but predictable. Therefore the owner can time submissions

* If working with the owner, a block proposer can pick a beneifical timestamp for maximum output.

**Impact**: MEDIUM

* Ability to choose who to send maximum APT Claims to

* Potential ability to drain the contract due to predictable RNG Claims.

## Proof of Concept

The below script can be added to the test section with an artificial timestamp showing a max-reward is possible.

```solidity
#[test(deployer = @pizza_drop, user = @0x333, framework = @0x1)]
fun poc_attacker_targets_max_reward(
    deployer: &signer, user: &signer, framework: &signer
) acquires State, ModuleData {
    // Start test time + publish AptosCoin so coin::register won't abort
    aptos_framework::timestamp::set_time_has_started_for_testing(framework);
    let (burn_cap, mint_cap) = aptos_framework::aptos_coin::initialize_for_test(framework);

    // Accounts
    aptos_framework::account::create_account_for_test(@pizza_drop);
    aptos_framework::account::create_account_for_test(signer::address_of(user));
    init_module(deployer);

    // Force Max Randoness Based on Block Timestamp
    let base_us: u64 = 1_756_541_886_987_060;
    let t: u64 = base_us - (base_us % 401) + 400;
    aptos_framework::timestamp::update_global_time_for_test(t);

    // Register
    let u = signer::address_of(user);
    register_pizza_lover(deployer, u);
    let amt = get_claimed_amount(u);

    // Log the amount
    std::debug::print(&b"Assigned amount (octas):");
    std::debug::print(&amt);

    // Assert max-value
    assert!(amt == 500, 9003);

    // Cleanup up
    aptos_framework::coin::destroy_burn_cap(burn_cap);
    aptos_framework::coin::destroy_mint_cap(mint_cap);
}
```

## Recommended Mitigation

Aptos has its own inbuilt randomness. Consult the documentation in references at the bottom and implement it correctly.

Within the framework imports at the top, utilise the Aptos randomness framework.

```diff
module pizza_drop::airdrop {
    use std::signer;
    use aptos_framework::account;
    use aptos_std::table::{Self, Table};
    use aptos_framework::event;
    use aptos_framework::timestamp;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::aptos_coin::AptosCoin;
+   use aptos_framework::randomness;
```

```diff
#[randomness]
- public entry fun register_pizza_lover(owner: &signer, user: address) acquires ModuleData, State {
+ entry fun register_pizza_lover(owner: &signer, user: address) acquires ModuleData, State {
    let state = borrow_global_mut<State>(get_resource_address());
    assert!(signer::address_of(owner) == state.owner, E_NOT_OWNER);
-    get_random_slice(user);
+    let reward = randomness::u64_range(100, 501);
+    table::add(&mut state.users_claimed_amount, user, reward);
    event::emit(PizzaLoverRegistered { user });
}
```

and remove `get_random_slice` entirely.

```diff
-    #[randomness]
-    entry fun get_random_slice(user_addr: address) acquires ModuleData, State {
-        let state = borrow_global_mut<State>(get_resource_address());
-        let time = timestamp::now_microseconds();
-        let random_val = time % 401;
-        let random_amount = 100 + random_val;  // 100-500 APT (in Octas: 10^8 smallest unit)
-        table::add(&mut state.users_claimed_amount, user_addr, random_amount);
-    }
```

## References

Aptos Randomness - <https://aptos.dev/build/smart-contracts/move-security-guidelines#randomness>
