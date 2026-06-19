---
title: Dává to smysl?
date: 2026-06-19
slug: does-it-make-sense
---

Už uběhla spousta času a já moc nepsal ani neprogramoval. Jednoduše mi došly nápady a motivace pouštět se do čehokoli.  
O LLMs se zajímám už nějakou dobu a teď se snažím pochopit trh, který kolem nich vznikl. Zkusil jsem napsat pár svých myšlenek. Možná se mýlím; pokud ano, dejte mi vědět.

# Můj naivní pohled na trh
Nejsem expert, jsem začátečník a měli byste si udělat vlastní průzkum. Tak pojďme na to.  

**Co jsou vlastně ty trhy zač?**  
Podle mě je nákup akcií něco jako půjčování peněz. Firma potřebuje pár miliard? Smradlavá banka jí nechce miliardy půjčit? 
Obraťte se na lidi (trh)! Mají hromadu peněz a není to tak, že by věděli, jak je lépe využít. Za každý dolar, který mi půjčí, jim vrátím něco navíc. Věřte mi, vydělám tolik peněz, že výnosy budou vysoké!  

Vím, že je to pravděpodobně složitější, ale takhle to vidím. Efektivní alokace kapitálu nebo tak něco.  

Kolem LLM je teď velké nadšení. Nejpíš ho máš plné zuby. Obuvnická firma oznámí přechod na AI? Akcie jdou nahoru o 580 % (fakt: [Akcie Allbirds po přechodu od bot k AI vzrostly o 580 %
](https://www.bbc.com/news/articles/c98mrepzgj7o))

<img src="/images/shoe-company.png" alt="Graf akcií obuvnické firmy" style="max-width: 500px; width: 100%;">

Akcie firem z oblasti AI, polovodičů a čipů se obchodují za velmi vysoké násobky P/E a trh do jejich cen započítává tržby očekávané za 2–3 roky. Možná se na to díváte a přijde vám to trochu divné.

**Dává něco z toho smysl?**  
Jste „AGI-pilled“? Pokud je odpověď ANO, pravděpodobně jste zatím vydělali spoustu peněz a všechno vám dává dokonalý smysl. Také nejspíš doufáte, že číslo půjde dál nahoru. Pokud jste odpověděli NE, jste možná trochu zahořklí, že vám ušly zisky, a možná tomu říkáte bublina.

Přece si nemohou ukrojit biliony dolarů tržeb, jak tvrdí, ne?

<img src="/images/hyperscaler-ai-capex.png" alt="Kapitálové výdaje hyperscalerů na AI" style="max-width: 700px; width: 100%;">

| Pohled                                | Aktuální tvrzení pro rok 2026                                                                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Odhad výdajů Gartneru**             | Celosvětové výdaje na AI mají v roce 2026 dosáhnout **2,52 bilionu USD**, meziročně o 44 % více. [Gartner](https://www.gartner.com/en/newsroom/press-releases/2026-1-15-gartner-says-worldwide-ai-spending-will-total-2-point-5-trillion-dollars-in-2026) |
| **Infrastruktura podle Goldman Sachs** | Kapitálové výdaje hyperscalerů na AI mají v roce 2026 dosáhnout přibližně **757 mld. USD** a v roce 2027 vzrůst na **920 mld. USD**. [Shrnutí Business Insider](https://www.businessinsider.com/stocks-to-buy-ai-capex-beneficiaries-tech-investing-goldman-sachs-2026-6) |
| **Výpočty skeptiků**                  | AI může do roku 2030 potřebovat přibližně **2 biliony USD ročních tržeb**, aby ospravedlnila růst výpočetní kapacity; možná mezera v tržbách činí **800 mld. USD**. [Shrnutí Bain od TeckNexus](https://tecknexus.com/ai-800b-revenue-gap-compute-power-and-roi-bain-report/) |
| **Stanfordský pohled na přínos spotřebitelům** | Odhadovaná hodnota nástrojů generativní AI pro spotřebitele v USA dosáhla začátkem roku 2026 **172 mld. USD ročně**. [Stanford AI Index 2026](https://hai.stanford.edu/ai-index/2026-ai-index-report) |
| **Umírněný makroekonomický pohled Whartonu** | AI zvýší HDP a produktivitu o **1,5 % do roku 2035** a o **3,7 % do roku 2075**. [Wharton](https://budgetmodel.wharton.upenn.edu/p/2025-09-08-the-projected-impact-of-generative-ai-on-future-productivity-growth/) |
| **Acemoglu – skeptický pohled**       | Dopad na HDP během deseti let se odhaduje přibližně na **1,1–1,6 %**, s velmi mírným růstem produktivity. [MIT Sloan](https://mitsloan.mit.edu/ideas-made-to-matter/a-new-look-economics-ai) |

Výstavba AI infrastruktury bude samozřejmě stát spoustu peněz. Tyto peníze nepocházejí z tržeb, firmy si je musí půjčit, takže pokud Fed zvýší sazby, bude to **HODNĚ BOLET**.
Všechno je v pohodě, pokud peníze vypůjčené a investované do CAPEX generují 20% roční výnos. To je proveditelné, když stavíte datová centra a generujete tokeny. Marže na tokenech jsou dobré. Chytřejší tokeny mají vyšší marže, zároveň ale poroste poptávka po open-source modelech, které jsou čím dál chytřejší, a tím pádem i použitelnější. Vyděláte tak jako tak.
Co když teď ale vyrábíte dvojnásobek tokenů, jenže 50 % z nich pochází z open-source modelů a má nižší marže?

Netuším, jaké mechanismy pozornosti GPT a Claude používají, nejspíš ale mají něco podobného a lepšího než [Compressed Sparse Attention – CSA](https://deepseek.ai/blog/deepseek-v4-compressed-attention) od DeepSeeku. Pokud to správně chápu, CSA nezlepšuje pozornost modelu, pouze umožňuje **výrazně zkomprimovat KV cache** – v zásadě máte tři úrovně komprese a jenom jednou z nich je „klasická“ a drahá plná pozornost. Když model od začátku trénujete tímto způsobem, o výkon prakticky nepřijdete, možná jen trochu, ale stojí to za to.

| Typ mechanismu pozornosti                           | Vlastnost                                                               |
| --------------------------------------------------- | ----------------------------------------------------------------------- |
| **Rané vrstvy – HCA**                               | **Levné globální shrnutí** celého kontextového okna.                    |
| **Střední vrstvy – střídání HCA a CSA**             | **Rovnováha** mezi přehledem na dlouhou vzdálenost a lokálními detaily. |
| **Poslední vrstva – plná nekomprimovaná pozornost** | **Maximální přesnost** výstupního tokenu bez ztráty informací.          |

Můj odhad je, že až se Nvidia **Rubin** začne nasazovat ve velkém, přijde nová generace uzavřených modelů. Vzniknou architektury a modely, které dnes nejsou možné nebo jsou ekonomicky neziskové. Pomůže to také západním laboratořím v konkurenci s těmi čínskými.
Rychlejší trénování, levnější generování tokenů, vyšší TPS – to všechno je **velká výhoda**.

| Metrika na rack       | GB300 NVL72  | Vera Rubin NVL72 | Přínos Rubinu       |
| --------------------- | ------------ | ---------------- | ------------------- |
| Paměť GPU             | 20 TB HBM3E  | 20,7 TB HBM4     | V zásadě stejná     |
| Propustnost HBM       | 576 TB/s     | 1 580 TB/s       | **2,74×**           |
| Propustnost NVLink    | 130 TB/s     | 260 TB/s         | **2×**              |
| Scale-out síť / GPU   | 800 Gb/s     | 1,6 Tb/s         | **2×**              |
| Paměť CPU             | 17 TB        | 54 TB            | **3,2×**            |
| Inference FP4         | 1 440 PFLOPS | 3 600 PFLOPS     | **2,5×**            |
| Trénování Dense FP4   | 1 080 PFLOPS | 2 520 PFLOPS     | **2,33×**           |
| Trénování Dense FP8   | ~360 PFLOPS¹ | 1 260 PFLOPS     | **3,5×**            |
| Trénování Dense BF16  | ~180 PFLOPS¹ | 288 PFLOPS       | **1,6×**            |

# Dokážou datová centra postavit?
Získat povolení může trvat déle než rok a nejméně další rok zabere výstavba datového centra.
To za předpokladu, že se vám podaří zajistit GPUs, elektřina, chlazena, síťovéhé připojení atd. Je docela možné, že ze sítě nedostanete žádnou elektřinu, takže se ji snažíte vyrábět přímo v areálu – a plynové turbíny jsou vyprodané až do roku 2030.
Proto se teď cena GPU hodiny zdvojnásobuje. Během příštího roku až dvou nepřibude dostatek kapacity, takže až staré smlouvy vyprší a budou se obnovovat za dvojnásobnou cenu, bude nutné zaplatit výraznou prémii. Ta je ještě vyšší, protože do ceny musíte započítat ušlou příležitost a tržby, pokud si kapacitu nezajistíte.
Možná v tomto scénáři umístění datových center do vesmíru skutečně dává smysl. Za každý GW výpočetního výkonu, který dokážete uvést do provozu během příštích 6–12 měsíců, dostanete zaplaceno dvojnásobně. Myslím si ale, že tato prémie bude postupem času klesat. Je to hra s vysokým rizikem a vysokou odměnou. To je možná důvod, proč hodnota SpaceX je tak vysoká.  
**Ve vesmíru je vždy slunečno a nepotřebujete povolení.**

Nemohu se zbavit pocitu, že se tu buduje obrovský nástroj pro sledování.
