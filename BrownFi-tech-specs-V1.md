_This is a technical document, describing all technical specilizations of BrownFi AMM version 1. The contents are math concept, protocol design, create new pools, add/remove liquidity, swap formulas, universial settings_

# 1. Math of BrownFi AMM
[**BrownFi**](https://mirror.xyz/0x64f4Fbd29b0AE2C8e18E7940CF823df5CB639bBa/5lSUhDUCCSZTxznxfkClDvLkwE3wr_swFCH_mT9fXLI) introduced a novel oracle-based AMM model. Given a pair (pool) of two tokens with liquidity reserve $(x, y)$ of token X and token Y, respectively. For an amount $\Delta x$ of token X to be swapped out, trader must pay $\Delta y$ of token Y in exchange, simply defined by:

 - $\Delta y = P(1+\frac{R}{2})\Delta x$, where $P$ is the global price fed by oracle;
 - The term $\frac{R}{2}$ is price impact, where $R=\frac{K * \Delta x}{x-\Delta x}$;
 - Kappa ($K$) is the parameter, controlling liquidity concentration on BrownFi's pools.

The Kappa ($K$) is limited by the range $0 < K \leq 2$. Smaller Kappa, greater liquidity concentration. Further on capital efficiency comparison, we have:

![image](https://github.com/user-attachments/assets/e2e4f23f-1449-4202-9c8e-a8cf4e8d511e)

# 2. Protocol design
BrownFi uses Uniswap V2 as a code base to custom and implement a novel AMM incorporating oracle price and a new trading mechanism. Some critical changes are applied and implemented at:

- The pair contract (liquidity pool contract);
- The router contract

# 3. Create new pools
Create a new liquidity pool is similar to the design of Uniswap V2 model with some additional changes to adapt with oracle-based design of BrownFi AMM.

- Initiate arbitrary amounts of the token pair $(x, y)$.
- Set price feed of the token pair.
- Set liquidity concentration, Kappa parameter.
- Set the base and the quote tokens. 
See an example on BrawnFi AMM deployed on Metis mainnet https://andromeda-explorer.metis.io/address/0x36D65d716093344B05c961D65b286fa13dde6f5B?tab=write_contract

# 4. Add/remove liquidity
Add/remove liquidity on BrownFi AMM's pools follows the convention of Uniswap V2 model. Liquidity position is fungible and based on ERC20 token standard. Add/remove liquidity must be directly proportional to the existing reserve ratio of the pool. 

## 4.1. Add LP
- The current reserves are $(x,y)$, and the existing LP tokens are $E=totalLPtokens, E>0$.  
- Assume that an LPer enters an amount $x'$ of token X, then the new reserve will become $x \mapsto x + x'$, the adding ratio is $a= \frac{x'}{x}.$ Thus, $y' = a \times y = \frac{x'}{x} \times y$ is the corresponding amount of token Y that the LPer must add along with the amount $x'$ of token X. The new pool reserve will be $(x + x', y + y')$, proportionally with the previous reserve (i.e. $\frac{x}{y}=\frac{x + x'}{y + y'}=\frac{x'}{y'}$).
- The pool share of the LP is $b=\frac{a}{1+a}=\frac{x'}{x+x'}$ whichever easier, hence minting (issuing) an amount of LP tokens $\frac{b*E}{1-b}=E\times \frac{x'}{x}$ whichever easier.

## 4.2. Remove LP 
An LPer wants to remove his liquidity provision, i.e. redeeming/burning his LP tokens for the pairing assets.  
- The current reserves are $(x,y)$, and  the total amount $E$ of existing LP tokens, $E>0$.  
- He has an amount $e$ of LP tokens, hence his LP share is $e/E$.
- When burning $e$ LP tokens, he will receive $\frac{e}{E}\times x$ token X and $\frac{e}{E}\times y$ token Y. Trading fee is automatically accrued in the LP share.

# 5. Swap formulas

## 5.1. Backward computation (get amountIN)
> there are two cases of a trade:   
> - (**B1-buy**) take OUT $Dx$ amount of token X => calculate _put IN_ amount $Dy$ of token Y,   
> - (**B2-sell**) take OUT $Dy$ amount of token Y => calculate _put IN_ amount $Dx$ of token X.

### 5.1.1. If (**B1-buy**): traders enter expecting _amountOUT_ $Dx$ of token X.  
1. Adding fee $dx=Dx *(1+fee)$. This is the _pseudo amountOUT_ used to compute price impact, trading price below. 
2. CHECK $10 * dx < 9 * x$, otherwise invalid.
3. Compute price impact $R=\frac{K*dx}{(x-dx)}$.
4. Token X price fed by oracle $p=P_{X/Y}$ (i.e. quoted by Y). 
5. Compute average trading price $p_t = p * (1 + R/2)$. 
6. Compute _amountIN_ $dy=dx * p_t$, this is also the **actual** _amountIN_ $Dy = dy$. 
7. Swap  $Dy$ amountIN of token Y for  $Dx$ amountOUT of token X.
8. The post-trade pool state is $(xt=x - Dx, yt=y + Dy).$
9. Pool verification $(x-dx)P + (y+dy) \geq (xP + y) + \frac{P * K * dx * dx}{2(x-dx)}$
 
### 5.1.2. If (**B2-sell**): traders enter expecting _amountOUT_ $Dy$ of token Y.  
1. Adding fee $dy=Dy *(1+fee)$. This is the _pseudo amountOUT_ used to compute price impact, trading price below.
2. CHECK $10 * dy < 9 * y$, otherwise invalid.
3. Compute price impact $R=\frac{K* dy}{(y-dy)}$. 
4. Token Y price fed by oracle  $p=P_{Y/X}=\frac{1}{P_{X/Y}}$. (i.e. quoted by X). 
5. Compute average trading price $p_t = p * (1 + R/2) = \frac{1}{P_{X/Y}} * (1 + R/2)$. 
6. Compute _amountIN_ $dx=dy * p_t$, this is also the **actual** _amountIN_ $Dx = dx$. 
7. Swap  $Dx$ amountIN of token X for  $Dy$ amountOUT of token Y.
8. The post-trade pool state is updated as $(xt=x + Dx, yt=y - Dy)$.
9. Pool verification $(x+dx)P + (y-dy) \geq (xP + y) + \frac{K * dy * dy}{2(y-dy)}$.


## 5.2) Forward computation (get amountOUT)

There are two cases of a trade:
- (B1-buy) enter _amountIN_ of token Y => calculate _amountOUT_ of token X,  
- (B2-sell) enter _amountIN_ of token X => calculate _amountOUT_ of token Y.

### 5.2.1. (B1-buy) 
Traders enter **actual** _amountIN_ $Dy$ of token Y => find **actual** _amountOUT_ $Dx$ of token X.  
1. Token X price fed by oracle $p=P_{X/Y}$ (i.e. quoted by Y). 
2. Compute _pseudo_ amountOUT $dx = \frac{px+Dy - \sqrt{(px-Dy)^2+2pKxDy}}{p(2-K)}$ if $K<2$. Otherwise, for $K=2$, we have $dx=\frac{xDy}{px+Dy}$, and the **actual** amountOUT is $Dx=dx/(1+ fee)$. 
3. Return to Steps (1 to 6) of Section 5.1.1 in Backward computation.  


### 5.2.2. (B1-sell) 
Traders enter **actual** _amountIN_ $Dx$ of token X => find _amountOUT_ $Dy$ of token Y.  
1. Token X price fed by oracle  $p=P_{X/Y}$. (i.e. quoted by Y).  
2. Compute _pseudo_ amountOUT $dy = \frac{pDx+y - \sqrt{(pDx-y)^2+2pKyDx}}{(2-K)}$ if $K<2$. Otherwise, for $K=2$, we have $dy=\frac{pyDx}{pDx+y}$, and the **actual** amountOUT is $Dy=dy/(1+fee)$.  
3. Return to Steps (1 to 6) of Section 5.1.2 in Backward computation.


## 5.3. Computing flow diagram

This flow is REGULAR on BrownFi AMM by math, and suggested. It always gives exact amountOUT (not exact amountIN). https://drive.google.com/file/d/1LkCCukacMpgUdiXJLXljp-AUGA3IKxPu/view?usp=sharing 
![image](https://github.com/user-attachments/assets/b8f32df1-8c78-4a92-b52d-2691ec1fdbce)

# 6. Universial settings
The following settings are universially applied for all BrownFi AMM's pools.  

- **Kappa** (the parameter controlling liquidity concentration) is set to be $K=0.001$, thus liquidity concentration is similar to Uniswap V3 range $\pm1$%.  
- **Trading fee** is applied for _amountOUT only_, and $fee = 0.0025$, i.e. 0.25%.
- **Protocol fee** is currently zero, i.e. NO fee split for the developer.


