# Introduction

**Protocol Name:** Polygon Pos

**Category:** DeFi

**Smart Contract:** LiFiDiamond

# Function Analysis

**Function Name:** fallback

**Block Explorer Link:** https://polygonscan.com/address/0x1231deb6f5749ef6ce6943a275a1d3e7486f4eae#code

**Function Code:** 

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import { LibDiamond } from "./Libraries/LibDiamond.sol";
import { IDiamondCut } from "./Interfaces/IDiamondCut.sol";
import { LibUtil } from "./Libraries/LibUtil.sol";

contract LiFiDiamond {
    constructor(address _contractOwner, address _diamondCutFacet) payable {
        LibDiamond.setContractOwner(_contractOwner);

        // Add the diamondCut external function from the diamondCutFacet
        IDiamondCut.FacetCut[] memory cut = new IDiamondCut.FacetCut[](1);
        bytes4[] memory functionSelectors = new bytes4[](1);
        functionSelectors[0] = IDiamondCut.diamondCut.selector;
        cut[0] = IDiamondCut.FacetCut({
            facetAddress: _diamondCutFacet,
            action: IDiamondCut.FacetCutAction.Add,
            functionSelectors: functionSelectors
        });
        LibDiamond.diamondCut(cut, address(0), "");
    }

    // Find facet for function that is called and execute the
    // function if a facet is found and return any value.
    // solhint-disable-next-line no-complex-fallback
    fallback() external payable {
        LibDiamond.DiamondStorage storage ds;
        bytes32 position = LibDiamond.DIAMOND_STORAGE_POSITION;

        // get diamond storage
        // solhint-disable-next-line no-inline-assembly
        assembly {
            ds.slot := position
        }

        // get facet from function selector
        address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;

        if (facet == address(0)) {
            revert LibDiamond.FunctionDoesNotExist();
        }

        // Execute external function from facet using delegatecall and return any value.
        // solhint-disable-next-line no-inline-assembly
        assembly {
            // copy function selector and any arguments
            calldatacopy(0, 0, calldatasize())
            // execute function call using the facet
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            // get any return value
            returndatacopy(0, 0, returndatasize())
            // return any return value or error back to the caller
            switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }

    // Able to receive ether
    // solhint-disable-next-line no-empty-blocks
    receive() external payable {}
}

```

# Explanation

## **Purpose:**
The purpose of the fallback function in the LiFiDiamond contract is to handle calls to functions that are not explicitly defined in the contract itself. When a call is made to the contract and the specified function is not found, the fallback function is triggered. It acts as a catch-all mechanism to ensure that any function call not directly implemented in the contract can still be executed, provided it exists in one of the facets of the diamond.

## **Detailed Usage:**
The fallback function leverages delegatecall to delegate the execution of the called function to an appropriate facet contract.

The reason why delegatecall is used in the fallback function is to execute the fallback function in the context of the calling contract (LiFiDiamond). This means any state changes, events emitted, or ether transfers occur in the context of the LiFiDiamond contract, not the facet contract.

Here's a detailed breakdown of how this works:

### **1. Retrieve Diamond Storage:** 
The fallback function first retrieves the diamond storage, which holds the mappings between function selectors and their corresponding facet addresses.

```
LibDiamond.DiamondStorage storage ds;
bytes32 position = LibDiamond.DIAMOND_STORAGE_POSITION;

assembly {
    ds.slot := position
}
```
### **2. Get Facet Address:** 
It then fetches the address of the facet contract that implements the function corresponding to the function selector (msg.sig).

```
address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;
```

### 3. **Check Facet Existence:** 
If the facet address is zero, it means the function does not exist in any facet, and the call reverts with an error.

```
if (facet == address(0)) {
    revert LibDiamond.FunctionDoesNotExist();
}
```

### 4. **Delegate Call Execution:** 
If a facet is found, the fallback function uses delegatecall to execute the function in the context of the calling contract, preserving the current contract's state and context.

```
assembly {
    calldatacopy(0, 0, calldatasize())
    let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
    returndatacopy(0, 0, returndatasize())
    switch result
    case 0 {
        revert(0, returndatasize())
    }
    default {
        return(0, returndatasize())
    }
}
```

## **Impact:**

The fallback function is crucial for the LiFiDiamond contract's functionality within the protocol for three main reasons:

### **1. Extensibility:** 
It allows the contract to dynamically handle new functions without requiring changes to the core contract. New functionality can be added by deploying new facets and updating the diamond storage.

### **2. Modularity:** 
The contract can be broken down into smaller, manageable facet contracts, each implementing a subset of the overall functionality. This separation of concerns makes the system easier to maintain and upgrade.

### **3. Upgradability:** 
With the fallback mechanism and delegatecall, the contract can be upgraded by changing the facet implementations without altering the core LiFiDiamond contract, thus preserving the contract's address and state.

