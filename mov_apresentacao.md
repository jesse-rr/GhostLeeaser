# Explicação do Projeto Deu Zebra & Contribuições de Jesse

Este documento apresenta uma explicação abrangente sobre a arquitetura, estrutura geral do projeto **Deu Zebra** e detalha minuciosamente as contribuições individuais desenvolvidas por **Jesse**, com links de arquivos, descrições e snippets de código.

---

## 1. Visão Geral e Arquitetura do Projeto

O aplicativo **Deu Zebra** foi desenvolvido como um MVP (Minimum Viable Product) móvel utilizando **React Native**, **Expo** e **TypeScript**, em total conformidade com a especificação do [PDF de Requisitos](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/docs/MOV-projeto-2.pdf).

O objetivo principal do app é gerar desculpas contextuais plausíveis para que trabalhadores informem imprevistos aos seus chefes. A geração das desculpas é potencializada pela **Gemini API** da Google (modelo `gemini-2.5-flash`), alimentada por dados reais coletados do dispositivo (localização GPS e dados climáticos da API Open-Meteo) combinados com perfis persistidos do usuário e do seu respectivo chefe.

### Estrutura de Diretórios do Projeto

Abaixo está o mapeamento da estrutura de diretórios em `src/`, seguindo as diretrizes de organização e responsabilidades:

*   **[src/components/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/components)**: Componentes de interface visual reaproveitáveis (como botões customizados, caixas de texto estruturadas, estados vazios e modais).
*   **[src/constants/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/constants)**: Arquivos de definições de constantes globais (chaves do AsyncStorage, tons das desculpas e sugestões de formulários).
*   **[src/context/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/context)**: Provedores de estado global (Context API) para controle dos perfis salvos, histórico de desculpas e configurações.
*   **[src/hooks/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/hooks)**: Hooks React personalizados de consumo de contextos e abstrações de lógica de negócios (como geolocalização, busca de clima e controle de geração).
*   **[src/navigation/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/navigation)**: Definição dos navegadores do aplicativo (Tab Navigator inferior e Stack Navigators de fluxos).
*   **[src/screens/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/screens)**: Telas do aplicativo (geração, histórico, configurações e cadastros de perfil).
*   **[src/services/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/services)**: Integrações com APIs externas (Open-Meteo, Gemini REST API, ViaCEP) e APIs nativas do dispositivo (localização GPS, persistência local e notificações).
*   **[src/theme/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/theme)**: Sistema de design visual base (tokens de cores, espaçamentos e fontes tipográficas).
*   **[src/types/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/types)**: Tipagens e contratos TypeScript do domínio do sistema.
*   **[src/utils/](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/utils)**: Lógica utilitária pura, incluindo validações de campo e montagem de prompts textuais para a IA.

---

## 2. Contribuições Individuais de Jesse

Jesse foi responsável por projetar e implementar as fundações do contexto ambiental (localização GPS, clima Open-Meteo), a lógica de comunicação de texto com a API do Gemini (serviço, hooks, e a interface gráfica de geração), validações robustas de formulários, autocompletar automático usando a API ViaCEP e a estruturação do prompt de IA inicial.

Abaixo, detalhamos cada uma dessas contribuições:

---

### A. Localização GPS Nativa

Para obter as coordenadas de latitude e longitude do dispositivo do usuário de forma nativa e segura, Jesse implementou o serviço de localização utilizando o SDK `expo-location`.

#### Arquivo: [localizacaoService.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/services/localizacaoService.ts)
Este arquivo solicita a permissão de geolocalização em primeiro plano. Caso o acesso seja concedido, obtém a posição atual do aparelho; se negado, dispara um erro tipado.

```typescript
import * as Location from "expo-location";
import { Coordenadas } from "../types/clima";

export const ERRO_PERMISSAO_LOCALIZACAO = "PERMISSAO_LOCALIZACAO_NEGADA";

async function obterCoordenadas(): Promise<Coordenadas> {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== Location.PermissionStatus.GRANTED) {
    throw new Error(ERRO_PERMISSAO_LOCALIZACAO);
  }

  const posicao = await Location.getCurrentPositionAsync();
  return {
    latitude: posicao.coords.latitude,
    longitude: posicao.coords.longitude,
  };
}

export const localizacaoService = { obterCoordenadas };
```

#### Arquivo: [useLocalizacao.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/hooks/useLocalizacao.ts)
Este hook gerencia o ciclo de vida e estado (carregando, coordenadas salvas e mensagens de erro) para facilitar o consumo da geolocalização na interface.

```typescript
import { useState } from "react";
import { Coordenadas } from "../types/clima";
import { localizacaoService } from "../services/localizacaoService";

interface EstadoLocalizacao {
  coordenadas: Coordenadas | null;
  carregando: boolean;
  erro: string | null;
}

export function useLocalizacao() {
  const [estado, setEstado] = useState<EstadoLocalizacao>({
    coordenadas: null,
    carregando: false,
    erro: null,
  });

  async function obterLocalizacao() {
    setEstado({ coordenadas: null, carregando: true, erro: null });
    try {
      const coordenadas = await localizacaoService.obterCoordenadas();
      setEstado({ coordenadas, carregando: false, erro: null });
      return coordenadas;
    } catch (e) {
      const mensagem = e instanceof Error ? e.message : "Nao foi possivel obter a localizacao";
      setEstado({ coordenadas: null, carregando: false, erro: mensagem });
      return null;
    }
  }

  return { ...estado, obterLocalizacao };
}
```

---

### B. Condições Climáticas (Open-Meteo API)

Para enriquecer o contexto das desculpas, Jesse implementou o consumo da API climática pública Open-Meteo, mapeando códigos meteorológicos técnicos para descrições humanizadas em português.

#### Arquivo: [climaService.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/services/climaService.ts)
Este serviço realiza uma requisição REST `fetch` para a API Open-Meteo passando a latitude e longitude, validando a integridade dos dados de retorno e traduzindo o `weather_code` obtido.

```typescript
import { Coordenadas, ResumoClima } from "../types/clima";

// Dicionário parcial do mapeamento de códigos climáticos da WMO
const descricoesPorCodigo: Record<number, string> = {
  0: "Céu limpo",
  1: "Predominantemente limpo",
  2: "Parcialmente nublado",
  3: "Nublado",
  61: "Chuva fraca",
  63: "Chuva moderada",
  65: "Chuva forte",
  95: "Trovoada",
  // ... outros códigos mapeados
};

function descreverTempo(codigo: number): string {
  return descricoesPorCodigo[codigo] ?? "Condição desconhecida";
}

async function buscarClima(coordenadas: Coordenadas): Promise<ResumoClima> {
  const url =
    "https://api.open-meteo.com/v1/forecast" +
    `?latitude=${coordenadas.latitude}` +
    `&longitude=${coordenadas.longitude}` +
    "&current=temperature_2m,weather_code";

  const resposta = await fetch(url);
  if (!resposta.ok) {
    throw new Error("Falha ao buscar o clima.");
  }

  const dados = await resposta.json();
  const codigoTempo = dados.current.weather_code;

  return {
    cidade: "",
    temperatura: dados.current.temperature_2m,
    codigoTempo,
    descricao: descreverTempo(codigoTempo),
  };
}

export const climaService = { buscarClima };
```

#### Arquivo: [useClima.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/hooks/useClima.ts)
Este hook consome o serviço do clima e realiza a geolocalização reversa (com `Location.reverseGeocodeAsync` ou OpenStreetMap Nominatim como contingência) para traduzir as coordenadas GPS no nome real da cidade do usuário.

```typescript
import { useState } from "react";
import { Coordenadas } from "../types/clima";
import { climaService } from "../services/climaService";
import * as Location from "expo-location";

export function useClima() {
  const [estado, setEstado] = useState<EstadoClima>({ clima: null, carregando: false, erro: null });

  async function buscarClima(coordenadas: Coordenadas) {
    setEstado({ clima: null, carregando: true, erro: null });
    try {
      const clima = await climaService.buscarClima(coordenadas);
      let cidade = "Desconhecido";
      try {
        const enderecos = await Location.reverseGeocodeAsync({
          latitude: coordenadas.latitude,
          longitude: coordenadas.longitude,
        });
        if (enderecos && enderecos.length > 0) {
          cidade = enderecos[0].city || "Desconhecido";
        }
      } catch {}

      // Fallback geocode API pública caso o nativo falhe
      if (cidade === "Desconhecido") {
        const res = await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&lat=${coordenadas.latitude}&lon=${coordenadas.longitude}`);
        if (res.ok) {
          const json = await res.json();
          cidade = json.address?.city || "Desconhecido";
        }
      }

      const climaComCidade = { ...clima, cidade };
      setEstado({ clima: climaComCidade, carregando: false, erro: null });
      return climaComCidade;
    } catch (e) {
      setEstado({ clima: null, carregando: false, erro: "Erro ao buscar clima" });
      return null;
    }
  }

  return { ...estado, buscarClima };
}
```

---

### C. Integração com Gemini API para Geração de Texto

Jesse implementou o canal de comunicação direto com a API REST do Gemini, integrando a montagem do prompt dinâmico e fornecendo mecanismos de redundância caso a API não responda.

#### Arquivo: [geminiService.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/services/geminiService.ts)
Realiza uma requisição do tipo POST enviando o prompt formatado. Caso não haja chave configurada ou ocorra algum erro na comunicação, o serviço aciona de forma transparente um banco local de desculpas genéricas estruturadas (`desculpaFallbackService`), garantindo resiliência total na experiência do usuário.

```typescript
import { EntradaGeracao } from "../types/desculpa";
import { montarPromptDesculpa } from "../utils/montarPrompt";
import { obterGeminiApiKey } from "../utils/geminiConfig";
import { desculpaFallbackService } from "./desculpaFallbackService";

const MODELO = "gemini-2.5-flash";

async function chamarGeminiTexto(prompt: string, apiKey: string): Promise<string> {
  const url = `https://generativelanguage.googleapis.com/v1beta/models/${MODELO}:generateContent?key=${apiKey}`;
  const resposta = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      contents: [{ parts: [{ text: prompt }] }],
    }),
  });

  if (!resposta.ok) throw new Error("Erro de comunicação com Gemini");
  const resultado = await resposta.json();
  return resultado.candidates?.[0]?.content?.parts?.[0]?.text.trim();
}

async function gerarDesculpa(entrada: EntradaGeracao): Promise<string[]> {
  const apiKey = obterGeminiApiKey();
  if (!apiKey) {
    // Mecanismo de Fallback local
    return [desculpaFallbackService.obterDesculpaFallback(entrada.tom, [])];
  }

  const prompt = montarPromptDesculpa(entrada);
  try {
    const resposta = await chamarGeminiTexto(prompt, apiKey);
    // Suporte a múltiplas desculpas separadas por ---
    return resposta.split(/\n-{3,}\n/).map(p => p.trim()).filter(Boolean);
  } catch {
    return [desculpaFallbackService.obterDesculpaFallback(entrada.tom, [])];
  }
}
```

#### Arquivo: [useGerarDesculpa.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/hooks/useGerarDesculpa.ts)
Este hook consolida os perfis persistidos do usuário (`usePerfil`), a lista histórica de desculpas salvas (`useDesculpas`) e o clima atual para criar a requisição unificada enviada ao Gemini, instanciando uma entidade `Desculpa` com IDs gerados dinamicamente.

```typescript
import { useState } from "react";
import { Desculpa, EntradaGeracao, TomDesculpa } from "../types/desculpa";
import { ResumoClima } from "../types/clima";
import { geminiService } from "../services/geminiService";
import { usePerfil } from "./usePerfil";
import { useDesculpas } from "./useDesculpas";

export function useGerarDesculpa() {
  const { perfilPessoa, perfilChefe } = usePerfil();
  const { desculpas } = useDesculpas();
  const [estado, setEstado] = useState({ desculpasGeradas: [], carregando: false, erro: null });

  async function gerar(tom: TomDesculpa, clima: ResumoClima | null, situacao?: string, quantidade = 1) {
    if (!perfilPessoa || !perfilChefe) {
      setEstado({ desculpasGeradas: [], carregando: false, erro: "Configure os perfis primeiro" });
      return null;
    }
    setEstado({ desculpasGeradas: [], carregando: true, erro: null });

    const entrada: EntradaGeracao = { perfilPessoa, perfilChefe, clima, historico: desculpas, tom, situacao, quantidade };

    try {
      const textos = await geminiService.gerarDesculpa(entrada);
      const novas = textos.map(texto => ({
        id: `${Date.now()}-${Math.floor(Math.random() * 1000)}`,
        texto,
        tom,
        criadaEm: new Date().toISOString(),
        avaliacao: null,
        favorita: false,
        contextoClima: clima,
        imagemUri: null,
      }));
      setEstado({ desculpasGeradas: novas, carregando: false, erro: null });
      return novas;
    } catch (e) {
      setEstado({ desculpasGeradas: [], carregando: false, erro: "Falha na geração" });
      return null;
    }
  }

  return { ...estado, gerar };
}
```

---

### D. Tela de Geração de Desculpas

Jesse desenvolveu a lógica e visual da tela principal de geração.

#### Arquivo: [GerarDesculpaScreen.tsx](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/screens/GerarDesculpaScreen.tsx)
Na montagem do componente, a tela dispara de forma paralela e em segundo plano a solicitação da localização e a leitura das condições do clima para exibir um feedback visual e atualizar o estado contextual. Adiciona feedbacks visuais dinâmicos de carregamento e transições utilizando a API de animações nativa `Animated` (com interpolações de rotação contínua para o ícone de carregamento).

```typescript
import React, { useEffect, useRef, useState } from "react";
import { SafeAreaView, ScrollView, Animated, Pressable, Text, View } from "react-native";
import { useLocalizacao } from "../hooks/useLocalizacao";
import { useClima } from "../hooks/useClima";
import { useGerarDesculpa } from "../hooks/useGerarDesculpa";

export function GerarDesculpaScreen() {
  const { obterLocalizacao, erro: erroLoc } = useLocalizacao();
  const { buscarClima, clima } = useClima();
  const { gerar, carregando: carregandoGeracao } = useGerarDesculpa();
  const animacaoRotacao = useRef(new Animated.Value(0)).current;

  // Carrega localização e clima no mount
  async function carregarContexto() {
    const coords = await obterLocalizacao();
    if (coords) await buscarClima(coords);
  }

  useEffect(() => {
    carregarContexto();
  }, []);

  // Controla o loop de rotação contínua da varinha mágica/ícone durante a geração
  useEffect(() => {
    let animacao: Animated.CompositeAnimation | null = null;
    if (carregandoGeracao) {
      animacaoRotacao.setValue(0);
      animacao = Animated.loop(
        Animated.timing(animacaoRotacao, {
          toValue: 1,
          duration: 1000,
          useNativeDriver: true,
        })
      );
      animacao.start();
    } else {
      animacaoRotacao.setValue(0);
    }
    return () => animacao?.stop();
  }, [carregandoGeracao]);

  return (
    <SafeAreaView>
      {/* Exibição dos estados climáticos, inputs de tom/situação e disparador da animação */}
    </SafeAreaView>
  );
}
```

---

### E. Validações de Formulário

A fim de evitar que perfis incompletos corrompam a estrutura do prompt enviado ao Gemini, Jesse implementou rotinas de validação de formulários simples e eficazes.

#### Arquivo: [validacao.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/utils/validacao.ts)
Define as funções auxiliares puras para checagem de textos preenchidos e tamanhos mínimos.

```typescript
function textoPreenchido(valor: string): boolean {
  return valor.trim().length > 0;
}

function tamanhoMinimo(valor: string, minimo: number): boolean {
  return valor.trim().length >= minimo;
}

export { textoPreenchido, tamanhoMinimo };
```

Essas validações foram acopladas de maneira síncrona nos formulários das telas de configuração:

1.  **[PerfilPessoaScreen.tsx](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/screens/PerfilPessoaScreen.tsx)**: Valida a digitação correta do nome do usuário e sua profissão antes de salvar as informações e prosseguir:
    ```typescript
    if (!textoPreenchido(formulario.nome)) {
      novosErros.nome = "O nome é obrigatório.";
    }
    if (!textoPreenchido(formulario.profissao)) {
      novosErros.profissao = "A profissão é obrigatória.";
    }
    ```
2.  **[PerfilChefeScreen.tsx](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/screens/PerfilChefeScreen.tsx)**: Garante a presença do nome do chefe do usuário:
    ```typescript
    if (!textoPreenchido(formulario.nome)) {
      setErroNome("O nome do chefe e obrigatorio.");
      return;
    }
    ```

---

### F. Autocompletar de CEP (ViaCEP API)

Para simplificar o cadastro do perfil do chefe do usuário, Jesse integrou o consumo da API do ViaCEP para autopreencher a localização do local de trabalho de forma automática a partir do CEP.

#### Arquivo: [PerfilChefeScreen.tsx](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/screens/PerfilChefeScreen.tsx)
Ao identificar a entrada completa de um CEP (8 dígitos numéricos), o aplicativo aciona a API e extrai os dados geográficos, preenchendo automaticamente a cidade e o respectivo estado correspondente no campo `localizacaoEmpresa`.

```typescript
interface RespostaViaCep {
  localidade?: string;
  uf?: string;
  erro?: boolean;
}

async function buscarCep(valor: string) {
  const apenasNumeros = valor.replace(/\D/g, "");
  if (apenasNumeros.length > 8) return;
  setCep(apenasNumeros);
  setErroCep(undefined);

  if (apenasNumeros.length !== 8) return;

  setBuscandoCep(true);
  try {
    const resposta = await fetch(`https://viacep.com.br/ws/${apenasNumeros}/json/`);
    if (!resposta.ok) throw new Error();

    const dados: RespostaViaCep = await resposta.json();
    if (dados.erro) {
      setErroCep("CEP nao encontrado.");
      return;
    }

    // Auto-preenchimento
    atualizarCampo("localizacaoEmpresa", `${dados.localidade} - ${dados.uf}`);
  } catch {
    setErroCep("Nao foi possivel buscar o CEP. Verifique a conexao.");
  } finally {
    setBuscandoCep(false);
  }
}
```

---

### G. Estruturação do Prompt de Inteligência Artificial

O prompt dinâmico que instrui a inteligência artificial a agir como o gerador de desculpas é a inteligência central do sistema. Jesse desenvolveu a primeira versão e as regras de restrição do prompt no arquivo utilitário.

#### Arquivo: [montarPrompt.ts](file:///home/jrr/IdeaProjects/projeto-2-du-dudu-e-edu/src/utils/montarPrompt.ts)
A função `montarPromptDesculpa` injeta dinamicamente as variáveis de ambiente (como clima atual do local do usuário, pistas de vida aleatórias para evitar repetições como se o usuário possui pets ou pais idosos, histórico recente de desculpas para a IA não se repetir e a situação opcional especificada pelo usuário), impondo restrições rígidas para que o texto gerado pareça uma mensagem autêntica do WhatsApp (evitando redundâncias e linguagens formais excessivas).

```typescript
import { EntradaGeracao } from "../types/desculpa";

function montarPromptDesculpa(entrada: EntradaGeracao): string {
  const { perfilPessoa, perfilChefe, clima, tom, historico } = entrada;
  const quantidade = entrada.quantidade ?? 1;
  const pistas = montarPistasDeVida(perfilPessoa, clima);

  const pedidoQuantidade = quantidade > 1
    ? `Escreva ${quantidade} mensagens DIFERENTES que eu mandaria pro meu chefe explicando por que nao vou conseguir cumprir algo hoje.`
    : `Escreva uma mensagem curta que eu mandaria pro meu chefe explicando por que nao vou conseguir cumprir algo hoje.`;

  let prompt = `${pedidoQuantidade}
Tom: ${tom}.

Eu me chamo ${perfilPessoa.nome} e trabalho como ${perfilPessoa.profissao}.
Meu chefe se chama ${perfilChefe.nome}${perfilChefe.empresa ? `, da ${perfilChefe.empresa}` : ""}${perfilChefe.localizacaoEmpresa ? ` (${perfilChefe.localizacaoEmpresa})` : ""}.

Regras importantes:
- Use NO MAXIMO UM motivo por mensagem. Nunca empilhe varios problemas.
- Soe como mensagem real de WhatsApp: direta, um pouco vaga, sem dramatizar.
- Evite ao maximo colocar o nome do chefe (${perfilChefe.nome}) na mensagem.
- Varie bastante o comeco e o estilo da mensagem para nao soar repetitivo.
- Nao invente nomes, datas ou compromissos que eu nao mencionei.
- Nao explique demais. Uma ou duas frases bastam.
- SEJA CRIATIVO e ORIGINAL.
`;

  if (clima) {
    prompt += `\nClima agora em ${clima.cidade}: ${clima.descricao}, ${clima.temperatura}°C.`;
  }

  // Anexa histórico para instruir explicitamente o modelo a não se repetir
  const historicoRecente = historico.slice(0, 5);
  if (historicoRecente.length > 0) {
    prompt += `\nDesculpas que ja usei recentemente (NAO repita, NAO use como base):`;
    for (const item of historicoRecente) {
      prompt += `\n- "${item.texto}"`;
    }
  }

  return prompt;
}
```

---

## 3. Resumo de Impacto das Contribuições de Jesse

Ao combinar hardware nativo (GPS via Expo), APIs Web (Open-Meteo, ViaCEP), e inteligência generativa (Gemini API com engenharia de prompts avançada), Jesse estruturou todo o motor central e os fluxos primários de contexto que diferenciam o **Deu Zebra** de um simples gerador estático de textos. Suas validações e tratamento de erros proporcionaram robustez ao fluxo geral do MVP.
