# Comparing price impact (slippage) between AMM pools: Uniswap V2 vs Uniswap V3 vs BrownFi  

## A little math of AMMs
**Uniswap V2** introduced constant product market making (CPMM) $x * y=k$, where the token price is defined by token reserve in the pool $p=y/x$. Consider the pool with token reserve $(10, 10)$, liquidity and swap constant $x* y=10* 10=100$ with the initial price $P= 1$. A swap out of $\Delta x$ of token X must provide $\Delta y$ of token Y in exchange. The constant product formula gives us $(x-\Delta x)(y+\Delta y)=k \Rightarrow \Delta y = \frac{k}{x-\Delta x}-y.$ 

[**Uniswap V3**](https://uniswap.org/whitepaper-v3.pdf) introduced the concentrated liquidity market making (CLMM), allowing us to find a curve limited by a price range such that it can serve the trade with optimal capital. Regarding Uniswap V3 model, a liquidity position is defined by both token reserve $(x, y)$ and a price range $[p_a, p_b]$. The liquidity and swap constant of V3 model reads $(x+\frac{L}{\sqrt{p_B}})(y+L\sqrt{p_a})=L^2$, where L is the virtual liquidity (comparable to the equivalent V2 model, i.e. $x* y =L^2$).   

For simplicity without loss of generality, we take price lower bound $p_a = 1.0001^{-n}$, price upper bound $p_b = 1.0001^n$, symmetrically. For $n=100, p_a= 1.0001^{-100} \approx 0.99$ and $p_b=1.0001^{100} \approx 1.01$, resulting $\pm1$% range. For $n=200, p_a= 1.0001^{-200} \approx 0.98$ and $p_b = 1.0001^{200} \approx 1.02$, resulting $\pm2$% range. Let consider 4 ranges of the same token reserve.     

- Uniswap V3 pool1 reserve (10,10), range [-9.5%, +10.5%] (i.e. $n=1000$), liquidity $(x+\frac{205}{1.0001^{500}})(y+\frac{205}{1.0001^{500}})\approx 205^2$, liquidity leverage (so capital efficiency) $\frac{205}{10}$ ~ X20.5
- Uniswap V3 pool2 reserve (10,10), range $\pm2$%, liquidity $(x+\frac{1000}{1.0001^{100}})(y+\frac{1000}{1.0001^{100}})\approx 1000^2$, capital efficiency $\frac{1000}{10}$ ~ X100
- Uniswap V3 pool3 reserve (10,10), range $\pm1$%, liquidity $(x+\frac{2000}{1.0001^{50}})(y+\frac{2000}{1.0001^{50}})\approx 2000^2$, capital efficiency $\frac{2000}{10}$ ~ X200

Because $x=y$ and $\frac{1}{\sqrt{p_B}}=\sqrt{p_a}$, we have $(x+\frac{L}{\sqrt{p_B}})^2=L^2, x+\frac{L}{\sqrt{p_B}}=L$. A swap out of $\Delta x$ of token X must provide $\Delta y$ of token Y in exchange. The CLMM  formula gives $(x+\frac{L}{\sqrt{p_B}}-\Delta x)(y+L\sqrt{p_a} +\Delta y)=L^2 \Leftrightarrow (L-\Delta x)(L +\Delta y)=L^2$ and hence $\Delta y=\frac{L^2}{L-\Delta x}-L$.  

[**BrownFi AMM**](https://mirror.xyz/0x64f4Fbd29b0AE2C8e18E7940CF823df5CB639bBa/5lSUhDUCCSZTxznxfkClDvLkwE3wr_swFCH_mT9fXLI) introduced a novel oracle-based AMM model. Given a token reserve $(x, y)$ and an amount $\Delta x$ of token X to be swapped out, trader must pay $\Delta y$ of token Y in exchange, simply defined by:

 - $\Delta y = P(1+\frac{R}{2})\Delta x$, where $P$ is the global price fed by oracle;
 - The term $\frac{R}{2}$ is price impact, where $R=\frac{K * \Delta x}{x-\Delta x}$;
 - Kappa ($K$) is the parameter controlling liquidity concentration on BrownFi's pools.

We consider four liquidity concentration on BrownFi AMM, controlled by $K_1=1, K_2=0.1, K_3=0.01$ and $K_4=0.001$. 

## What do traders care? 
The average trading price is the only thing a trader cares, defined by $\frac{\Delta y}{\Delta x}$. Smaller price impact, closer trading price to global price, better experience for average traders. Higher liquidity concentration makes lower price impact, smaller slippage, better trading experience. Standardizing token reserve (10,10), initial price $P=1$ for all pools of Uniswap and BrownFi's AMMs, we will compare price slippage $\frac{\Delta y}{\Delta x}-1$ between them. In prior to comparison, we find the intersections of the slippage curves (equivalently price impact curves). Each intersection represents order size (over the total of 10 token reserve) and percentage of slippage. 

| Slippage curves               | BrownFi $K_1=1$ | BrownFi $K_2=0.1$ | BrownFi $K_3=0.01$  | BrownFi $K_4=0.001$ |
| :----------------             | ------:         | ----:            | ----:             |----:     |
| Uniswap V2                    |                 |                  |                   |  |
| Uniswap V3 ($\pm10$%)         |                 |                  |$I_2(9.02, 4.6$%)  | $I_5(9.9, 5.05$%)  |
| Uniswap V3 ($\pm2$%)          |                 |                  | $I_1(5, 0.5$%)   | $I_4(9.5, 0.95$%)  |
| Uniswap V3 ($\pm1$%)          |                 |                  |                   |$I_3(9, 0.45$%)  |

![image](https://github.com/user-attachments/assets/c030d4bc-d486-430b-be11-b424a96bc544)

## Slippage (Price impact) comparison
- Easily see that Uniswap V2 causes greater slippage than all BrownFi pool (for $K<2). Particularly, if $K=2$, then BrownFi and Uniswap V2 are equivalent.
- Regading three Uniswap V3 pools, each BrownFi pools (K3 & K4) has lower slippage on the left side of the intersecting point, greater on the right, respectively. Particularly, BrownFi $K_4=0.001$ is mostly equivalent to Uniswap V3 range $\pm1$%.

Further on capital efficiency comparison, we have:

![image](https://github.com/user-attachments/assets/057e846c-5b0c-462c-9fcf-d284006bf1b7)

