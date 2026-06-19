---
title: Does it make sense?
date: 2026-06-19
---

A lot of time passed and I wasn't writing or coding much. I simply ran out of ideas and motivation to pursue anything.  
I've been interested in the LLM space for a while and now I'm trying to understand the market behind it. I may be wrong, if so, please tell me.

# My silly view of the market
I'm not an expert, I'm a noob and you should do your own research. So let's start.  

**What are these markets anyway?**  
From my understanding, buying stocks is like lending money. Company needs few billies? Stinky bank doesn't want to loan you billies? 
Ask the people (market)! They have ton of cash and it's not like they have good ways to use it. For every dollar they lend me, I will pay back some extra. Trust me, I will make so much money that the returns will be large!  

I know it's probably more complicated but that's how I see it. Effective allocation of capital or something.  

Right now, there is a lot of excitement about LLMs. You are probably sick of it. A shoe company announces pivot to AI? Stocks go up by 580% (real btw: [Allbirds shares soar 580% after pivot from shoes to AI
](https://www.bbc.com/news/articles/c98mrepzgj7o))

<img src="/images/shoe-company.png" alt="shoe company stock chart" style="max-width: 500px; width: 100%;">.

Companies in this AI/semiconductor/chip sector trade for very high P/E ratios and the market is pricing in revenue 2-3 years in the future. You can look at this and and feel a little strange. 

**Does any of this make sense?**  
Are you "AGI pilled"? If the answer is YES, you probably made a lot of money so far and it all makes perfect sense to you. You also probably hope, that the number keeps going up. If you answered NO, you are a bit bitter that you missed on the gains and you might be calling this a bubble. 

Surely they can't capture the trillions of dollars of revenue like they say, right? 

<img src="/images/hyperscaler-ai-capex.png" alt="Hyperscaler AI Capex" style="max-width: 700px; width: 100%;">

| Camp                               | Current 2026 claim                                                                                                                                                                                                              |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Gartner spending**               | Worldwide AI spending forecast at **$2.52T in 2026**, up 44% YoY. [Gartner](https://www.gartner.com/en/newsroom/press-releases/2026-1-15-gartner-says-worldwide-ai-spending-will-total-2-point-5-trillion-dollars-in-2026)      |
| **Goldman infrastructure**         | Hyperscaler AI capex projected around **$757B in 2026**, rising to **$920B in 2027**. [Business Insider summary](https://www.businessinsider.com/stocks-to-buy-ai-capex-beneficiaries-tech-investing-goldman-sachs-2026-6)      |
| **Skeptic math**                   | AI may need about **$2T in annual revenue by 2030** to justify compute growth, with a possible **$800B revenue gap**. [TeckNexus summary of Bain](https://tecknexus.com/ai-800b-revenue-gap-compute-power-and-roi-bain-report/) |
| **Stanford consumer-surplus view** | Generative AI tools’ estimated value to U.S. consumers reached **$172B annually by early 2026**. [Stanford AI Index 2026](https://hai.stanford.edu/ai-index/2026-ai-index-report)                                               |
| **Wharton moderate macro**         | AI increases GDP/productivity **1.5% by 2035**, **3.7% by 2075**. [Wharton](https://budgetmodel.wharton.upenn.edu/p/2025-09-08-the-projected-impact-of-generative-ai-on-future-productivity-growth/)                            |
| **Acemoglu skeptic**               | GDP impact over 10 years around **1.1%-1.6%**, with very modest productivity gains. [MIT Sloan](https://mitsloan.mit.edu/ideas-made-to-matter/a-new-look-economics-ai)                                                          |

Obviously the AI buildout will need a lot of money. The money doesn't come from revenue, it has to be borrowed, so if FED rises the rates, it will hurt **A LOT**. 
It's all cool if the borrowed money that goes into CAPEX generates 20% returns per year. This is doable, if you build datacenters and make tokens. The margins on tokens are good. Smarter tokens have higher margin but there will be increased demand for opensource models as they get smarter and thus more usable. You make money either way.
But what if you're now serving 2x tokens but 50% are from opensource models and have lower margins?

I have no clue what attention mechanism is GPT and Claude are using, but they most likely have something similar and better than DeepSeeks [Compressed Sparse Attention - CSA](https://deepseek.ai/blog/deepseek-v4-compressed-attention). From my understanding, the CSA doesn't improve the models attention but only allows to **compress the KV cache by a lot** - you basically have 3 levels of compression and one of them is the "classic" and expensive full attention. If you train the model from the beginning using this approach, you don't really use performance, maybe a little, but it's worth. 

| Type of attention                             | Property                                                         |
| --------------------------------------------- | ---------------------------------------------------------------- |
| **Early layers — HCA**                        | **Cheap global summary** of the entire context window.           |
| **Middle layers — alternating HCA and CSA**   | **Balance** between long-range overview and local detail.        |
| **Final layer — full uncompressed attention** | **Maximum precision** for the output token, no information loss. |

My guess is when Nvidia **Rubin** starts being deployed in large numbers, new generation of closed source models will be deployed. Architectures and models that are now not possible or not economically viable will be deployed. It will also help the Western labs against Chinese labs.
Faster training, cheaper to make tokens, higher TPS - all a **big advantage**. 

| Metric per NVL72 rack | GB300 NVL72  | Vera Rubin NVL72 | Rubin gain       |
| --------------------- | ------------ | ---------------- | ---------------- |
| GPU memory            | 20 TB HBM3E  | 20.7 TB HBM4     | Essentially same |
| HBM bandwidth         | 576 TB/s     | 1,580 TB/s       | **2.74×**        |
| NVLink bandwidth      | 130 TB/s     | 260 TB/s         | **2×**           |
| Scale-out network/GPU | 800 Gb/s     | 1.6 Tb/s         | **2×**           |
| CPU memory            | 17 TB        | 54 TB            | **3.2×**         |
| FP4 inference         | 1,440 PFLOPS | 3,600 PFLOPS     | **2.5×**         |
| Dense FP4 training    | 1,080 PFLOPS | 2,520 PFLOPS     | **2.33×**        |
| Dense FP8 training    | ~360 PFLOPS¹ | 1,260 PFLOPS     | **3.5×**         |
| Dense BF16 training   | ~180 PFLOPS¹ | 288 PFLOPS       | **1.6×**         |
# Can they build the datacenters?
It can take a more than a year to get a permit and at least another year to build the datacenter. 
IF you are lucky enough to have GPU allocation, power, cooling, networking etc. There is a good chance you are not getting any power from the grid, so you scramble to produce power behind the meter and gas turbines are sold out until 2030.
That's why the price of GPU hours is doubling now. There will be not enough capacity added in next 1-2 years so there is a big premium to be paid when old contracts expire and are renewed for 2x price. The premium si even larger, because you have to price in the missed opportunity/revenue if you don't secure the capacity. 
Maybe in this scenario, putting the datacenters in space actually does make a sense. You get paid 2x for every GW of compute you can get online in next 6-12 months. But I think that the premium decreases as time goes on. It's really a high risk, high reward game. This might be the reason why SpaceX stock is so expensive.  
**It's always sunny in space and you don't need permits.**  





I can't shake the feeling that there is massive surveillance tool being build.


