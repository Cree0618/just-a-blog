---
date: 2026-06-21
title: Dopad dobrých open-source modelů
---
GLM 5.2 vyšel před pár dny a už si získává hodně pozornosti. Když přemýšlím o nedostatku pamětí, napadlo mě: **Zvýší to jen poptávku po paměti a výpočetním výkonu?** Teď můžeš provozovat velmi chytré otevřené modely levněji, protože jsou open-source, a tím pádem je na tokenech menší marže. To znamená, že si můžeš dovolit víc tokenů. Víc tokenů znamená, že se při jejich produkci použilo víc paměti. Pokud škálování výpočetního výkonu vždy funguje, škálování paměti jde ruku v ruce s ním.

DeepSeek V4 ukázal nové modely komprimované attention, které zmenšily KV cache více než 10x. GLM 5.2 ale tak agresivně komprimovanou attention neimplementoval a dosáhl lepších výsledků. Pravděpodobně to bylo způsobeno lepším tréninkem a dalšími věcmi?  

**Je tedy komprese KV cache špatná?**  
Komprese není bezeztrátová. Pokaždé, když data komprimuješ, ztrácíš signál. U dlouhodobých a komplexních úloh začínají být nuance důležité. Dá se argumentovat tím, že ztratit tuhle nuanci se NEVYPLATÍ. Chyby se rychle hromadí. Prostě použij víc test-time compute. Výpočetní výkon pro tebe není tak drahý, abys kvůli němu obětoval inteligenci.  

Ale můžeš zastávat i opačný názor. Možná chceš stlačit cenu tokenů níž a trénovat/upravovat architekturu modelu kolem komprimované KV cache. Možná najdeš způsob, jak signál zesílit? Ukázalo se, že můžeš použít nižší přesnost, trénovat model s touto nižší přesností (NVFP4 nebo NVFP8) a obětovat jen málo výkonu. Nvidia už navrhuje svoje GPU kolem architektur s nižší přesností, takže čekám, že vědí proč. Úzká spolupráce se SOTA laby ti dá dobrou představu o jejich potřebách. Nepřekvapilo by mě, kdyby GPT 5.5 běžel v NVFP8 nebo NVFP4 podle úrovně reasoning.

Celkově si myslím, že tlak půjde směrem k optimalizaci nákladů na tokeny. To bude stejně důležité jako inteligence. Objem bude čím dál relevantnější než SOTA kvalita.
