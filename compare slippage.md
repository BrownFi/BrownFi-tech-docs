# Comparing price impact (slippage) between AMM pools: Uniswap V2 vs Uniswap V3 vs BrownFi  

## A little math of AMMs
**Uniswap V2** introduced constant product market making (CPMM) $x * y=k$, where the token price is defined by token reserve in the pool $p=y/x$. Consider the pool with token reserve (10,10), liquidity and swap constant $x* y=10* 10=100$ with the initial price $P= 1$. A swap out of $\Delta x$ of token X must provide $\Delta y$ of token Y in exchange. The constant product formula gives us $(x-\Delta x)(y+\Delta y)=k \Rightarrow \Delta y = \frac{k}{x-\Delta x}-y.$ 

[**Uniswap V3**](https://uniswap.org/whitepaper-v3.pdf) introduced the concentrated liquidity market making (CLMM), allowing us to find a curve limited by a price range such that it can serve the trade with optimal capital. Regarding Uniswap V3 model, a liquidity position is defined by both token reserve $(x, y)$ and a price range $[p_a, p_b]$. The liquidity and swap constant of V3 model reads $(x+\frac{L}{\sqrt{p_B}})(y+L\sqrt{p_a})=L^2$, where L is the virtual liquidity (comparable to the equivalent V2 model).   

For simplicity without loss of generality, we take price lower bound $p_a = 1.0001^{-n}$, price upper bound $p_b = 1.0001^n$, symmetrically. For $n=100, p_a= 1.0001^{-100} \approx 0.99$ and $p_b=1.0001^{100} \approx 1.01$, resulting +/-1% range. For $n=200, p_a= 1.0001^{-200} \approx 0.98$ and $p_b = 1.0001^{200} \approx 1.02$, resulting +/-2% range. Let consider 4 ranges of the same token reserve.     

- Uniswap V3 pool1 reserve (10,10), range [-9.5%, +10.5%] (i.e. $n=1000$), liquidity $(x+\frac{1005.06}{1.0001^{500}})(y+\frac{1005.06}{1.0001^{500}})\approx 205.05^2$, liquidity leverage (so capital efficiency) ~ X2.05
- Uniswap V3 pool2 reserve (10,10), range +/-2%, liquidity $(x+\frac{1005.06}{1.0001^{100}})(y+\frac{1005.06}{1.0001^{100}})\approx 1005.06^2$, capital efficiency ~ X100.5
- Uniswap V3 pool3 reserve (10,10), range +/-1%, liquidity $(x+\frac{2005.1}{1.0001^{50}})(y+\frac{2005.1}{1.0001^{50}})\approx 2005.1^2$, capital efficiency ~ X200.5

A swap out of $\Delta x$ of token X must provide $\Delta y$ of token Y in exchange. The CLMM  formula gives $(x+\frac{L}{\sqrt{p_B}}-\Delta x)(y+L\sqrt{p_a+\Delta y})=L^2$. Because $x=y$ and $\frac{1}{\sqrt{p_B}}=\sqrt{p_a}$, we have $\Delta y=\frac{L}{x+\frac{L}{\sqrt{p_B}}-\Delta x}-x-\frac{L}{\sqrt{p_B}}$.  

[**BrownFi AMM**](https://mirror.xyz/0x64f4Fbd29b0AE2C8e18E7940CF823df5CB639bBa/5lSUhDUCCSZTxznxfkClDvLkwE3wr_swFCH_mT9fXLI) introduced a novel oracle-based AMM model. Given a token reserve $(x, y)$ and an amount $\Delta x$ of token X to be swapped out, trader must pay $\Delta y$ of token Y in exchange, simply defined by:

 - $\Delta y = P(1+\frac{R}{2})\Delta x$, where $P$ is the global price fed by oracle;
 - The term $\frac{R}{2}$ is price impact, where $R=\frac{K * \Delta x}{x-\Delta x}$;
 - Kappa ($K$) is the parameter controlling liquidity concentration on BrownFi's pools.

We consider four liquidity concentration on BrownFi AMM, controlled by $K_1=1, K_2=0.1, K_3=0.01$ and $K_4=0.001$. 

## What do traders care? 
The average trading price is the only thing a trader cares, defined by $\frac{\Delta y}{\Delta x}$. Smaller price impact, closer trading price to global price, better experience for average traders. Higher liquidity concentration makes lower price impact, smaller slippage, better trading experience. Standardizing token reserve (10,10), initial price $P=1$ for all pools of Uniswap and BrownFi's AMMs, we will compare price slippage $\frac{\Delta y}{\Delta x}-1$ between them. In prior to comparison, we find the intersections of the slippage curves (equivalently price impact curves).  

| Slippage curves               | BrownFi $K_1=1$   | BrownFi $K_2=0.1$ | BrownFi $K_3=0.01$  | BrownFi $K_4=0.001$ |
| :----------------             | ------:         | ----:            | ----:             |----:     |
| Uniswap V2                    |                 |                  |                   |  |
| Uniswap V3 ($\pm10$%)         |                 |                  |$I_2$(9.02,0.046)  | $I_5$(9.9, 0.0505)  |
| Uniswap V3 ($\pm2$%)          |                 |                  | $I_1$(5,0.005)    | $I_4$(9.5, 0.0095)  |
| Uniswap V3 ($\pm1$%)          |                 |                  |                   |$I_3$(9, 0.0045)  |

![image](https://github.com/user-attachments/assets/c030d4bc-d486-430b-be11-b424a96bc544)

## Slippage (Price impact) comparison
- Easily see that Uniswap V2 causes greater slippage than all BrownFi pool (for $K<2). Particularly, if $K=2$, then BrownFi and Uniswap V2 are equivalent.
- Regading three Uniswap V3 pools, each BrownFi pools (K3 & K4) has lower slippage on the left side of the intersecting point, greater on the right, respectively.

Further on capital efficiency comparison, we have:
![image](https://github.com/user-attachments/assets/212feaf6-e934-47a0-9815-800208439b15)

