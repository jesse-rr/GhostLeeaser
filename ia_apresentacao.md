# Apresentação do Projeto: Game Development Viability Advisor

Esta apresentação foi estruturada com base nos entregáveis da disciplina (projeto_ia_icd_ads.docx.md) e nos detalhes do artigo científico do projeto (Template-jrr-1.pdf).

---

## 1. Motivação e Problema

* **Contexto do Mercado:**
  * O mercado de jogos para PC na plataforma Steam é extremamente competitivo, recebendo dezenas de novos lançamentos diariamente.
  * A grande maioria dos títulos independentes (indies) falha em gerar retorno financeiro suficiente para cobrir seus custos básicos de desenvolvimento.
* **O Problema:**
  * Grandes publicadoras de jogos possuem departamentos dedicados de pesquisa de mercado e inteligência de negócios.
  * Desenvolvedores independentes e pequenos estúdios não possuem esses recursos, dependendo quase que exclusivamente de intuição criativa e "achismos" ao tomar decisões de design e negócios.
* **A Solução Proposta:**
  * Um sistema gratuito de apoio à decisão (web app) que une Aprendizado de Máquina (modelos preditivos) e análise heurística de mercado para guiar desenvolvedores na concepção comercial de seus jogos.
* **Público-Alvo:**
  * Desenvolvedores de jogos independentes, estudantes de game design e pequenos estúdios de desenvolvimento.

---

## 2. Dados e Abordagem de IA Adotada

* **Fonte de Dados:**
  * Steam Games Dataset (Kaggle), contendo registros históricos de 122.277 jogos da plataforma Steam.
* **Definição de Sucesso (Variável Alvo - is_successful):**
  * Critério binário: um jogo é classificado como "bem-sucedido" se possuir taxa de aprovação dos usuários (approval rate) igual ou superior a 70% E uma nota média efetiva (Metacritic ou score de usuários) igual ou superior a 7 (em escala de 10).
* **Variáveis de Entrada (Features):**
  * Preço, gêneros, tags descritivas, tempo médio de jogo, estimativa de wishlists, avaliações da crítica e dos usuários, pico de jogadores simultâneos (CCU), número de conquistas, quantidade de DLCs, suporte a sistemas operacionais (Windows, Mac, Linux) e data de lançamento (mês, ano, estação).
* **Pré-processamento de Dados:**
  * Remoção de registros duplicados ou nulos.
  * Imputação de valores ausentes por meio da mediana das colunas.
  * Codificação de listas de gêneros e tags (gerando 94 colunas binárias).
  * Tratamento de desbalanceamento de classes através da técnica SMOTE.
  * Normalização dos dados numéricos usando MinMaxScaler.
  * Seleção dos melhores atributos via SelectKBest (f_classif).
* **Abordagem de IA:**
  * Avaliação de três classificadores supervisionados: Random Forest, XGBoost e LightGBM.
  * Criação de um Super Ensemble por votação suave (soft voting), combinando as probabilidades preditas pelos três modelos para gerar a predição final de sucesso.
  * Desenvolvimento em paralelo de um motor heurístico baseado em 14 fatores de mercado, cujos pesos são calibrados automaticamente por correlações de Pearson com a taxa de sucesso histórica.

---

## 3. Demonstração da Solução (Como Funciona a Aplicação)

* **Arquitetura da Aplicação:**
  * Interface Web Estática (HTML5, Vanilla CSS, Vanilla Javascript) para interação com o usuário através de formulários dinâmicos.
  * Backend API REST (Python Flask) que processa as requisições em tempo real.
* **Fluxo de Uso:**
  * O desenvolvedor insere os dados de seu projeto conceitual no formulário (gêneros, tags, faixa de preço, estimativa de tempo de jogo, estimativa de custo de desenvolvimento, etc.).
  * A API Flask recebe o payload JSON e executa simultaneamente:
    1. A inferência pelo Super Ensemble de ML para estimar a probabilidade de sucesso histórica.
    2. O cálculo da nota de viabilidade (0 a 100) através do motor heurístico de 14 fatores.
  * O frontend recebe a resposta consolidada em JSON e renderiza graficamente um painel de resultados contendo:
    * Score geral de viabilidade comercial e veredicto (Alta Viabilidade, Viabilidade Moderada, Risco Elevado, Baixa Viabilidade).
    * Probabilidade de sucesso estimada pela IA.
    * Gráficos detalhando a pontuação e peso individual de cada um dos 14 fatores de viabilidade (como adequação de preço, potencial de gênero, apelo de tags, saturação de mercado e ponto de equilíbrio de custos).
    * Alertas éticos dinâmicos que sugerem caminhos para o desenvolvedor caso haja contradições entre a IA e o motor heurístico.

---

## 4. Resultados Obtidos e Impactos

* **Desempenho dos Modelos no Conjunto de Teste (48.536 Amostras):**
  * Baseline (Classe Majoritária): Acurácia de 50,3% e F1-score de 0,0%.
  * Random Forest: Acurácia de 91,1% e F1-score de 91,0%.
  * XGBoost: Acurácia de 95,25% e F1-score de 95,0%.
  * LightGBM: Acurácia de 95,37% e F1-score de 95,0%.
  * **Super Ensemble:** Acurácia de 95,32% e F1-score de 95,0%.
* **Experimentos Realizados:**
  * Experimento 1 (Hiperparâmetros): Ajuste de profundidade máxima de 14 para Random Forest para evitar overfitting; definição de 60 estimadores com learning rate de 0,1 para XGBoost.
  * Experimento 2 (Tags): O modelo treinado apenas com gêneros foi comparado com o modelo que inclui as 71 tags mais frequentes. A inclusão das tags aumentou a acurácia em 4,5% e reduziu significativamente os falsos positivos.
  * Experimento 3 (Normalização): O uso de normalização via MinMaxScaler aumentou a estabilidade e desempenho da Random Forest em 1,8%.
* **Impacto Comercial e de Desenvolvimento:**
  * O sistema permite a criadores independentes validar suas ideias e otimizar parâmetros críticos (como a precificação e a escolha de tags) antes de iniciarem a produção de fato, reduzindo riscos financeiros.

---

## 5. Limitações e Próximos Passos

* **Limitações Identificadas:**
  * **Viés de Sobrevivência:** A base de dados registra apenas jogos que foram efetivamente publicados no catálogo da Steam. Projetos cancelados ou abandonados não estão presentes, o que pode inflar as taxas gerais de sucesso.
  * **Foco Regional:** Os dados refletem primordialmente o comportamento de consumo e o catálogo de mercados ocidentais e asiáticos de grande escala, com generalização reduzida para ecossistemas de mercados emergentes.
  * **Estimativa de Wishlists:** Por ser uma métrica privada, as wishlists foram estimadas indiretamente a partir das recomendações de usuários, gerando possíveis imprecisões numéricas.
  * **Processamento Offline:** Não há integração em tempo real com as mudanças diárias do catálogo oficial da Steam.
* **Trabalhos Futuros (Próximos Passos):**
  * Integração direta com a API Web da Steam para atualização automatizada e incremental da base de dados.
  * Expansão da modelagem para abranger ecossistemas de consoles (PlayStation, Xbox, Nintendo Switch) e dispositivos móveis (iOS e Android).
  * Implementação de Processamento de Linguagem Natural (NLP) para análise de sentimento nos comentários escritos de usuários, enriquecendo as variáveis de qualidade e aprovação.

---

## Glossário de Termos

* **Aprendizado de Máquina (Machine Learning / ML):**
  * Subcampo da Inteligência Artificial que desenvolve algoritmos capazes de aprender padrões e fazer previsões ou decisões a partir de dados históricos, sem serem programados explicitamente para aquela tarefa específica.
* **Ensemble (Método Ensemble / Super Ensemble):**
  * Técnica em ciência de dados que combina a predição de múltiplos modelos de aprendizado de máquina individuais (como Random Forest, XGBoost e LightGBM) para gerar uma predição única e mais robusta, reduzindo a variância e o viés que cada modelo teria individualmente. No projeto, a votação suave (soft voting) tira a média aritmética das probabilidades atribuídas por cada modelo.
* **Random Forest (Floresta Aleatória):**
  * Algoritmo de aprendizado de máquina supervisionado baseado em árvores de decisão. Ele cria uma "floresta" com diversas árvores treinadas em subconjuntos diferentes dos dados e faz a votação da maioria para classificar uma amostra.
* **XGBoost e LightGBM (Gradient Boosting):**
  * Algoritmos de aprendizado de máquina baseados em árvores de decisão que utilizam a técnica de Boosting (impulsionamento). Eles constroem árvores de forma sequencial, onde cada nova árvore tenta corrigir os erros residuais cometidos pelas árvores anteriores. O LightGBM destaca-se pela otimização de uso de memória e velocidade de processamento de grandes bases.
* **SMOTE (Synthetic Minority Over-sampling Technique):**
  * Técnica usada para resolver problemas de bases de dados desbalanceadas (onde uma classe tem muitas amostras e a outra possui pouquíssimas). O SMOTE cria instâncias sintéticas (artificiais mas matematicamente coerentes) da classe minoritária com base no perfil das vizinhanças mais próximas.
* **Baseline (Linha de Base):**
  * Um ponto de partida simples ou um modelo heurístico trivial usado para avaliar a qualidade e a relevância de modelos mais complexos. No projeto, a linha de base de classe majoritária representava 50,3% de acurácia (um chute aleatório ou predição estática da classe mais comum).
* **Acurácia (Accuracy):**
  * A proporção de predições corretas (verdadeiros positivos e verdadeiros negativos) em relação ao total de amostras avaliadas. Uma acurácia de 95% significa que o modelo acertou a classificação em 95 de cada 100 jogos testados.
* **F1-Score:**
  * Uma métrica de desempenho que calcula a média harmônica entre Precisão (quantos dos jogos marcados como de sucesso eram realmente de sucesso) e Sensibilidade/Recall (quantos dos jogos que realmente tiveram sucesso o modelo foi capaz de identificar). É especialmente útil para validar modelos em cenários onde há desbalanceamento de classes.
* **SelectKBest:**
  * Um método de seleção de atributos que avalia estatisticamente a relação entre cada feature de entrada e a variável alvo, selecionando apenas as "K" características com maior poder discriminativo. Ajuda a reduzir a dimensionalidade dos dados e o ruído do modelo.
* **MinMaxScaler (Normalização):**
  * Um transformador de dados que dimensiona todos os atributos numéricos de forma a estarem dentro de um intervalo específico (geralmente entre 0 e 1), dividindo a diferença pelo valor máximo menos o mínimo. Evita que variáveis com escalas numéricas muito grandes dominem algoritmos sensíveis a distâncias.
* **Heurística (Motor Heurístico):**
  * Um método ou regra prática concebida para encontrar soluções rápidas e eficientes para problemas complexos. No sistema, é um conjunto estruturado de 14 equações empíricas baseadas na proximidade com percentis estatísticos do mercado.
* **Viés de Sobrevivência (Survivorship Bias):**
  * Um erro lógico onde focamos apenas nos membros de um grupo que "sobreviveram" a um determinado processo (no caso, jogos que conseguiram ser efetivamente desenvolvidos e publicados na Steam), ignorando aqueles que falharam antes de atingir essa etapa (como projetos cancelados), o que pode levar a conclusões excessivamente otimistas.
* **CCU (Concurrent Users / Jogadores Simultâneos):**
  * Métrica que mede o número de usuários que estão jogando ativamente o jogo ao mesmo tempo. É um indicador direto de engajamento e popularidade de uma comunidade.
* **API REST:**
  * Estilo de arquitetura de software que permite a comunicação padronizada entre diferentes sistemas web por meio de requisições HTTP (GET, POST, etc.) e intercâmbio de dados estruturados (usualmente em formato JSON).
