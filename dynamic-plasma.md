| Field | Description |
| ----- | ----------- |
| ZIP | - |
| Title | Dynamic Plasma |
| Author | vilkris |
| Status | DRAFT |
| Type | Spork, Protocol |
| Created | 2024-02-12 |
| License | BSD-2-Clause |
| License-Code | GPL v3.0 |
| Comments-URI | - |

# Table of Contents
1. [Abstract](#abstract)
2. [Motivation](#motivation)
3. [Specification](#specification)
4. [Rationale](#rationale)
5. [Account Block Ordering](#account-block-ordering)
6. [Reference Implementation](#reference-implementation)

# Abstract

Dynamic Plasma - a pricing mechanism that dynamically adjusts the price of plasma resources and the size of momentums to balance the usage of plasma resources and to deal with transient congestion.

We propose adding fusion price and work price values to the protocol, which can increase and decrease each momentum according to a formula which is a function of resource plasma used in the parent momentum and adjustable resource plasma targets. The algorithm results in the respective resource price increasing when the resource's usage is over the resource's usage target and decreasing when the usage is below target. Account blocks must commit a plasma amount of at least the base plasma amount multiplied by the plasma resource's current price in order for the block to be considered eligible for inclusion in the next momentum.

# Motivation

In order to target specific resource requirements for a node (e.g. RAM and CPU requirements), there should be a hard limit on network throughput to limit the overall computational workload imposed on a node. Since network throughput is limited, there should exist a mechanism that sets a price on network usage in a fair and predictable manner depending on demand.

To improve usability, both QSR fusion and proof-of-work (PoW) can be used to cover the plasma requirements of a transaction. In order to balance the usage of these resources, there should exist a mechanism that prices the resources depending on their usage, making it infeasible for one resource type to take over the full bandwidth of the network for an extended period of time.

This ZIP proposes to solve both of these issues by introducing price values for fused plasma (fusion price) and PoW plasma (work price). These values are adjusted up and down by the protocol depending on how far away a resource's usage is from the resource's per-momentum usage target. When a resource's usage exceeds its target, the resource's price increases slightly and when the usage is below target, the price decreases slightly. Because these price changes are constrained, the maximum difference in resource price from momentum to momentum is predictable. This predictability allows wallets to automatically set a transaction's plasma amount in such a way that ensures the transaction's eligilibility for inclusion in the upcoming momentum in a highly reliable fashion. It is expected that most users will not have to manually set plasma amounts, even in periods of high network activity.

This ZIP has drawn inspiration from Ethereum's base fee adjustment mechanism presented in [EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md).

# Specification
## Momentum data structure and hash
NextFusionPrice and NextWorkPrice properties are added to the momentum data structure and are included in the momentum hash.

```go
type DynamicPlasmaMomentum struct {
	Version         uint64
	ChainIdentifier uint64

	Hash         types.Hash
	PreviousHash types.Hash
	Height       uint64    

	TimestampUnix uint64    
	Timestamp     *time.Time 

	Data    []byte          
	Content MomentumContent

	ChangesHash types.Hash 

	producer  *types.Address  
	PublicKey ed25519.PublicKey
	Signature []byte            

	NextFusionPrice uint64
	NextWorkPrice   uint64
}
```

## Momentum validity
Total nominal base plasma values are calculated for fused plasma and PoW plasma on each momentum. The nominal base plasma values are compared to the per-momentum nominal resource base plasma targets and the price of a resource is increased or decreased based on how far away the value is from its target. The calculated resource prices apply to the next momentum.

The following default parameters are used:

* The maximum base plasma in a momentum is 4,200,000
* The target fused base plasma usage in a momentum is 1,050,000
* The target PoW base plasma usage in a momentum is 1,050,000
* The maximum change in resource price is 10%
* The price change denominator is 20

Pseudo-code presenting the additional momentum validation rules once the Dynamic Plasma spork is active. Uses integer math, all values are integers.

```go
AccountBlockBasePlasma := 21000
EmbeddedSimplePlasma := 52500
PriceScaleFactor := 1000
MinResourcePrice := 1000

// Configurable values (via governance)
MaxBasePlasmaInMomentum := plasmaContract.MaxBasePlasmaInMomentum // default: 4200000
FusedPlasmaTarget := plasmaContract.FusedPlasmaTarget             // default: 1050000
PowPlasmaTarget := plasmaContract.PowPlasmaTarget                 // default: 1050000
PriceChangeDenominator := plasmaContract.PriceChangeDenominator   // default: 20
MaxPriceChangePercent := plasmaContract.MaxPriceChangePercent     // default: 10

// Validate momentum version
if momentum.Version < 2 {
    panic("incorrect momentum version")
}

fusionPrice := MinResourcePrice
workPrice := MinResourcePrice
if previousMomentum.Version >= 2 {
    // The resource prices calculated in the previous momentum are enforced on the current momentum
    fusionPrice := previousMomentum.FusionPrice
    workPrice := previousMomentum.WorkPrice
}

contractBlockCount := 0
nominalFusedBasePlasma := 0
nominalPowBasePlasma := 0
maxContractBlocks := MaxBasePlasmaInMomentum / EmbeddedSimplePlasma // 80 blocks by default

// Validate account block rules
for _, block := range momentum.Blocks {
      if IsEmbeddedAddress(block.Address) {
      // Validate contract account block count
      contractBlockCount++
      if contractBlockCount > maxContractBlocks {
        panic("exceeded maximum allowed contract account blocks in momentum")
      }
    } else {
      // Validate resource prices used for the block
      minAllowedFusedPlasma := AccountBlockBasePlasma * fusionPrice / PriceScaleFactor
      minAllowedPriceRatio := minAllowedFusedPlasma * workPrice * block.BasePlasma
      blockPriceRatio := (block.FusedPlasma * workPrice + block.PowPlasma * fusionPrice) * AccountBlockBasePlasma
      if blockPriceRatio < minAllowedPriceRatio {
        panic("block price is too small")
      }

      // Calculate nominal base plasma values and validate total base plasma usage
      // Attribute the block's base plasma to fusion and PoW based on their ratio
      f := block.FusedPlasma * fusionPrice * workPrice
      p := block.PowPlasma * fusionPrice * workPrice
      fused := f * block.BasePlasma / (f + p)
      nominalFusedBasePlasma += fused
      nominalPowBasePlasma += block.BasePlasma - fused
      totalBasePlasma := nominalFusedBasePlasma + nominalPowBasePlasma
      if totalBasePlasma > MaxBasePlasmaInMomentum {
        panic("exceeded maximum allowed base plasma in momentum")
      }
    }
}

// Calculate and validate next fusion price
usedPlasmaDelta := nominalFusedBasePlasma - FusedPlasmaTarget
change := fusionPrice * usedPlasmaDelta / targetPowBasePlasma / PriceChangeDenominator
nextFusionPrice := fusionPrice + change

maxNextPrice := fusionPrice * (MaxPriceChangePercent + 100) / 100
minNextPrice := fusionPrice * (100 - MaxPriceChangePercent) / 100

if nextFusionPrice < MinResourcePrice {
    nextFusionPrice = MinResourcePrice
} else if nextPrice < minNextPrice {
    nextFusionPrice = minNextPrice
} else if nextPrice > maxNextPrice {
    nextFusionPrice = maxNextPrice
}

if nextFusionPrice != momentum.FusionPrice {
    panic("mismatch in momentum fusion price")
}

// Calculate and validate next work price
usedPlasmaDelta = nominalPowBasePlasma - PowPlasmaTarget
change = workPrice * usedPlasmaDelta / targetPowBasePlasma / PriceChangeDenominator
nextWorkPrice := workPrice + change

maxNextPrice = workPrice * (MaxPriceChangePercent + 100) / 100
minNextPrice = workPrice * (100 - MaxPriceChangePercent) / 100

if nextWorkPrice < MinResourcePrice {
    nextWorkPrice = MinResourcePrice
} else if nextWorkPrice < minNextPrice {
    nextWorkPrice = minNextPrice
} else if nextWorkPrice > maxNextPrice {
    nextWorkPrice = maxNextPrice
}

if nextWorkPrice != momentum.WorkPrice {
    panic("mismatch in momentum work price")
}

// Rest of momentum validation...
```

## Account Block validity

No changes to account block validity within an account chain are introduced in this ZIP.

## Configurable parameters
Pillars have the ability to adjust the values of the following parameters via the governance module:

| Parameter Name | Default Value | Minimum Valid Value | Maximum Valid Value | Data Type
| - | - | - | - | - |
| MaxBasePlasmaInMomentum | 4,200,000 | 210,000 | 210,000,000,000,000 | uint64
| FusedPlasmaTarget | 1,050,000 | 105,000 | MaxBasePlasmaInMomentum - PowPlasmaTarget  | uint64 |
| PowPlasmaTarget | 1,050,000 | 105,000 | MaxBasePlasmaInMomentum - FusedPlasmaTarget | uint64 |
| MaxPriceChangePercent | 10% | 1% | 100% | uint8 |
| PriceChangeDenominator | 20 | 1 | 100 | uint8 |

## Plasma Contract Updates

### SetVariables Method

`SetVariables` sets the configurable parameters for Dynamic Plasma. The method can only be called by the governance contract. Once the parameters are set they will take immediate effect on protocol level.

Plasma cost: Embedded Simple

JSON ABI:
```
{"type":"function","name":"SetVariables", "inputs":[
  {"name":"maxBasePlasmaInMomentum","type":"uint64"},
  {"name":"fusedPlasmaTarget","type":"uint64"},
  {"name":"powPlasmaTarget","type":"uint64"},
  {"name":"maxPriceChangePercent","type":"uint8"},
  {"name":"priceChangeDenominator","type":"uint8"}
]}
```

### PlasmaVariables Variable

The `PlasmaVariables` variable type contains the configurable parameters for Dynamic Plasma.

JSON ABI:
```
{"type":"variable","name":"plasmaVariables","inputs":[
  {"name":"maxBasePlasmaInMomentum","type":"uint64"},
  {"name":"fusedPlasmaTarget","type":"uint64"},
  {"name":"powPlasmaTarget","type":"uint64"},
  {"name":"maxPriceChangePercent","type":"uint8"},
  {"name":"priceChangeDenominator","type":"uint8"}
]}
```

## Plasma RPC Updates

### embedded.plasma.getVariables

**Request params:** none

**Response results:** returns a PlasmaVariables object

```
{
  "type": "object",
  "properties": {
    "maxBasePlasmaInMomentum": {
      "type": "uint64"
    },
    "fusedPlasmaTarget": {
      "type": "uint64"
    },
    "powPlasmaTarget": {
      "type": "uint64"
    },
    "maxPriceChangePercent": {
      "type": "uint8"
    },
    "priceChangeDenominator": {
      "type": "uint8"
    }
  },
  "required": [
    "maxBasePlasmaInMomentum",
    "fusedPlasmaTarget",
    "powPlasmaTarget",
    "maxPriceChangePercent",
    "priceChangeDenominator"
  ]
}
```

# Rationale

### Fusion Price and Work Price

The fusion price and work price values are introduced to allow for a harmonized abstraction for both fusion and PoW plasma cost.

### Nominal Base Plasma

Base plasma is an abstraction that represents computational resources (and network bandwidth). In order to balance how much of the network’s computational resources can be used by PoW and fusion, there has to be a mechanism to associate PoW and fused plasma usage with base plasma. An account block's base plasma is attributed to fusion and PoW depending on their ratio.

### MaxBasePlasmaInMomentum, FusedPlasmaTarget, and PowPlasmaTarget

The default value of 4,200,000 for MaxBasePlasmaInMomentum equals 200 account blocks using 21,000 plasma (AccountBlockBasePlasma) each. The default value of 1,050,000 for both FusedPlasmaTarget and PowPlasmaTarget is 25% of MaxBasePlasmaInMomentum. This means that the combined per-momentum plasma target is 50% of the maximum, which equals 2,100,000 plasma (100 account blocks using 21,000 plasma each). Since the dynamic pricing mechanism adjusts the plasma prices depending on network saturation, the long term average per-momentum plasma usage should be close to the target of 2,100,000 plasma in a fully utilized network. These default values were selected so that the proposed changes in this ZIP do not introduce significant changes to the node's computational requirements. Currently the protocol limits the maximum amount of account blocks to one hundred per momentum.

The plasma usage targets (FusedPlasmaTarget and PowPlasmaTarget) are initially set to 1,050,000 for both plasma types, since it is hard to predict how the different resource types will be used in a fully saturated network. Adjusting the ratio to favor one over the other doesn't seem justified right now, but it is possible to do via the governance module if the need arises.

### MaxPriceChangePercent and PriceChangeDenominator

To keep the price of plasma predictable, the maximum rate at which the prices can change is constrained. Knowing what the possible price range of plasma will be over a period of time makes it easier for clients to attach plasma to transactions in an efficient manner. The price change denominator controls the rate at which a resource's price can change and the max price change percent limits the maximum percentage the price can change at once.

### PriceScaleFactor

To prevent having to use decimal numbers for on-chain data, the plasma price values are scaled by a factor of 1000 and used as integers.

### Contract Blocks

Contract account blocks do not require any plasma to be attached to them to be considered valid blocks. This means that contract account blocks are not counted towards the per-momentum total plasma usage amount and will not affect the plasma pricing calculations.

The increased plasma requirements for user blocks calling embedded contract methods cover the additional computational complexity imposed on the node by the contract executions. The complexity-dependent plasma requirements ensure that in a fully utilized network the average computational load to process momentums stays stable in the long term, even though the computational resources required to process a momentum differs.

*Related: [Vitalik Buterin's reasoning on why he thinks block size variance is not an issue in Ethereum](https://web.archive.org/web/20250319152731/https://notes.ethereum.org/@vbuterin/eip_1559_spikes)*

# Account Block Ordering

While account block ordering is not a part of this ZIP's specification, all implementations have to consider how to order account blocks by default. It is suggested that contract account blocks are prioritized over user account blocks and that user account blocks are ordered based on the block's plasma price or time of arrival.

# Reference Implementation

https://github.com/hypercore-one/hyperqube_z/tree/feature/dynamic_plasma
