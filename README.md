# Deduplicação de bug reports com LLM como juiz

Pipeline em Google Colab que detecta bug reports duplicados em uma base de issues do Mozilla Core. A cada issue nova, o sistema recupera candidatas por busca híbrida (BM25 + embeddings), pede a um LLM que julgue cada par e, se encontrar uma duplicata, aponta a issue original. Depois a issue entra na base, como em um gestor de projetos (Jira, Bugzilla).

O repositório reúne o notebook, o dataset original, a amostra usada e os resultados da execução com **base de 10.000 issues e teste de 1.000 issues**, que inclui a comparação com baselines sem LLM.

## Sumário

1. [Conteúdo do repositório](#1-conteúdo-do-repositório)
2. [Como o pipeline funciona](#2-como-o-pipeline-funciona)
3. [Dados e amostragem](#3-dados-e-amostragem)
4. [Como executar](#4-como-executar)
5. [Configuração da execução reportada](#5-configuração-da-execução-reportada)
6. [Protocolo de avaliação](#6-protocolo-de-avaliação)
7. [Resultados (base 10.000, teste 1.000)](#7-resultados-base-10000-teste-1000)
8. [Descrição dos arquivos de resultado](#8-descrição-dos-arquivos-de-resultado)
9. [Limitações](#9-limitações)
10. [Trabalhos futuros](#10-trabalhos-futuros)

---

## 1. Conteúdo do repositório

> Ajuste os nomes das pastas à organização que você escolher; a estrutura abaixo é uma sugestão.

```
.
├── README.md
├── pipeline_dedup_bug_reports.ipynb      # notebook completo (amostragem, pipeline e avaliação)
├── dados/
│   └── mozilla_core.csv                  # dataset original completo (Mozilla Core)
└── resultados/
    └── 10k/
        ├── amostra_1100.csv              # amostra usada na execução (base + teste, ver nota abaixo)
        ├── metricas.json
        ├── comparacao_sistemas.csv
        ├── matrizes_confusao.png
        ├── resultados_avaliacao.csv
        ├── erros_fp_fn.csv
        └── base_total.csv
```

**Nota sobre o nome da amostra.** O notebook grava a amostra como `amostra_1100.csv` mesmo quando `N_BASE` e `N_EVAL` são alterados. Na execução reportada ela contém **11.000 issues** (10.000 de base + 1.000 de teste), não 1.100.

O CSV original completo (`mozilla_core.csv`, centenas de milhares de issues) fica em `dados/`. A amostra de 11.000 issues usada na execução fica junto dos resultados, em `resultados/10k/`, e foi gerada a partir dele pelo próprio notebook (semente 42). Como o CSV original é grande, confira o limite de tamanho de arquivo do GitHub (100 MB por arquivo) e use Git LFS se necessário.

## 2. Como o pipeline funciona

| Etapa | O que faz | Escolha |
|---|---|---|
| Pré-processamento | Mantém só o relato original; remove ruído (ver abaixo) | Expressões regulares |
| Recuperação híbrida | BM25 incremental + embeddings densos, fundidos por RRF; ficam as 5 melhores candidatas | `BAAI/bge-base-en-v1.5` |
| Juiz LLM | Para cada candidata, pergunta se as duas issues descrevem o mesmo defeito e lê P("Yes") dos logits em uma só passada, sem gerar texto | `Qwen/Qwen3-8B`, 4 bits (nf4), cabe em uma GPU T4 gratuita |
| Decisão | Duplicata se a maior P("Yes") entre as candidatas for ≥ limiar | Limiar 0,5 |
| Inserção | A issue entra na base com o campo `duplicate_of` | Base cresce durante a simulação |

### Pré-processamento

- Mantém apenas o **relato original**. Comentários a partir de "Updated by" são cortados, pois podem conter frases como "this bug is a duplicate of N" e vazar o rótulo.
- Remove o cabeçalho "Created by ... Additional Details", e-mails, telefones, IPs, endereços hexadecimais e linhas decorativas ASCII.
- Tokenização separa camelCase e snake_case.
- O título pesa o dobro no BM25.
- Limites de texto: 2.000 caracteres para embeddings e 900 para o juiz.

### Recuperação híbrida

- **BM25** com índice invertido atualizado a cada issue inserida (k1 = 1,5; b = 0,75).
- **Embeddings** normalizados, comparados por cosseno.
- **RRF** (Reciprocal Rank Fusion, constante 60) sobre as 50 melhores de cada lista; as 5 primeiras seguem para o juiz.

### Juiz e decisão

O juiz recebe a issue nova e uma candidata e responde "Yes" ou "No". A probabilidade de "Yes" é calculada a partir dos logits dessas duas respostas. A issue é considerada duplicata se a maior probabilidade entre as 5 candidatas atingir o limiar, e a candidata de maior probabilidade é apontada como original. As respostas do juiz ficam em um cache (`.jsonl`), o que permite retomar uma execução interrompida sem repetir chamadas.

### Limiares

O limiar do juiz e os dos baselines foram calibrados com **60 issues da própria base**, por exclusão de uma issue por vez, e nunca com issues de teste.

## 3. Dados e amostragem

A amostragem é feita **dentro do notebook**, a partir do CSV completo, em duas passadas: primeiro só as colunas leves (`Issue_id`, `Duplicated_issue`, `Resolution`, `Title`, `Created_time`) para escolher as issues; depois, em blocos, só as linhas escolhidas com todas as colunas.

- **Clusters.** As duplicatas são agrupadas em clusters, resolvendo cadeias de duplicatas até a issue raiz.
- **Base:** 10.000 issues, sendo 5.000 duplicadas e 5.000 não duplicadas.
- **Teste:** 1.000 issues diferentes das da base, sendo 500 duplicadas e 500 não duplicadas.
- **Onde fica a original de cada duplicata de teste** (`FRAC_ORIG_TESTE = 0,5`):
  - em 250 casos, a original está **na base** (a duplicata de teste é uma duplicata a mais desse cluster, no máximo uma por cluster);
  - em 250 casos, a original está **no próprio teste**, como uma das não duplicadas, e é sempre criada antes da duplicata. Esses clusters não têm nenhuma issue na base.
- As demais não duplicadas de teste são issues isoladas, que nenhuma outra issue do dataset aponta, o que evita duplicatas ocultas entre elas.
- As issues são ordenadas por data (UTC), e a simulação as processa nessa ordem.

Colunas adicionadas à amostra: `is_duplicate`, `master_id`, `split` (`base` ou `eval`) e `orig_local` (`base` ou `teste`).

## 4. Como executar

1. Abra `pipeline_dedup_bug_reports.ipynb` no Google Colab e selecione uma GPU (T4 gratuita é suficiente).
2. Obtenha `dados/mozilla_core.csv` do repositório e coloque-o em `/content`, ou na pasta do Drive configurada (`MyDrive/dedup_bugs`), ou envie o arquivo quando o notebook pedir. Com a mesma semente e a mesma configuração, o notebook gera uma amostra equivalente à de `resultados/10k/amostra_1100.csv`.
3. Ajuste a seção de configuração, se quiser: `N_BASE`, `N_EVAL`, `FRAC_ORIG_TESTE`, `TOP_K`, `SEED` e `SALVAR_NO_DRIVE`.
4. Execute as células em ordem. Com `SALVAR_NO_DRIVE = True` o notebook monta o Google Drive (pede autorização na primeira vez).
5. Cada execução grava seus arquivos em `execucoes/<data_hora>/` dentro da pasta de saída. O cache do juiz fica na raiz da pasta de saída e é reaproveitado entre execuções.

Para reproduzir a execução reportada, use os valores da seção 5. O tempo da simulação com o teste de 1.000 issues foi de aproximadamente **51,6 minutos**.

Se o Drive estiver desligado, a saída vai para `/content/saida`, que é apagada ao fim da sessão do Colab.

## 5. Configuração da execução reportada

| Parâmetro | Valor |
|---|---|
| `SEED` | 42 |
| `N_BASE` / `N_EVAL` | 10.000 / 1.000 |
| `FRAC_ORIG_TESTE` | 0,5 |
| Modelo de embeddings | `BAAI/bge-base-en-v1.5` |
| Modelo juiz | `Qwen/Qwen3-8B` (4 bits) |
| `TOP_K` / `POOL` / `RRF_K` | 5 / 50 / 60 |
| `ALPHA_HIB` (híbrido linear) | 0,5 |
| `N_CAL` | 60 |
| `MAX_CHARS_EMB` / `MAX_CHARS_LLM` | 2.000 / 900 |
| `PROMPT_VERSION` | v1 |
| Limiares calibrados | juiz 0,5; BM25 0,1498; embeddings 0,8176; híbrido linear 0,4917 |

Ao fim da simulação a base chega a 11.000 issues (10.000 iniciais mais as 1.000 de teste inseridas).

## 6. Protocolo de avaliação

Quatro sistemas decidem os mesmos 1.000 casos de teste:

| Sistema | Como decide |
|---|---|
| **Híbrido + LLM juiz** | Recuperação híbrida, 5 candidatas e juiz Qwen3-8B |
| **BM25** | Maior similaridade BM25 (normalizada pelo limite teórico) contra a base inteira |
| **Embeddings** | Maior cosseno contra a base inteira |
| **Híbrido linear** | 0,5 × BM25 normalizado + 0,5 × cosseno, contra a base inteira |

Métricas: matriz de confusão (classe positiva = duplicata), acurácia com IC de 95% por bootstrap, precisão, recall, F1, AUC, AP e o **F1 estrito**, em que o acerto só conta se a original apontada está no cluster correto. A significância das diferenças usa teste de McNemar e bootstrap pareado.

## 7. Resultados (base 10.000, teste 1.000)

### Comparação entre os sistemas

| Sistema | VP | VN | FP | FN | Acurácia [IC 95%] | Precisão | Recall | F1 | AUC | AP |
|---|---|---|---|---|---|---|---|---|---|---|
| **Híbrido + LLM juiz** | 308 | 400 | 100 | 192 | **0,708** [0,68, 0,73] | 0,755 | 0,616 | 0,678 | 0,768 | 0,770 |
| BM25 | 270 | 272 | 228 | 230 | 0,542 [0,51, 0,57] | 0,542 | 0,540 | 0,541 | 0,575 | 0,578 |
| Embeddings | 227 | 401 | 99 | 273 | 0,628 [0,60, 0,66] | 0,696 | 0,454 | 0,550 | 0,709 | 0,710 |
| Híbrido linear | 188 | 424 | 76 | 312 | 0,612 [0,58, 0,64] | 0,712 | 0,376 | 0,492 | 0,666 | 0,674 |

### F1 estrito (exige apontar a original correta)

| Sistema | Precisão | Recall | F1 |
|---|---|---|---|
| Híbrido + LLM juiz | 0,55 | 0,45 | **0,49** |
| BM25 | 0,27 | 0,27 | 0,27 |
| Embeddings | 0,45 | 0,29 | 0,35 |
| Híbrido linear | 0,51 | 0,27 | 0,35 |

No caso do juiz, 408 issues foram marcadas como duplicatas: 224 com a original correta, 100 alarmes falsos e 84 duplicatas reais com a original errada.

### Testes pareados (juiz contra cada baseline)

| Comparação | Δ acurácia | p-valor | Δ AUC |
|---|---|---|---|
| Juiz vs BM25 | +0,167 | 4,6e-17 | +0,194 |
| Juiz vs Embeddings | +0,079 | 6,5e-6 | +0,058 |
| Juiz vs Híbrido linear | +0,096 | 2,6e-8 | +0,102 |

Os intervalos de confiança do ΔAUC excluem zero nas três comparações.

### Onde as duplicatas se perdem

- Em **317 de 500** duplicatas (63%) a original está entre as 5 candidatas (recall@5 da recuperação híbrida). Para comparação, o BM25 sozinho chega a 50,8% e os embeddings sozinhos a 63,0%.
- Das 317 recuperáveis, o juiz detecta **256 (81%)**.
- As **183** duplicatas restantes nunca chegam ao juiz. O gargalo é a recuperação.
- A probabilidade P("Yes") é bimodal, e o limiar quase não importa: variando-o de 0,01 a 0,9, a acurácia fica entre 0,708 e 0,712.

### Por local da original

| Original na... | Recall@5 | Duplicatas detectadas pelo juiz |
|---|---|---|
| Base (250 casos) | 184/250 (74%) | 172 |
| Teste (250 casos) | 133/250 (53%) | 136 |

### O juiz é prejudicado por ver só 5 candidatas?

Os baselines olham a base inteira, enquanto o juiz vê apenas as 5 candidatas. Para verificar se isso o prejudica:

- Restringindo os baselines às mesmas 5 candidatas, a melhor acurácia deles é 0,62 (BM25), 0,66 (embeddings) e 0,62 (híbrido linear), todas abaixo de 0,708.
- O top 1 correto de um baseline fora das candidatas do juiz ocorreu em 8 (BM25), 19 (embeddings) e 7 (híbrido linear) de 500 duplicatas. Em apenas 16 duplicatas (3,2%) algum baseline acertou com a original fora das candidatas.
- Em 167 duplicatas, a original ficou fora das candidatas e nenhum baseline acertou.

Os dados não mostram desvantagem relevante para o juiz.

### Efeito do tamanho da base

Comparando com uma execução anterior (base 1.000, teste 100), a acurácia do juiz foi de 0,73 para 0,708, enquanto a do BM25 caiu de 0,67 para 0,542, a dos embeddings de 0,75 para 0,628 e a do híbrido linear de 0,77 para 0,612. Essa comparação está **confundida**, porque base e teste cresceram 10× ao mesmo tempo. Para isolar o efeito da base, o BM25 foi reavaliado na mesma amostra de validação com bases de tamanhos diferentes:

| Tamanho da base | Original no top 1 (BM25) | AUC (BM25) |
|---|---|---|
| 1.040 | 51% | 0,632 |
| 2.000 | 47% | 0,612 |
| 5.000 | 38% | 0,593 |
| 10.000 | 34% | 0,577 |

Na execução com base de 1.000 e teste de 100 as diferenças entre sistemas não foram significativas (p entre 0,5 e 0,86), por causa do tamanho pequeno do teste.

### Análise de erros

- **100 falsos positivos**, 77 deles com P ≥ 0,99. Na leitura de 10 deles, parte parece duplicata não rotulada (por exemplo #28270 e #46520) e parte é erro real do juiz (por exemplo #112097 contra #84500).
- **84 duplicatas com a original errada**: em 32 a original estava entre as candidatas e em 52 ficou de fora.
- Na execução menor, vários falsos negativos eram duplicatas por causa raiz com redação diferente (por exemplo "Cached images" e "ImageFrame displays error icons too quickly").

## 8. Descrição dos arquivos de resultado

| Arquivo | Conteúdo |
|---|---|
| `metricas.json` | Configuração, tamanho da amostra, limiares, acurácia na calibração e todas as métricas dos quatro sistemas |
| `comparacao_sistemas.csv` | Tabela comparativa dos quatro sistemas |
| `matrizes_confusao.png` | Matrizes de confusão dos quatro sistemas |
| `resultados_avaliacao.csv` | Resultado por issue de teste: decisão, probabilidade e original apontada, por sistema |
| `erros_fp_fn.csv` | Falsos positivos e falsos negativos do juiz, para inspeção manual |
| `base_total.csv` | Base ao fim da simulação, com os vínculos `duplicate_of` das issues inseridas |
| `amostra_1100.csv` | Amostra de entrada (base + teste, com `split`, `is_duplicate`, `master_id`, `orig_local`) |

## 9. Limitações

**Do pipeline**

- **Teto de recuperação.** Com 5 candidatas, só 63% das duplicatas chegam ao juiz.
- **Precisão cai com o tamanho da base.** Mais issues trazem mais candidatas parecidas e mais alarmes falsos.
- **Juiz conservador.** A P("Yes") é bimodal: o juiz decide com muita certeza, certa ou errada.
- **Custo.** Cerca de 51,6 minutos para 1.000 issues em GPU gratuita.

**Da avaliação**

- **Rótulos imperfeitos.** Duplicatas por causa raiz e duplicatas não marcadas limitam qualquer método baseado apenas em texto.
- **Originais dentro do teste.** Quando a original está no teste, ela não tem issues irmãs na base, o que reduz o recall@5 (53% contra 74%). É um efeito do desenho da amostragem.
- **Calibração pequena.** Os limiares usam só 60 issues.
- **Uma semente, uma amostra.** Não há repetições, e as duas execuções comparadas variaram base e teste ao mesmo tempo.

## 10. Trabalhos futuros

- Ablação de `TOP_K` (1, 3, 5, 10, 20), medindo primeiro o recall@K, que não exige o LLM.
- Variações do prompt do juiz, mudando um elemento por vez.
- Várias sementes e calibração com mais issues.
- Fixar a base e variar só o teste (e o contrário) para separar os efeitos do tamanho.

## Trabalho relacionado

GPTrace (Herter et al., ICSE 2026), que deduplica crash reports com embeddings de LLM.

## Licença e uso dos dados

Defina aqui a licença do código e informe a origem e os termos de uso do dataset do Mozilla Core.
