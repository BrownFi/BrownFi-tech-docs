# BrownFi V2: design for the price adapter & admin roles
To ensure highest security level for BrownFi AMM, we design an oracle price adapter to fetch price for all pools, and separate admin (setter) roles regarding various settings in the BrownFi protocol. Read the related BrownFi V2 tech-specifications [HERE](https://github.com/BrownFi/BrownFi-tech-docs/blob/main/BrownFi-techspecs-V2.md).

## 1. Oracle price adapter
### 1.1. Multi-sources
BrownFi V2 introduces a new design for oracle price adapter: 2 price sources. The adapter address can be changed to a new contract address. 

The adapter pulls or receives token prices from oracles, then feeds to pair contracts (liquidity pools) for swaps and/or adding LP. By default, at least one oracle price source **MUST** be set.   

Optionally, the second oracle (_implement later_) can be enabled and added by the _OracleSetter_. Then, the adapter will compute the price mean, then feed to pair contracts. The procedure is:  
- Pull _oraclePrice1_ and _oraclePrice2_
- Verify that two sources have the same decimals.
- Verify that variance of the two price sources doesn't exceed 0.5%, i.e. $variance = \frac{\|oraclePrice1 - oraclePrice2\|}{oraclePrice1 + oraclePrice2} \leq 1/200=0.005$. Otherwise, invalid price => revert TX. 
- Compute the mean (average) price $meanPrice = (oraclePrice1 + oraclePrice2)/2$.

### 1.2. Adding pool skewness to price
In V2, BrownFi introduces pool skewness (share imbalance) to oracle price before swap. This is to incentivize trades toward balancing between the two token reserves, otherwise discourage.   
However, we want this feature to be ON/OFF, with default OFF. 

Consider the pool reserve $(x, y)$ with oracle price $P^o_X, P^o_Y$, compute the absolute skewness of the pool $S=\frac{\|xP^o_X-yP^o_y\|}{xP^o_X + yP^o_y}$.  
- If $Xreserve \geq Yreserve$, then we compute new price with skewness $P_X=P^o_X(1-\lambda* S)$ and $P_Y=P^o_Y(1+\lambda*S)$;
- If $Xreserve \leq Yreserve$, then we compute new price with skewness $P_X=P^o_X(1+\lambda* S)$ and $P_Y=P^o_Y(1-\lambda*S)$.

Here, per pool, we introduce a new configurable param $lambda (\lambda)$ defined by max imbalance target (80-20) with default $\lambda=0.02$. Lambda is limited within $[0.001, 1]$.
 
## 2. Admin roles
We define three admind (setter) roles associated with certain param settings: oracle price setter (_OracleSetter_), Business Setter (BizSetter) and protocol suppervisor (_Pauser_). The 3 roles are independent. After deployment, the deployer must transfer the following roles to appropriated new admin addresses. 

**OracleSetter** acts on a specific pool. Two pools may have two separated Oracle Setters who are responsible for the 3 following settings: 
- _SetPriceOracle_: set adapter contract address
- _SetOracleof_: set price feedID
- _SetMinPriceAge_: set the minimal valid time for an updated price (i.e. invalid price if exceeding the minimal time-period)

**BizSetter** acts on a specific pool. Two pools may have two separated Bussiness Setters who are responsible for the 4 following settings: 
- _setFee_: set transaction fee
- _setKappa_: set parameter which controls liquidity concentration
- _setFeeto_: set receiver of protocol fee (receiving diluted LP tokens per swap) which belongs to the developer
- _setProtocolfee_: set the share percentage of trading fee which will belong to the developer

**Pauser** acts on all pools (i.e. the entire protocol). 
- _Pause_ the entire protocol in case of attacks or emergent vulnerabilities. 

## 3. Timelock to effective setting
To prevent acidental and sudden change in the protocol, we apply timelock regarding setting change until effectiveness, except protocol pausing.  
- All admin role changes require a timelock = 24h to be effective. In the mean time of waiting for the end of timelock, the protocol is still active with the previous admin addresses.
- All settings on OracleSetter require a timelock of 8 hours to be effective. In the mean time of waiting for the end of timelock, the protocol is still active with the old settings.  
- The settings "_Pause the entire protocol_" and on Bizsetter (_setFee_, _setKappa_, _setFeeto_, _setProtocolfee_) are **immediately effective**. 

![image](https://github.com/user-attachments/assets/f9ee760a-44cc-4d84-91d4-2da0f70eabec)


## Notes for unitests
- Test changes of 3 admin roles in 5-minute timelock
- Test settings on OracleSetter in 3-minute timelock
- Test role separation: 1 role 1 separate admin address which MUST-NOT overlap each other. That means BizSetter admin cannot set the param _SetOracleof_ of the OracleSetter role.
- Test the immediate pause. Aater that, no call, no tx, no state transition are allowed on all pools of BrownFi AMM.
- Test other settings on BizSetter which is immediately effective. 
- Test overlapping per setting before effectiveness: for example, submit 1st TX to change Kappa, while waiting to be effective, another TX is submitted to change Kappa. The protocol should care the latest TX. In the mean time, the protocol is still active with the old settings.  



