# Project Quantowsky

Este repositório contém a implementação, backtest e análise da estratégia **Quantowsky**, um modelo de investimento, desenvolvido em Python.

O projeto explora a ineficiência de mercado onde empresas financeiramente robustas (*Quality*), quando compradas a preços razoáveis (*Value*), tendem a superar o mercado no longo prazo, unindo análise fundamentalista contábil com otimização de portfólio baseada em risco.

## Sumário
1. [Visão Geral](#visão-geral)
2. [Tese de Investimento](#tese-de-investimento)
3. [Metodologia e Modelagem](#metodologia-e-modelagem)
    - [Fase 1: Seleção (Piotroski F-Score)](#fase-1-seleção-piotroski-f-score)
    - [Fase 2: Alocação (Risk Parity)](#fase-2-alocação-risk-parity)
4. [Regras de Portfólio e Backtest](#regras-de-portfólio-e-backtest)
5. [Resultados](#resultados)
6. [Engenharia de Dados](#engenharia-de-dados)

---

## Visão Geral

| Parâmetro | Detalhe |
| :--- | :--- |
| **Estratégia** | Portfólio |
| **Classe de Ativos** | Ações (Universo IBOVESPA) |
| **Benchmark** | IBOV  |
| **Rebalanceamento** | Anual (1º de Maio) |
| **Tecnologia** | Python (Pandas, Scipy, Yfinance)|

---

## Tese de Investimento

A estratégia baseia-se na adaptação e união de dois modelos acadêmicos consagrados:

1.  **Piotroski F-Score (Joseph Piotroski):** Uma abordagem fundamentalista de *Value Investing* que utiliza 9 critérios contábeis para avaliar a solidez financeira de uma empresa (score 0 a 9). A hipótese é que este filtro separa efetivamente "vencedores" de "perdedores".
2.  **Risk Parity (Markowitz/MPT):** Uma abordagem quantitativa de alocação. Em vez de dividir o capital igualmente ($1/N$), o modelo aloca pesos de forma que cada ativo contribua com a mesma quantidade de risco para a carteira, aumentando a resiliência do portfólio.

3.  **Objetivo:** Construir um portfólio que capture o prêmio do fator "Qualidade" enquanto otimiza a distribuição de risco, visando gerar retornos superiores ajustados ao risco (*Sharpe Ratio*) no mercado brasileiro.

---

## Metodologia e Modelagem

O algoritmo opera em duas fases distintas e sequenciais.

### Fase 1: Seleção (Piotroski F-Score)

Nesta etapa, ocorre um processo de filtragem robusta para definir o "Universo Investível" e selecionar os ativos de maior qualidade.

1. **Filtro Setorial:** Exclusão de empresas do setor financeiro (Bancos/Seguradoras), pois a estrutura de DRE não é compatível com o F-Score original.
2. **Filtro de Liquidez:** Seleção de ativos com volume médio diário > R$ 500k ou R$ 1M (parametrizável) para garantir a execução.
3. **Cálculo do Score:** Aplicação dos 9 critérios binários baseados nas demonstrações financeiras anuais:



| Categoria | Critérios |
| :--- | :--- |
| **Rentabilidade** | 1. Lucro Líquido Positivo <br> 2. Fluxo de Caixa Operacional (FCO) Positivo <br> 3. ROA Crescente <br> 4. Qualidade do Lucro ($FCO > Lucro Líquido$) |
| **Alavancagem** | 5. Redução da Alavancagem <br> 6. Aumento da Liquidez Corrente <br> 7. Sem Diluição de Ações |
| **Eficiência** | 8. Aumento da Margem Bruta <br> 9. Aumento do Giro do Ativo |

*Seleção Final:* O algoritmo seleciona o *Top N* empresas (ex: 10 ou 15) com as maiores pontuações para compor a carteira.

### Fase 2: Alocação (Risk Parity)

Após a seleção, a alocação de pesos não é igualitária. Utilizamos a paridade de risco (ERC - *Equal Risk Contribution*).

Para isso, calculamos a Matriz de Covariância ($\Sigma$) usando retornos diários históricos (*Lookback* de 252 dias). O problema de otimização busca encontrar o vetor de pesos $w$ onde a contribuição marginal de risco de cada ativo seja idêntica.

A condição ideal buscada é:

$$w_{i} \cdot (\Sigma w)_{i} = w_{j} \cdot (\Sigma w)_{j}$$

Onde a contribuição de risco do ativo $i$ é dada pelo seu peso multiplicado pela sua volatilidade marginal no portfólio. Otimizamos para minimizar a soma dos erros quadrados dessa diferença utilizando `scipy.optimize`.

---

## Regras de Portfólio e Backtest

Para garantir a robustez e replicabilidade da estratégia, foram definidas regras estritas:

* **Prevenção de Lookahead Bias:** A data de rebalanceamento é fixada em **1º de Maio** de cada ano. Isso garante que os dados contábeis do ano anterior (publicados até abril) já estejam auditados e disponíveis publicamente.
* **Custos de Transação:** Foi aplicado um custo de 0.002 (0.2%) sobre o *turnover* da carteira a cada rebalanceamento para simular custos reais de execução.
* **Janela de Análise:** 2015 a 2024.

---

## Resultados

O backtest demonstrou que a estratégia superou o Benchmark (Ibovespa) em retorno total, volatilidade e *Sharpe Ratio*. A performance foi otimizada ao reduzir o filtro de liquidez para incluir *Small/Mid Caps*, onde o fator F-Score tende a encontrar maior alfa.

### Performance Comparativa (2015-2024)

| Métrica | Estratégia (Quantowsky) | Benchmark (IBOV) |
| :--- | :---: | :---: |
| **Retorno Total** | **238.56%** | 123.95% |
| **Retorno Médio Anual** | **14.53%** | 9.38% |
| **Volatilidade Anual** | **22.53%** | 24.67% |
| **Sharpe Ratio** | **0.64** | 0.38 |
| **Max Drawdown** | -46.89% | -46.82% |

*(Resultados baseados no cenário com Liquidez Mínima de R$ 0.5M)*.

## Engenharia de Dados

A construção do *dataset* foi um desafio central do projeto, dado que fontes públicas raramente oferecem dados contábeis estruturados para backtest histórico.

* **Fontes:** Portal de Dados Abertos da CVM (Demonstrações Financeiras Padronizadas) e Yahoo Finance (Preços).
* **Tratamento de Dados:**
    * Extração granular de múltiplos arquivos (DRE, BPA, BPP, DFC).
    * Mapeamento de contas contábeis e tradução de `CNPJ` para `Ticker` utilizando scripts auxiliares e suporte de IA para padronização.
    * Limpeza de dados para garantir a consistência temporal dos balanços.

---

*Disclaimer: Este projeto tem fins estritamente educacionais e acadêmicos. Os resultados apresentados são baseados em simulações (backtest) e não constituem recomendação de investimento.*