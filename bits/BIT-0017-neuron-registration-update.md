# BIT-0017: Neuron Registration Redesign

- **BIT Number:** 0017
- **Title:** Neuron Registration Redesign
- **Author(s):** rd4Fun
- **Discussions-to:** [https://discord.com/channels/1120750674595024897/1412842821953781830]
- **Status:** Draft
- **Type:** Subtensor
- **Created:** 2025-09-20
- **Updated:** [Date]
- **Requires:** [none]


## Abstract

Current issues:
 - large burn price fluctuations
 - long periods where nobody can register
 - open to gaming by subnet owners
 - a *batch* of miners enters in the first 3 blocks of an interval
 - registration is dependent on the quality of the registration script not market forces

Proposed fixes:
 - price and difficulty are dynamic, changing every block 
 - remove registration interval, target registration per interval and adjustment alpha.
 - remove rate limits from registration (let price determine entry rate)
 - add a `burned_register_limit` extrinsic


A redesigned and updated registration method is required to remove identified issues that have been observed with the current interval registration method.
i.e. the design is:
- uses a minimum number of owner/sudo settable hyperparameters
- both PoW and burn methods are to be supported
- the subnet price is dynamic per registration and block
- the subnet difficulty dynamic per registration and block

The primary purpose of the registration is to *rate limit* the entrance of **new** miners into the subnet:  
There should be only one key driver that enables rate limiting - cost or difficulty of registration.
- The best KPI for the owner to monitor is the quantity of immune miners in the subnet. 
- the price that is paid for a neuron registration should be determined by the market (not the quality of the selected miners registration scripts)

Note: this BIT does NOT address:
- selection of the miner to replace - there is a CoR pruning score removal discussion addressing this
- the immunity period subject to gaming by the owner
- how price, difficulty, neuron selection, UID immunity and max UID reduction are to work when subnets have multiple unique mechanisms

## Motivation

Some subnet owners have modified the registration settings to:
- block validators and miners from registering a UID
- manipulate hyperparameters in a batch process to register their own UID 
- set immunity so that existing miners are protected
- set immunity so that only the owner hotkey is not in immunity.  
some Miners have developed registration scripts that block new entities from the subnets
- New miners are blocked from entering the subnet by miner registration scripts and the ability to prioritise registration extrinsic.
- The price of the registration is not dynamic enough to follow demand


## Specification

### Proposed new mechanism:
The registration price and difficulty for new subnets should be defined by two new hyperparameters  
- *NeuronReductionInterval*: This will define the number of blocks until the registration price halves from the current value
- *NeuronIncreaseMult*: This will define the multiplier that the registration price increases on a registration (default 2)
- the price and difficulty calculation uses a simple linear reduction in value by block,  where y is price or difficulty:  
> y = (last_y * neuron_increase_mult) - (last_y / neuron_reduction_interval) * (current_block - last_reg_block)
- A new registration extrinsic is created `burned_register_limit` where the user can set the maximum price for a burn registration
- if two registrations occur less than half the NeuronReductionInterval apart - apply the multiplier to both price and difficulty. 
- When a **PoW** registration occurs the *Burn price* is frozen for one NeuronReductionInterval
- When a **Burn** registration occurs the *PoW difficulty* is frozen for one NeuronReductionInterval

- Both Burn and Difficulty have a minimum limit  
- Neither Burn or Difficulty have a maximum limit  

#### Additional storage variables
- last_registration_PoW - **TRUE** if the last registration was a Proof of Work registration
- minBurnCost - the minimum cost for a burn in this subnet
- minDifficulty - the minimum difficulty for PoW in this subnet

### Price and Difficulty algorithm formulas
When a registration occurs, set:  
- high_registration_rate = TRUE if current_block - last_reg_block < NeuronReductionInterval / 2
- burn_at_last_reg = Burn Value at last registration (either type of registration)  
- difficulty_at_last_reg = Difficulty Value at last registration (either type of registration)  
- last_reg_block = current block
- last_registration_PoW = TRUE if last Registration used PoW

*neuron_burn_price calculation per block*
```
if last_registration_PoW and not high_registration_rate then  
    burn_blocks_since_reg = current_block - (last_reg_block + NeuronReductionInterval)
    if burn_blocks_since_reg > 0  then
        neuron_burn_price = (burn_at_last_reg * 1 ) - (burn_at_last_reg / NeuronReductionInterval * burn_blocks_since_reg)
    else
        neuron_burn_price = burn_at_last_reg
else
    neuron_burn_price = (burn_at_last_reg * NeuronIncreaseMult) - (burn_at_last_reg / NeuronReductionInterval) * (current_block - last_reg_block)
neuron_burn_price = max (neuron_burn_price, minBurnCost)
```
*neuron_difficulty calculation per block*
```
if not last_registration_PoW and not high_registration_rate then
    difficulty_blocks_since_reg = current_block - (last_reg_block + NeuronReductionInterval)
    if difficulty_blocks_since_reg > 0 then
        neuron_difficulty = (difficulty_at_last_reg * 1) - (difficulty_at_last_reg / NeuronReductionInterval * difficulty_blocks_since_reg)
    else
        neuron_difficulty = difficulty_at_last_reg
else
    neuron_difficulty = (difficulty_at_last_reg * NeuronIncreaseMult) - (difficulty_at_last_reg / NeuronReductionInterval) * (current_block - last_reg_block)
neuron_difficulty = max(neuron_difficulty, minDifficulty)
```

Limits

NeuronReductionInterval must be > 0  (divide by zero)
NeuronReductionInterval > 4 recommended so the high_registration_rate test uses at least 2 blocks
NeuronReductionInterval needs a high limit - e.g. must return to the same difficulty or cost within one 1 or 2 days 

## Rationale

### the freezing:
 in an ideal situation one registration occurs every NeuronReductionInterval, by freezing the other registration method for one interval, the unused registration method is not made easier or cheaper until the Interval is over.
### the high registration rate
If two registrations occur quickly in series, by increasing the difficulty or price of the other type of registration it slows down the possible registration using the other method.
### the burned_register_limit extrinsic
If two registrations occur next to each other, the second user might end up paying much more than expected.
- By having a price limit, the second user can choose to fail if their price limit is exceeded.
- The burned_register_limit also enables the possibility of price limited multiple registrations in the same block, with each registration multiplying the cost.

### Hyperparameter access
rd4Fun opinion:  
- The NeuronIncreaseMult should be a sudo only because it is too easy for the subnet owner to game and manipulate. 
- The NeuronReductionInterval should be a owner changeable as this parameter controls the entry rate of new neurons into the subnet
### Limits
If NeuronReductionInterval does not have a high limit then the owner can register and keep the price artificially high for a long time 
- a discussion is needed to determine a high limit, especially with regards to UID count reduction and subnet mechanisms.
- suggestion is 7200 i.e. one registration per day for 256 UID's
NeuronIncreaseMult other than 2
  - value of 1 means fixed entry price - this is possible by setting neuron reduction interval to 1
  - value greater than 2 
    - This permits less neurons to register while the miners find the registration market price.
    - once the price reaches a market level, a high value will probably cause higher variation in entry price due to timing
    - if the general consensus is for the owner to be able to change this value then an absolute max of 4 or 5 is recommended as this ends up being a power multiplier

## Backwards Compatibility

PoW registration still works
Burn registration still works

## Security Considerations

### Attack vectors
1. Owner increasing the NeuronReductionInterval so the price remains high for many days (up to 65535 blocks)
    - set a high limit to the possible value of NeuronReductionInterval
2. Owner decreasing the NeuronReductionInterval to 1 or 2 to snipe a cheap registration then resetting the interval to a long value
    - the reset to a high value is blocked by the hyperparameter change rate limit
3. Owner increasing the NeuronIncreaseMult above 16 so the price remains high for a very long time
    - make this parameter a sudo call
4. Owner decreasing the NeuronIncreaseMult to less than 1 ... makes it cheap for all miners to be deregistered and forced into immunity
    -  make this parameter a sudo call
5. Miner using a script to register
  - timing is now longer block based, its price based
  - script must pay more for registrations as each registration price increases
6. Miner using MeV tactics to snipe the cheapest registration
  - FIFO on the Proof of authority nodes 
  - the sniping miner must pay more for his registrations as he can no longer batch register at the one price
7. Miner blocking anyone else from registering
  - the price doubling each registration makes it economically cost prohibitive

### Mitigation - limits and formulas
1. range limit the NeuronIncreaseMult from 1 to 5 with up to 3 decimal points
2. range limit NeuronReductionInterval from 1 to x where x is a small multiple of days

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
