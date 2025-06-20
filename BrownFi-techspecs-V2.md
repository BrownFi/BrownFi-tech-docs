_This is a technical document, describing all technical specilizations of BrownFi AMM version 2. The contents are math concept, protocol design, create new pools, add/remove liquidity, swap formulas, fee & other protocol settings_  
[BrownFi V1](https://github.com/BrownFi/BrownFi-tech-docs/blob/main/BrownFi-techspecs-V1.md) (current **Beta Production**) is audited by Verichain, see [audit report](https://github.com/verichains/public-audit-reports/blob/main/Verichains%20Public%20Audit%20Report%20-%20BrownFi%20AMM%20Smartcontracts%20-%20v1.0.pdf)

**We compare the two versions.** While using the same Codebase of [Uniswap V2](https://github.com/Uniswap/v2-core) and its design, the two versions of BrownFi AMM protocol have some diffentiations.         

| Essentials             | BrownFi V1 | BrownFi V2 | 
| :----------------      | ------:         | ----:            | 
| Price mechanism       |  oracle + price impact   | oracle + skewness + price impact  | 
| Pool creation         | Permissioned            |  permissionless   |
| LP share              | ERC20 token           | ERC20 token       |
| Add LP                | by token ratio     | 50-50 by dollar-value      |
| Remove LP             | by token ratio     | by token ratio        |
| Trading fee          | applied on amountOUT      | applied on amountIN   | 
| Protocol fee         | NO (not implemented fee split yet)   | YES (implemented fee split) |
| Order size           | no limit            | $\leq 80$% reserve limited by pair contract  | 
| Oracle adapter        | single source       | 2 sources (1 MUST, 1 optional)  |
| Admin role           | one for all         | separate 3 roles     |

Read the design of oracle price adapter and admin roles [HERE](https://github.com/BrownFi/BrownFi-tech-docs/blob/main/adapter%20&%20admin%20roles.md). 

# 1. Math of BrownFi AMM
[**BrownFi**](https://mirror.xyz/0x64f4Fbd29b0AE2C8e18E7940CF823df5CB639bBa/5lSUhDUCCSZTxznxfkClDvLkwE3wr_swFCH_mT9fXLI) introduced a novel oracle-based AMM model. Given a pair (pool) of two tokens with liquidity reserve $(x, y)$ of token X and token Y, respectively. For an amount $\Delta x$ of token X to be swapped out, trader must pay $\Delta y$ of token Y in exchange, simply defined by:

 - $\Delta y = P(1+\frac{R}{2})\Delta x$, where $P=P_{X/Y}$ is the global price of X, quoted by Y, fed by oracle adapter;
 - The term $\frac{R}{2}$ is price impact, where $R=\frac{K * \Delta x}{x-\Delta x}$;
 - Kappa ($K$) is an adjustable and configurable parameter, controlling liquidity concentration on BrownFi's pools.

The Kappa ($K$) is limited by the range $0.0001 \leq K \leq 2$. Smaller Kappa, greater liquidity concentration. 

# 2. Oracle price and skewness
Basically, all pool of BrownFi AMM need oracle price, fed by the [oracle adapter](https://github.com/BrownFi/BrownFi-tech-docs/blob/main/adapter%20&%20admin%20roles.md). In V2, BrownFi introduces pool skewness (share imbalance) adding to oracle price before swap. This is to incentivize trades toward balancing between the two token reserves, otherwise discourage. 

Consider the pool reserve $(x, y)$ with oracle price $P^o_X, P^o_Y$, compute the absolute skewness of the pool $S=\frac{\|xP^o_X-yP^o_y\|}{xP^o_X + yP^o_y}$.  
- If $xP^o_X \geq yP^o_y$, then we compute new price with skewness $P_X=P^o_X(1-\lambda* S)$ and $P_Y=P^o_Y(1+\lambda*S)$;
- If $xP^o_X \leq yP^o_y$, then we compute new price with skewness $P_X=P^o_X(1+\lambda* S)$ and $P_Y=P^o_Y(1-\lambda*S)$.

Here, per pool, we introduce a new configurable param lambda $(\lambda)$ defined by max imbalance target (80-20) with default $\lambda=0$ and limit range $[0, 1]$. The following diagram indicates where we need skewness (for swap) or NOT (for add LP).  
![image](https://github.com/user-attachments/assets/e61a5b86-fedf-4965-ac8e-8caa5cc2e2da)

# 3. Liquidity provision
## 3.1. Create new pools
Create a new liquidity pool is permissionless similarly to the design of Uniswap V2 model with some additional changes to adapt with oracle-based design of BrownFi AMM.

- Initiate arbitrary amounts of the token pair $(x, y)$.
- Set oracle price feed of the token pair.
- Set liquidity concentration, Kappa parameter, requiring limits $0.0001 \leq K \leq 2$.

## 3.2. Add liquidity

In V1, LP is added/removed such that token ratio in the pool is unchanged.
- **Pros**: strictly compatible with Uniswap V2 design of the core contract.
- **Cons**: LPers cannot rebalance liquidity between two tokens in the pool. Thus, imbalance of the pool inventory is maintained,
causing higher price impact on the less side and smaller price impact on the larger side. This is different to UniswapV2 model as its inventory is always 50-50 balanced by dollar value.  
![image](https://github.com/user-attachments/assets/ff71695b-ce81-4f1c-beeb-1b5ea136ea28)

We want to  extend the framework to reduce the cons, improving LP UX. This can be done by converting assets to **USD value** (or a pre-defined quote asset), then computing LP share. Price-feed is required to compute dollar value of the pool and new adding LP.  

-  Assume that the total supplying LP tokens are $E=totalLPtokens, E>0$.
-  Assume that at a specific time $t$, the pool has token reserve $(x, y)$ with corresponding dollar price $P^o_X, P^o_Y$ fed by oracle adapter **_without sknewness_** factor, and the pool value $V= x * P^o_X + y * P^o_Y$.
-  Bob wanna add LP amount of $(x', y')$ with USD-value $B = x' * P^o_X + y' * P^o_Y$, required $x' * P^o_X = y' * P^o_Y$, i.e. 50-50 liquidity provision, equivalently $B = 2 * \min(x' * P^o_X, y' * P^o_Y)$.
-  Bob's liquidity share is $s=\frac{B}{V+B}$ and we mint an amount of new LP token by $\frac{newLP}{E+newLP}=\frac{B}{V+B}=s$, hence $newLP=\frac{sE}{1-s}$. We must have $\frac{newLP}{E}=\frac{B}{V}$ or **$newLP=E\times \frac{B}{V}$.**
  
### Trick at LP initiation
When someone initiates a pool, we must pass the ZERO state. Assume the we initiate $x>0$ token X and $y>0$ token Y to create a new trading pair (i.e. a new liquidity pool), with corresponding dollar oracle-price $P^o_X, P^o_Y$, and the initiating value $B= x * P^o_X + y * P^o_Y$. Then we do:

- mint the minimum amount of LP tokens = 1000 and send to DEAD address (i.e. burn to 0x0...00);
- mint the initiating amount of LP token $E_0-1000=B-1000$ for the pool creator, i.e. setting $E_0/V_0=1$.

## 3.3. Remove LP 
-  Assume that the total supplying LP tokens are $E=totalLPtokens, E>0$.
-  Assume that at a specific time, the pool has token reserve $(x, y)$ with corresponding dollar price $P_X, P_Y$, and the pool value $V= x * P_X + y * P_Y$.
-  Bob wanna remove LP share by burning an amount of $s$ LP tokens to receive $(x', y')$ token X and token Y, repectively. We must ensure the equality by USD-value $\frac{s}{E}=\frac{B}{V}$ where $B = x' * P_X + y' * P_Y$. This equation may have infinite number of solutions. Thus, additionally, we require a proportional withdrawal on both sides of the pool reserve, i.e. $\frac{x'}{x}=\frac{y'}{y}$. 
-  The solution presenting amounts of LP withdrawal is $x' = \frac{s}{E}x, y' = \frac{s}{E}y$, satisfying all our requirements (no need price update here).

# 4. Pool state verification

Per swap, the pool must be guaranteed that post-trade inventory (**without fee**) is greater or equal pre-trade inventory plus premium (price impact). The verification method to ensure a safe accounting for LP regardless computing process (with rounding and routing), meaning LP always get trading fee and price impact premium as intented. Additionally, inventory verification apply **skewness price** to encourage pool balancing by trades.   

**NOTE**: Trading fee is applied on amountIN only. We have _actual amountIN = pseudo amountIN * (1 + fee); pseudo amountIN = actual amountIN / (1 + fee)_.

Assume that the pair of token X, token Y has **price with skewness** $P_X=P_{X/USD}, P_Y=P_{Y/USD}$ both quoted by US dollar and computed by formula in Section 2.  


## 4.1. BUY verification
- Pre-trade inventory $xP_X + yP_Y$. 
- For, actual amountOUT $dx$ and pseudo amountIN $dy$ (**without fee**), we compute post-trade inventory $(x-dx)P_X + (y+dy)P_Y$.  

Pool contract **verifies** the two condition:
- $10 * dx \leq 8 * x$
- MUST hold $(x-dx)P_X + (y+dy)P_Y - P_X\frac{K * dx * dx}{2(x-dx)} \geq (xP_X + yP_Y)$.

## 4.2. SELL verification
- Pre-trade inventory $xP_X + yP_Y$. 
- For, pseudo amountIN $dx$ (**without fee**) and actual amountOUT $dy$, we compute post-trade inventory $(x+dx)P_X + (y-dy)P_Y$.  

Pool contract **verifies** the two condition:
- $10 * dy \leq 8 * y$
- MUST hold $(x+dx)P_X + (y-dy)P_Y  - P_Y\frac{K * dy * dy}{2(y-dy)} \geq (xP_X + yP_Y)$.

## 4.3. Language of amount-IN/OUT
Without caring on buy/sell, we can re-state the inventory verification formula in the language of unified amountIN and amountOUT as followed:  
$$(Reserve_{OUT} - amount_{OUT})*Price_{OUT} + (Reserve_{IN} + amount_{IN})*Price_{IN} - Price_{OUT} * \frac{K * (amount_{OUT})^2}{2(Reserve_{OUT} - amount_{OUT})} \geq Reserve_{OUT} * Price_{OUT} + Reserve_{IN} * Price_{IN}$$

# 5. Swap formulas
BrownFiV2 router computes amountIN and amountOUT for swap as mostly similar as V1, except trading fee is **applied** for **amountIN** (instead of _amountOUT_ in V1). The price is fed by oracle adapter with **skewness** factor in Section 2, not pure oracle price. 

## 5.1. Backward computation (get amountin) 
> there are two cases of a trade:   
> - (**B1**) take OUT $dx$ amount of token X => calculate _put IN_ amount $dy$ of token Y,   
> - (**B2**) take OUT $dy$ amount of token Y => calculate _put IN_ amount $dx$ of token X.

### 5.1.1. If (**B1**): enter/give _amountOUT_ $dx$ of token X. 
1. CHECK $10 * dx \leq 8 * x$, otherwise exceed limit (failed).
2. Compute price impact $R=K*dx/(x-dx)$
3. Token X price fed by oracle $P=P_{X/Y}$ (i.e. quoted by Y). 
4. Compute average trading price $Pt = P * (1 + R/2)$.
5. Compute _pseudo amountIN_ $dy=dx * Pt$.
6. **Verify** post-trade inventory vs pre-trade inventory (**without fee**) $(x-dx)P_X + (y+dy)P_Y - P_X\frac{K * dx * dx}{2(x-dx)} \geq (xP_X + yP_Y)$.
7. Add fee to _amountIN_, compute actual _amountIN_ $Dy = dy*(1+fee)$. => check trader's token balance to ensure the trader has sufficient amount of token X for swap.
8. Swap  $Dy$ amountIN of token Y for  $dx$ amountOUT of token X.

> The post-trade pool state is $(xt=x - dx, yt=y + Dy).$   

### 5.1.2. If (**B2-sell**): traders enter expecting _amountOUT_ $dy$ of token Y.  
1. CHECK $10 * dy \leq 8 * y$, otherwise exceed limit (failed).
2. Compute price impact $R=\frac{K* dy}{(y-dy)}$. 
3. Token Y price fed by oracle  $P=P_{Y/X}=\frac{1}{P_{X/Y}}$. (i.e. quoted by X). 
4. Compute average trading price $P_t = P * (1 + R/2) = \frac{1}{P_{X/Y}} * (1 + R/2)$. 
5. Compute _pseudo amountIN_ $dx=dy * P_t$.
6. **Verify** post-trade inventory vs pre-trade inventory (**without fee**) $(x+dx)P_X + (y-dy)P_Y - P_Y\frac{K * dy * dy}{2(y-dy)} \geq (xP_X + yP_Y)$.
7. Add fee to _amountIN_, compute actual _amountIN_  $Dx = dx*(1+fee)$. => check trader's token balance to ensure the trader has sufficient amount of token X for swap.
8. Swap  $Dx$ amountIN of token X for  $dy$ amountOUT of token Y.

> The post-trade pool state is updated as $(xt=x + Dx, yt=y - dy)$.   

## 5.2. Forward computation (get amountOUT)

There are two cases of a trade:
- (B1-buy) enter _amountIN_ of token Y => calculate _amountOUT_ of token X,  
- (B2-sell) enter _amountIN_ of token X => calculate _amountOUT_ of token Y.

Visit the detail of finding the [solutions HERE](https://github.com/BrownFi/BrownAMM-intro/blob/main/solving-quadratic.md). Note that $K$ is limited within range $0.001 \leq K \leq 2$. and we have one unique solution per case $K=2$ or $K<2$.   

Token price fed by oracle $P_X, P_Y$ (i.e. quoted by US dollar).  

### 5.2.1. (B1-buy) 
Traders enter **actual** _amountIN_ $Dy$ of token Y => find **actual** _amountOUT_ $Dx$ of token X.  
1. Compute _pseudo_ amountIN $dy=Dy/(1+fee)$.
2. Compute actual amountOUT $dx = \frac{xP_X+dyP_Y - \sqrt{(xP_X-dyP_Y)^2+2P_X P_Y * K x dy}}{P_X(2-K)}$ if $K<2$. Otherwise, for $K=2$, we have $dx=\frac{xdyP_Y}{xP_X+dyP_Y}$. 
3. Return to Steps (1 to 6) of Section 5.1.1 in Backward computation.  

### 5.2.2. (B1-sell) 
Traders enter **actual** _amountIN_ $Dx$ of token X => find _amountOUT_ $Dy$ of token Y.  
1. Compute _pseudo_ amountIN $dx=Dx/(1+fee)$.
2. Compute actual amountOUT $dy = \frac{P_Xdx+yP_Y - \sqrt{(P_Xdx-yP_Y)^2+2P_XP_Y * K y dx}}{P_Y(2-K)}$ if $K<2$. Otherwise, for $K=2$, we have $dy=\frac{ydxP_X}{dxP_X+yP_Y}$.  
3. Return to Steps (1 to 6) of Section 5.1.2 in Backward computation.


# 6. Protocol fee (splitted for the developer)

Per swap, LPers earn premium fee (derived from price impact) and trading fee. However, only trading fee is partially splitted into protocol fee for the protocol developer.   

**Implement fee split at the core contract**, so dev earns fee for all routers (including aggregators).

**Requirement**: dollar-valued LP

**Solution**: mint LP token for dev per swap according to the dollar-amount of protocol fee. 

**Computating protocol fee**

-  The price is pure oracle price **withOUT** skewness.
-  Assume that the total supplying LP tokens are $E=totalLPtokens, E>0$.
-  Assume that the swap is given by an actual amountIN whose pseudo amountIN (without trading fee) is $pseudoAmountIN = \frac{actualAmountIN}{1+fee}$. Thus, the trading fee amount is $actualAmountIN - \frac{actualAmountIN}{1+fee} = \frac{actualAmountIN}{1+fee}* fee$.
- Given the oracle price of token-IN is $P_{tokenIN}$, we compute $protocolFee = \frac{actualAmountIN}{1+fee}* fee * m * P^o_{tokenIN}$ in dollar value. Where $0\leq m \leq 1$ is a configurable param with default $m=0.1$ (i.e. 10% of LP revenue).
- Mint an amount of new LP token by $newLP=E * \frac{protocolFee}{PoolValue_{posttrade}} = E * \frac{protocolFee}{x_1P^o_X + y_1P^o_Y}$ then transfer to the dev wallet (_FeeTo_ setting). This should be compatible with  [adding new LP issue](https://github.com/orgs/BrownFi/projects/1/views/1?pane=issue&itemId=81293597) and equivalently to [LP computation](https://github.com/BrownFi/BrownAMM-dev/blob/main/compute-LP.md). 

> The price to compute the dev LP is **skewness price** in Section 2.   


# 6. Admin roles and protocol settings
BrownFi is an oracle-based AMM, thus, it needs regular parameter configurations by the protocol admins. We define admin roles and their associated settings [(details HERE)](https://github.com/BrownFi/BrownFi-tech-docs/blob/main/adapter%20&%20admin%20roles.md). Nevertheless, NO admin role can withdraw liquidity, keeping the principle of noncustodial AMM. 

## 6.1. Parameter settings
The following parameters are applied for all BrownFi AMM's pools and can be re-configurated by the protocol admins.
- **Kappa** (the parameter controlling liquidity concentration) is limited in the range $0.0001 \leq K \leq 2$. The defaut is set to be $K=0.01$, thus liquidity concentration (or liquid depth) is similar to Uniswap V3 range $\pm2$%.
- **Lambda** param is to control the impact of  price with skewness, limited in range $[0,1]$.
- **Trading fee** is applied for _amountIN only_, and $fee = 0.003$, i.e. 0.3%. The limited range is $0 \leq fee \leq 0.1$, i.e. cap by 10%. Trading fee is implemented at the core contract, i.e. pair contract when verifying inventory using amountIN and amountIN_withoutfee.
- **Protocol fee** $m$ (default $m=0.1$) is a configurable param, where $0\leq m \leq 1$. Protocol fee receipient is set by _feeTo_ function on Factory contract.
- **MinPriceAge**: the minimal valid time for an updated price (i.e. invalid price if exceeding the minimal time-period).

## 6.2. Three admin roles

**OracleSetter** acts on a specific pool. Two pools may have two separated Oracle Setters who are responsible for the 3 following settings: 
- _SetPriceOracle_: set adapter contract address
- _SetOracleof_: set price feedID
- _SetMinPriceAge_: set the minimal valid time for an updated price (i.e. invalid price if exceeding the minimal time-period)

**BizSetter** acts on a specific pool. Two pools may have two separated Bussiness Setters who are responsible for the 4 following settings: 
- _setFee_: set transaction fee
- _setKappa_: set parameter which controls liquidity concentration
- _setLambda_: set Lambda param to compute  price with skewness
- _setFeeto_: set receiver of protocol fee (receiving diluted LP tokens per swap) which belongs to the developer
- _setProtocolfee_: set the share percentage of trading fee which will belong to the developer

**Pauser** acts on all pools (i.e. the entire protocol). 
- _Pause_ the entire protocol in case of attacks or emergent vulnerabilities. 

**NOTE** that all 3 roles above are asigned by the factory admin after protocol deployment. 
After deploying the protocol, the deployer MUST transfer the admin roles to multisig wallets [(details of admin roles HERE)](https://github.com/BrownFi/BrownFi-tech-docs/blob/main/adapter%20&%20admin%20roles.md).  Nevertheless, NO admin role can withdraw liquidity, keeping the principle of noncustodial AMM. 

# Testcases
2 sheets with skewness https://docs.google.com/spreadsheets/d/1Smc8OTL4EaiyXJ6chdxViT3GJI5w3Fpz/edit?usp=sharing&ouid=101802233739943862069&rtpof=true&sd=true
