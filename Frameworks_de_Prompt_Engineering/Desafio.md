# Um dia na Hill Valley Tech

A Hill Valley Tech é uma empresa fictícia que serve de palco para este desafio. Tem cinco sistemas em produção, cada um com seu papel bem definido. O Chronos é o API gateway e a plataforma core, ponto de entrada de todo tráfego da empresa. Por trás dele, o Ledger é um data warehouse em PostgreSQL que guarda histórico de transações e eventos, enquanto o Reactor toca o processamento assíncrono por filas de mensagens. Em paralelo a tudo isso, o Beacon mantém a observabilidade do ambiente inteiro, métricas, logs e alertas, e é por ele que o plantão enxerga o que está acontecendo. Fora do core principal, o Lift é um produto em beta que o time vem amadurecendo à parte.

O time que toca essa operação também é enxuto. Doc Brown, o CTO, responde pela direção técnica. Jennifer Parker é a PM que prioriza o que o produto entrega. Lorraine Baines lidera a SRE e responde pelo plantão, é dela a cobrança por runbooks e procedimentos documentados. George McFly é o engenheiro sênior veterano que escreveu boa parte do sistema legado, e muita coisa ainda roda exatamente como ele deixou anos atrás. Goldie Wilson, a CEO, observa tudo pelo prisma de custo e crescimento. E Strickland, head de segurança e compliance, é quem bate carimbo nos padrões internos que todo código novo precisa seguir.

Nos próximos cenários você vai pegar algumas dessas demandas que chegam à mesa do time. Em cada questão, a entrega é um prompt de IA aplicando o framework indicado no enunciado, executado em um modelo, com o output registrado e a justificativa mostrando como os componentes do framework apareceram no prompt.

A Questão 08 foge desse padrão: a escolha do framework fica por sua conta entre os cinco do capítulo, com comparação explícita contra duas alternativas.


## Como a entrega deve ser feita
A entrega é um repositório público no GitHub contendo os prompts, outputs e justificativas das 8 questões. O link do repositório deve ser enviado ao final.

A organização interna do repositório é livre e intencional. No Capítulo 4 deste módulo será abordado Criação e Versionamento de Prompts, e a estrutura escolhida aqui será comparada com as práticas apresentadas lá. Guardar a decisão, ela é parte do aprendizado.

## Cada questão exige 4 campos obrigatórios:

Prompt: o texto exato usado.
Modelo: qual modelo foi executado (ex.: GPT-4o, Claude Sonnet 4, Gemini 2.5 Pro, Llama 3 via Ollama) e, em 1 linha, por que esse modelo foi escolhido para a tarefa.
Output: a resposta real do modelo, na íntegra ou em trecho relevante.
Justificativa: em 2 a 4 linhas, mostrar como os componentes do framework indicado no enunciado aparecem no prompt. Na Q08 a justificativa precisa comparar o framework escolhido com 2 alternativas.
Algumas orientações práticas:

Usar ao menos 2 providers distintos ao longo do desafio (OpenAI, Anthropic, Google, Meta ou local via Ollama).
Registrar outputs ruins também. Se um resultado não ficou bom, comentar na justificativa o que faria diferente.
Os dados dos cenários são fictícios, sem necessidade de sanitização.

Sete das oito questões já trazem o framework definido no enunciado, ali o valor está em aplicar bem os componentes no prompt e explicar como cada um aparece.

Na Q08 a escolha é sua entre os cinco do capítulo e a justificativa compara com 2 alternativas. Registrar o raciocínio, inclusive o que não funcionou, faz parte do valor da entrega.

## Bônus - Marketing pessoal (opcional)

Seção opcional, sem peso na avaliação. O que você aprendeu até este capítulo do curso é matéria-prima para posts de LinkedIn, threads no X ou artigos técnicos. Abaixo vão 3 temas baseados em aulas já cobertas (Fundamentos de IA, LLMs, Ferramentas e Plataformas, GenAI no cotidiano técnico, Fundamentos de Prompt Engineering), pensados para gerar autoridade na área.

### Tema 1 - Como escolher o modelo de IA certo: custo, latência, qualidade e privacidade
Aproveita o que foi visto sobre providers (OpenAI, Anthropic, Google, Meta via Ollama), formas de consumo (chatbot, agente, API) e os critérios para decidir entre eles. É a pergunta que todo time técnico está fazendo agora, e poucos profissionais sabem responder além de "uso ChatGPT".

Formato sugerido: carrossel de LinkedIn (5 a 7 slides). Hook (ex.: "Qual modelo de IA seu time deveria estar usando? GPT, Claude, Gemini ou Llama? A resposta depende de 4 critérios."), um slide por critério com exemplo prático, slide final com uma matriz de decisão simples.

### Tema 2 - Os 5 frameworks de prompt engineering aplicados a Cloud, DevOps e SRE
Aproveita as aulas dos frameworks (R-T-F, T-A-G, B-A-B, C-A-R-E, R-I-S-E) e do comparativo final. Um framework por seção, com um exemplo concreto da área (genérico, não os do desafio) e a indicação de quando usar cada um.

Formato sugerido: artigo longo no LinkedIn ou thread no X com 6 a 8 posts, um por framework e o último consolidando a árvore de decisão.

### Tema 3 - Tokens e janela de contexto: o que todo profissional técnico deveria entender antes de jogar prompt na IA
Aproveita as aulas de tokens, tokenização e janela de contexto. Tema técnico que pouca gente explora bem mas que impacta diretamente custo (pay-per-token) e qualidade da resposta (lost-in-the-middle, estouro da janela). Forte para diferenciar de quem só usa IA pela interface.

Formato sugerido: artigo técnico no Medium, Dev.to ou blog pessoal. Estrutura: o que é um token, por que importa para custo, por que importa para qualidade, exemplos práticos com prompts curtos vs prompts inflados.