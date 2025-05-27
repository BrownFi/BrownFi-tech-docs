# BrownFi V2: design for the price adapter & admin roles
To ensure highest security level for BrownFi AMM, we design an oracle price adapter to fetch price for all pools, and separate admin (setter) roles regarding various settings in the BrownFi protocol. Read the related BrownFi V2 tech-specifications [HERE](https://github.com/BrownFi/BrownFi-tech-docs/blob/main/BrownFi-techspecs-V2.md).

## Oracle price adapter
BrownFi V2 introduces a new design for oracle price adapter: 2 price sources and skewness.  

The adapter pulls or receives asset prices from oracle, then feed to pair contracts (liquidity pools) for swaps and/or adding LP. By default, at least one oracle price source **MUST** be set.   

Optionally, the second oracle can be enabled and added by the _OracleSetter_. Then, the adapter will compute the price mean, then feed to pair contracts. The procedure is:  
- Pull _oraclePrice1_ and _oraclePrice2_
- Verify that two sources have the same decimals.
- Verify that variance of the two price sources doesn't exceed 1%, i.e. $variance = \frac{\|oraclePrice1 - oraclePrice2\|}{oraclePrice1 + oraclePrice2} \leq 1/200=0.005$. Otherwise, invalid price => revert TX. 
- Compute the mean (average) price $meanPrice = (oraclePrice1 + oraclePrice2)/2$.

## Admin roles
We define four admind (setter) roles associated with certain param settings: oracle price setter (_OracleSetter_), Tuning Setter (_TuningSetter_), Business Setter (BizSetter) and protocol suppervisor (_Pauser_). The 3 roles are independent. After deployment, the deployer must transfer 

**OracleSetter**:
- setDecimalShift
- setPricefeed1
- setQTI1
- setPricefeed2
- setQTI2
- setPriceVariance

**TuningSetter**:
- setFee: set transaction fee
- setKappa: set parameter which controls liquidity concentration
- setSkewness: enable/disable skewness params

**BizSetter**:
- setFeeto: set receiver of protocol fee (receiving diluted LP tokens per swap) which belongs to the developer
- setProtocolfee: set the share percentage of trading fee which will belong to the developer

**Pauser**: 
- Pause the entire protocol

## Timelock to effective setting
To prevent acidental and sudden change in the protocol, we apply timelock regarding setting change until effectiveness, except protocol pausing.  
- All role changes require a timelock = 24h to be effective.
- All settings (on oracle price adapter, setFee, setKappa, setFeeto, setProtocolfee) require 1 hours to be effective.
- Only "_Pause the entire protocol_" is **immediately effective**. 

![image](https://github.com/user-attachments/assets/e5fe665c-316c-453a-967d-b98dd9e655a7)



