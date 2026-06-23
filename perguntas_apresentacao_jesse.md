# Perguntas de Treinamento para a Apresentação - Contribuições de Jesse

Este documento contém perguntas simuladas e respostas detalhadas focadas exclusivamente nas contribuições de **Jesse** no projeto **Deu Zebra**. Utilize-o para estudar e se preparar para as perguntas da banca/professor.

---

## 🧭 Bloco 1: Localização GPS Nativa (`expo-location`)

### Pergunta 1
> **"Jesse, explique como você implementou a captura de localização GPS nativa no aplicativo. Por que você estruturou a lógica dividida entre um arquivo de serviço (`localizacaoService.ts`) e um hook customizado (`useLocalizacao.ts`), e como o aplicativo lida com a recusa de permissão do usuário?"**

#### Resposta Sugerida
*   **Separação de Responsabilidades:** A lógica foi dividida para manter o código limpo, testável e seguindo os princípios de SOLID.
    *   O [localizacaoService.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/services/localizacaoService.ts) é um serviço puro que interage diretamente com o SDK `expo-location`. Ele se preocupa apenas com a obtenção dos dados do hardware e o tratamento imediato do status de permissão.
    *   O [useLocalizacao.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/hooks/useLocalizacao.ts) é um hook React que gerencia o estado da interface (se está carregando, se há erro na tela, ou se as coordenadas foram adquiridas), encapsulando o ciclo de vida do React.
*   **Fluxo de Permissão:** Quando chamamos `obterCoordenadas()`, o SDK solicita a permissão em primeiro plano (`requestForegroundPermissionsAsync`).
    *   Se o usuário **conceder**, chamamos `getCurrentPositionAsync()` para capturar a latitude e a longitude reais e retorná-las na estrutura `Coordenadas`.
    *   Se o usuário **negar**, o serviço lança um erro com a string constante `PERMISSAO_LOCALIZACAO_NEGADA`.
*   **Tratamento de Recusa na UI:** O hook captura esse erro no bloco `catch` e atualiza o estado de erro (`erro: mensagem`). Isso permite que a tela exiba amigavelmente ao usuário que a geolocalização falhou porque a permissão foi negada, permitindo que a aplicação continue rodando com fallbacks de clima e endereço sem quebrar a tela do app.

---

## 🌤️ Bloco 2: Clima e Contingência de Geocodificação Reversa (Open-Meteo & Nominatim)

### Pergunta 2
> **"Jesse, sobre a integração climática com a API Open-Meteo, como você mapeou os dados técnicos de clima para textos amigáveis? Além disso, por que você implementou uma chamada manual à API do Nominatim (OpenStreetMap) no hook `useClima.ts` se o Expo já fornece o método `Location.reverseGeocodeAsync`?"**

#### Resposta Sugerida
*   **Mapeamento de Clima:** A API Open-Meteo fornece um código numérico chamado `weather_code` baseada nos padrões meteorológicos da WMO (World Meteorological Organization). No arquivo [climaService.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/services/climaService.ts), eu criei um dicionário estático chamado `descricoesPorCodigo` que mapeia cada código relevante para uma string amigável em português (exemplo: `0` para "Céu limpo", `61` para "Chuva fraca", `95` para "Trovoada", etc.).
*   **Estratégia de Contingência (Fallback de Geocode):**
    *   O método nativo `Location.reverseGeocodeAsync` do Expo consome serviços nativos do sistema operacional (Google Play Services no Android ou Apple Location Services no iOS). Em emuladores, aparelhos sem Google Play Services configurado ou quando há limite de requisições, esse método nativo costuma falhar ou retornar uma lista vazia.
    *   Para garantir que o aplicativo continue descobrindo o nome da cidade do usuário mesmo que o método nativo falhe, eu implementei um fallback utilizando uma requisição HTTP direta à API pública do **Nominatim (OpenStreetMap)**.
    *   Se a geocodificação do Expo falhar ou retornar `Desconhecido`, o código executa um `fetch` para `nominatim.openstreetmap.org/reverse` informando a latitude e a longitude obtidas e extrai a cidade do JSON retornado (`json.address?.city`). Isso garante resiliência e que a IA do Gemini receba a cidade correta para gerar a desculpa.

---

## 🤖 Bloco 3: Integração com Gemini API e o Sistema de Fallback Local

### Pergunta 3
> **"Jesse, explique como foi implementada a comunicação com a IA do Gemini e por que você optou por chamadas REST brutas (via `fetch`) em vez do SDK oficial da Google. Detalhe também como funciona o mecanismo de resiliência local do `desculpaFallbackService` se a chave de API estiver ausente ou a rede falhar."**

#### Resposta Sugerida
*   **Chamadas REST via `fetch`:** Optamos por usar chamadas REST via `fetch` em vez do SDK `@google/generative-ai` por dois motivos principais no ambiente de desenvolvimento mobile:
    1.  **Redução de dependências e tamanho do bundle:** Evita adicionar pacotes pesados de Node.js que às vezes exigem polyfills complexos para rodar no ambiente hermético do React Native/Expo.
    2.  **Transparência e Controle:** Uma requisição HTTP `fetch` direta para o endpoint `/v1beta/models/gemini-2.5-flash:generateContent` nos dá controle total dos cabeçalhos, tempo limite e tratamento de códigos de erro HTTP.
*   **Mecanismo de Resiliência (Fallback):**
    *   No arquivo [geminiService.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/services/geminiService.ts), a função `gerarDesculpa` primeiro valida se existe uma chave de API configurada através do `obterGeminiApiKey()`.
    *   Caso a chave **não exista**, ou caso a chamada `fetch` **falhe** (capturada no bloco `catch` devido a falta de internet, erro de servidor 500, etc.), o sistema de forma transparente aciona o [desculpaFallbackService.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/services/desculpaFallbackService.ts).
    *   O `desculpaFallbackService` carrega uma base local de desculpas estáticas agrupadas por tom a partir de [desculpasFallback.json](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/constants/desculpasFallback.json).
*   **Prevenção de Repetição:** O serviço de fallback recebe o histórico recente de desculpas e usa um `Set` para filtrar e remover desculpas que o usuário já enviou recentemente. Ele seleciona aleatoriamente uma desculpa do "pool" restante. Isso garante que, mesmo sem internet e sem chave de API, o usuário receba desculpas úteis e não repetitivas.

---

## 🎨 Bloco 4: Tela de Geração e Animações Fluidas (`Animated` API)

### Pergunta 4
> **"Jesse, na tela `GerarDesculpaScreen.tsx`, você dispara a carga do contexto climático logo ao carregar a tela e implementou uma animação de rotação contínua para o feedback de carregamento. Como foi programado esse ciclo de vida assíncrono e por que você utilizou o parâmetro `useNativeDriver: true` na animação?"**

#### Resposta Sugerida
*   **Ciclo de Vida Assíncrono:** No `useEffect` disparado na montagem da tela (`[]`), chamamos a função `carregarContexto()`. Ela primeiro aguarda a localização (`await obterLocalizacao()`). Se obtiver coordenadas com sucesso, chama a API de clima de forma encadeada (`await buscarClima(coords)`). O usuário vê indicadores discretos de carregamento enquanto isso ocorre, mas o fluxo da interface não trava, permitindo que ele digite uma situação ou escolha um tom enquanto o clima carrega em background.
*   **Animação com `Animated`:** Para dar um visual premium e interativo, quando a geração com Gemini é iniciada (`carregandoGeracao` se torna `true`), disparados um loop infinito de animação rotacionando o ícone/varinha mágica da tela.
    *   Usamos `Animated.loop` combinado com um `Animated.timing` que varia o valor de `0` a `1` em 1000 milissegundos.
    *   Quando a geração termina, no cleanup do `useEffect` ou quando `carregandoGeracao` muda para `false`, chamamos `animacao.stop()` e reiniciamos o valor da animação para `0`.
*   **Importância do `useNativeDriver: true`:**
    *   Ao definir `useNativeDriver: true`, estamos instruindo o React Native a enviar toda a definição da animação para o lado nativo da aplicação (Java/Kotlin no Android ou Objective-C/Swift no iOS) **antes** de executá-la.
    *   Isso significa que a animação rodará diretamente na thread de renderização nativa (UI Thread), em vez de ter que passar pela ponte JavaScript (JS bridge) a cada frame (60 vezes por segundo).
    *   Como resultado, mesmo se a thread do JavaScript estiver ocupada processando dados ou montando a lista de desculpas recebidas, a animação de rotação continuará perfeitamente fluida e sem engasgos (sem lag de FPS).

---

## 📝 Bloco 5: Validação de Formulários e API de CEP (ViaCEP)

### Pergunta 5
> **"Jesse, explique as regras que você criou em `validacao.ts` e como integrou o autocompletar de CEP na tela do chefe. O que acontece se o usuário digitar um CEP com formato inválido ou inexistente?"**

#### Resposta Sugerida
*   **Validações Puras:** Em [validacao.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/utils/validacao.ts), mantemos funções utilitárias desacopladas da UI, como `textoPreenchido` e `tamanhoMinimo`, para garantir reutilização. Elas validam strings limpando espaços em branco (`trim()`). Na tela de Perfil do Usuário e do Chefe, elas impedem o envio de dados vazios ou curtos demais, exibindo mensagens de erro personalizadas no formulário.
*   **Integração do ViaCEP (PerfilChefeScreen):**
    *   No campo de CEP da tela de Perfil do Chefe, o aplicativo monitora o input do usuário. Primeiro, removemos caracteres não numéricos (`replace(/\D/g, "")`).
    *   Se a string resultante possuir **exatamente 8 dígitos**, o aplicativo inicia uma busca em segundo plano para o endpoint `https://viacep.com.br/ws/${cep}/json/`.
    *   Ao obter a resposta da API, se ela retornar sucesso, extraímos a localidade (cidade) e a UF (estado) e preenchemos automaticamente o campo de endereço da empresa no formato `Cidade - UF`.
*   **Tratamento de CEP Inválido/Inexistente:**
    *   Se o CEP for inexistente, o retorno da API ViaCEP traz a propriedade `{ erro: true }`. O código trata isso definindo o estado de erro `setErroCep("CEP nao encontrado.")` e limpa os campos de preenchimento automático.
    *   Se houver um problema de rede ou formato de CEP inválido antes do envio (ex. menos de 8 caracteres), a requisição nem sequer é enviada, ou o bloco `catch` intercepta o erro de rede e exibe a mensagem `"Nao foi possivel buscar o CEP. Verifique a conexao."` na tela, sem travar o aplicativo.

---

## 🧠 Bloco 6: Estruturação do Prompt e Engenharia de IA

### Pergunta 6
> **"Jesse, você foi o responsável por estruturar o prompt dinâmico inicial em `montarPrompt.ts`. Como você desenhou as regras desse prompt para que a IA se comporte exatamente como um trabalhador escrevendo no WhatsApp e evite respostas repetitivas?"**

#### Resposta Sugerida
*   **Estruturação Dinâmica:** A função `montarPromptDesculpa` recebe um objeto com o perfil do usuário (nome, profissão), perfil do chefe (nome, empresa, localização da empresa), dados climáticos do momento, tom desejado, histórico e a situação. Ela mescla essas informações em blocos estruturados de texto.
*   **Regras de Estilo WhatsApp:** No prompt, inseri restrições imperativas muito claras para moldar o estilo da resposta da IA:
    *   *Directness:* "Use no máximo UM motivo por mensagem. Nunca empilhe vários problemas." (Mensagens reais de WhatsApp costumam focar em um único problema, como pneu furado, e não em várias desculpas ao mesmo tempo).
    *   *Naturalness:* "Soe como mensagem real de WhatsApp: direta, um pouco vaga, sem dramatizar. Evite ao máximo colocar o nome do chefe na mensagem." (Evita a formalidade mecânica comum em prompts genéricos de IA, onde a IA costuma começar com "Prezado [Chefe], venho por meio desta informar...").
    *   *Conciseness:* "Não explique demais. Uma ou duas frases bastam."
*   **Tratamento de Repetição (Histórico):**
    *   Para garantir que o aplicativo continue gerando desculpas originais, nós enviamos no prompt as últimas 5 desculpas que o usuário já utilizou (`historico.slice(0, 5)`).
    *   Adicionamos uma instrução explícita no prompt: `Desculpas que já usei recentemente (NÃO repita, NÃO use como base)`. Isso força a IA a buscar novas vertentes criativas de desculpas toda vez que o botão é clicado, impedindo que o chefe desconfie de um padrão nas mensagens.
