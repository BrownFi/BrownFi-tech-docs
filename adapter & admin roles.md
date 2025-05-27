# New design for the price adapter & admin roles
To ensure highest security for BrownFi AMM, we design an oracle price adapter to fetch price for all pools, and separate admin (setter) roles regarding various settings in the BrownFi protocol. 

## Oracle price adapter

## Admin roles
We define three admind (setter) roles associated with certain param settings: oracle setter, . The 3 roles are independent. After deployment, the deployer must transfer 

Oracle setter:
- setDecimalShift
- setPricefeed
- setQTI

Biz setter:
- setFee
- setFeeto
- setProtocolfee
- setKappa

Pauser: 
- Pause the entire protocol

![image](https://github.com/user-attachments/assets/aeac3395-a072-4cad-b808-d12e44d36783)


