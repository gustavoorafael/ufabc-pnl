# Gerador Inteligente de Testes a partir de Documentação de API

Projeto prático da disciplina **Processamento de Linguagem Natural (UFABC, 2026.Q3)**, Prof. Alexandre Donizeti Alves.

O notebook converte a documentação **OpenAPI** da [Swagger Petstore](https://petstore3.swagger.io/) em **casos de teste executáveis**. Um LLM interpreta os textos e propõe os testes; um *harness* em Python, independente do LLM, executa as requisições HTTP e **julga** cada caso.

> **Questão central:** até que ponto um LLM consegue converter documentação de API em testes úteis, mantendo a avaliação objetiva e reproduzível?

**Princípio de projeto:** o agente planeja e propõe; o harness executa e julga. "Aprovado" ou "reprovado" nunca é opinião do modelo.

## Tecnologias

| Componente | Escolha |
|---|---|
| LLM | Google Gemini 3.5 Flash-Lite (`gemini-3.5-flash-lite`) |
| Framework | LangChain (`langchain`, `langchain-google-genai`): prompts, cadeias LCEL, saídas estruturadas, retriever, agente e ferramentas |
| API / corpus | Swagger Petstore (OpenAPI 3.0): `https://petstore3.swagger.io/api/v3/openapi.json` |
| Ambiente | Google Colab (Python 3) |

## Técnicas de PLN

| Técnica (lista do enunciado) | Aplicação |
|---|---|
| **Classificação de Textos** | Classifica cada operação por intenção, risco e natureza, a partir do **texto** da descrição (o verbo HTTP não é informado ao modelo e serve depois como gabarito) |
| **Reconhecimento de Entidades Nomeadas** | Extrai recursos, identificadores, parâmetros, campos, restrições e códigos de resposta, cada um com o trecho de origem; um verificador determinístico confirma que o trecho existe na documentação |

**Extras:** RAG (retriever TF-IDF como `BaseRetriever` do LangChain, com `chunk_id` e origem de cada trecho), geração de dados sintéticos (valores válidos e inválidos a partir dos schemas) e agente com ferramentas controladas.

## Arquitetura

```
OpenAPI ─► 1 Coleta/normalização ─► 2 RAG (trechos com chunk_id)
                                        │
                     ┌──────────────────┴──────────────────┐
                     ▼                                     ▼
          3a Classificação (Gemini)             3b NER (Gemini)
                     └──────────────────┬──────────────────┘
                                        ▼
                         4 Geração de casos (Gemini, saída estruturada)
                                        ▼
                  Validação determinística do caso (contrato + evidências)
                                        ▼
                  5 Execução HTTP controlada (Python, só GET, timeout, limites)
                                        ▼
                  6 Julgamento determinístico + métricas + relatório
```

Resultados possíveis: **aprovado**, **reprovado**, **inconclusivo** (falha de rede ou instabilidade do servidor, não prova defeito da API) e **bloqueado** (caso inválido ou inseguro, não executado).

## Escopo

`GET /pet/findByStatus`, `GET /pet/{petId}` e `GET /store/inventory`.
Fora do escopo: testes de carga, autenticação avançada, operações destrutivas, mutações concorrentes e cobertura completa da API.

## Como executar

1. Abra `notebooks/PLN_PROJETO_PRÁTICO.ipynb` no Google Colab.
2. Obtenha uma chave no [Google AI Studio](https://aistudio.google.com/) e cadastre em **Secrets** (ícone de chave) com o nome `GOOGLE_API_KEY`, habilitando o acesso ao notebook.
3. Execute **todas** as células em ordem (*Ambiente de execução → Executar tudo*).

Fora do Colab: `pip install -r requirements.txt`, defina `export GOOGLE_API_KEY=...` e execute o notebook no Jupyter.

Disponibilidade, cotas e identificador do modelo devem ser verificados no momento da execução.

## Resultados

Os arquivos gerados pela execução ficam em `results/`:

| Arquivo | Conteúdo |
|---|---|
| `classificacao.json` | Classificação textual de cada operação |
| `entidades.json` | Entidades extraídas com trecho de origem |
| `casos_gerados.json` | Casos de teste (LLM, agente e proposital) com evidências e problemas de validação |
| `relatorio_execucao.csv` | Resultado de cada caso: HTTP, veredito e motivo |
| `metricas.json` | Métricas consolidadas |
| `figura_resultados.png` | Gráficos de resultados e métricas |
| `openapi_petstore.json` | Cópia do OpenAPI usada na execução |

### Métricas

*(preencher com os valores de `results/metricas.json` da execução apresentada)*

| Métrica | Valor |
|---|---|
| Casos gerados / executados / bloqueados | |
| Aprovados / reprovados / inconclusivos | |
| Cobertura de operações | |
| Validade estrutural das saídas do LLM | |
| Rastreabilidade | |
| Classificações corretas (vs. verbo HTTP) | |
| Entidades fundamentadas na documentação | |

## Limitações

* Escopo restrito a três operações `GET`.
* A Petstore é pública e compartilhada: dados e respostas variam entre execuções. Por isso, quando o estado do servidor é desconhecido, os casos aceitam todos os códigos documentados plausíveis.
* "Reprovado" indica divergência entre o comportamento observado e o contrato documentado; a causa pode estar na documentação, na implementação ou nos dados compartilhados e deve ser analisada pelos motivos do relatório.
* O vocabulário de verificações é fechado (5 tipos): garante julgamento determinístico, mas limita a expressividade dos testes.
* Saídas de LLM não são determinísticas; os JSONs em `results/` registram a execução apresentada.

## Segurança

A chave da API nunca é impressa nem salva em arquivo, e `.gitignore` bloqueia `.env` e similares. O harness só envia `GET` ao host da Petstore, a operações da lista permitida e com parâmetros documentados.

## Equipe

* Gustavo Oliveira Rafael
* Henrique Simões Bonfim
* Lívia de Pádua Ferné
* Henrique Ishida
