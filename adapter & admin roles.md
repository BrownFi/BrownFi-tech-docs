# BrownFi V2: design for the price adapter & admin roles
To ensure highest security level for BrownFi AMM, we design an oracle price adapter to fetch price for all pools, and separate admin (setter) roles regarding various settings in the BrownFi protocol. Read the related BrownFi V2 tech-specifications [HERE](https://github.com/BrownFi/BrownFi-tech-docs/blob/main/BrownFi-techspecs-V2.md).

## 1. Oracle price adapter
BrownFi V2 introduces a new design for oracle price adapter: 2 price sources. The adapter address can be changed to a new address. 

The adapter pulls or receives token prices from oracles, then feeds to pair contracts (liquidity pools) for swaps and/or adding LP. By default, at least one oracle price source **MUST** be set.   

Optionally, the second oracle can be enabled and added by the _OracleSetter_. Then, the adapter will compute the price mean, then feed to pair contracts. The procedure is:  
- Pull _oraclePrice1_ and _oraclePrice2_
- Verify that two sources have the same decimals.
- Verify that variance of the two price sources doesn't exceed 0.5%, i.e. $variance = \frac{\|oraclePrice1 - oraclePrice2\|}{oraclePrice1 + oraclePrice2} \leq 1/200=0.005$. Otherwise, invalid price => revert TX. 
- Compute the mean (average) price $meanPrice = (oraclePrice1 + oraclePrice2)/2$.

## 2. Admin roles
We define four admind (setter) roles associated with certain param settings: oracle price setter (_OracleSetter_), Tuning Setter (_TuningSetter_), Business Setter (BizSetter) and protocol suppervisor (_Pauser_). The 3 roles are independent. After deployment, the deployer must transfer the following roles to appropriated new admin addresses. 

**OracleSetter**:
- _SetPriceOracle_: set adapter contract address
- _SetOracleof_: set price feedID
- _SetMinPriceAge_: set the minimal valid time for an updated price (i.e. invalid price if exceeding the minimal time-period)

**TuningSetter**:
- setFee: set transaction fee
- setKappa: set parameter which controls liquidity concentration

**BizSetter**:
- setFeeto: set receiver of protocol fee (receiving diluted LP tokens per swap) which belongs to the developer
- setProtocolfee: set the share percentage of trading fee which will belong to the developer

**Pauser**: 
- Pause the entire protocol

## 3. Timelock to effective setting
To prevent acidental and sudden change in the protocol, we apply timelock regarding setting change until effectiveness, except protocol pausing.  
- All admin role changes require a timelock = 24h to be effective.
- All settings (on OracleSetter, on TuningSetter, on BizSetter) require a timelock of 8 hours to be effective.
- Only "_Pause the entire protocol_" is **immediately effective**. 

![image](https://github.com/user-attachments/assets/e5fe665c-316c-453a-967d-b98dd9e655a7)

## Notes for unitests
- Test changes of 4 admin roles in 5-minute timelock
- Test role separation: 1 role 1 separate admin address which MUST-NOT overlap each other. That means BizSetter admin cannot set the params _setFee_ and _setKappa_ of TuningSetter role.
- Test the immediate pause
- Test other settings in 3-minute timelock
- Test overlapping per setting before effectiveness: for example, submit 1st TX to change Kappa, while waiting to be effective, another TX is submitted to change Kappa. The protocol should care the latest TX.  



