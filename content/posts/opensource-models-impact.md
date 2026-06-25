---
date: 2026-06-21
title: The impact of good open-source models
---
GLM 5.2 was released few days ago and It's already gaining a lot of traction. As I'm thinking about the memory shortage I thought - **Is this just going to increase the memory and compute demand?** Now you can run very smart open models cheaper because it's open-source thus less margin on tokens. This means that you can pay for more tokens. More tokens means more memory was used in producing them. If scaling compute is what is what always works, scaling memory goes hand in hand. 

DeepSeek V4 showed novel models of compressed attention that reduced the KV cache by more than 10x.
But GLM 5.2 did not implement such a aggressively compressed attention and achieved better results. This was probably caused by better training and other things?
**So is compressing the KV cache bad?**
The compression is not lossless. You lose signal every time you compress the data. On long horizon, complex tasks, nuances become important. You can make a case that losing that nuance is NOT worth it. Errors keep piling up quickly. Just use more test time compute. Compute is not that expensive for you to sacrifice inteligence.  
But you can take the opposite stance. You might want to push the cost of tokens lower and train/adjust the model architecture around compressed KV cache. Maybe you can find a way to amplify the signal? It's been shown, that you can use lower precision, train the model with this lower precision (NVFP4 or NVFP8) and sacrifice only little performance. Nvidia is already designing their GPUs around lower precision architectures so I expect that they know why. Working closely with SOTA labs gives you good idea about their needs. I wouldn't be surprised of GPT 5.5 is in NVFP8 or NVFP4 based on the reasoning level.  

Overall I think that the push will be towards optimizing costs of tokes. This will be as important as intelligence. Volume over SOTA quality will be become more relevant. 
