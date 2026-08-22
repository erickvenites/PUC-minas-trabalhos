# Observações — Experimento 2: Pipes and Filters

## Sistema utilizado

Pipeline de Triagem de Currículos para uma vaga de Engenheiro Backend,
organizado como uma cadeia de filtros independentes:

```
LeitorDeCurriculos (Producer)
    → ValidadorDeCurriculo (Tester)
    → NormalizadorDeCampos (Transformer)
    → FiltroPorExperienciaMinima (Tester)
    → FiltroPorPretensaoSalarial (Tester)
    → CalculadorDeScore (Transformer)
    → RelatorioDeTriagem (Consumer)
```

- `framework.py` → abstração `Filtro` (ABC) e `Pipeline` (encadeamento)
- `dominio.py` → objetos de dados (`Curriculo`, `Vaga`, `ResultadoTriagem`)
- `filtros/producer.py`, `filtros/testers.py`, `filtros/transformers.py`,
  `filtros/consumer.py` → os filtros concretos
- `main.py` → composição do pipeline e dados de entrada (6 currículos brutos)

Execução: `python3 main.py`

---

## 1. O que eu alterei?

Alterei um **critério de aprovação** usado pelo filtro `FiltroPorPretensaoSalarial`,
por meio do dado de entrada da vaga em `main.py`:

```python
# ANTES
vaga = Vaga(
    ...,
    salario_maximo=18_000.0,
)

# DEPOIS
vaga = Vaga(
    ...,
    salario_maximo=15_000.0,  # orçamento reduzido de 18.000 para 15.000
)
```

Não toquei em nenhum filtro, na ordem do pipeline, no `framework.py` nem nos
currículos de entrada — apenas no valor de `salario_maximo` que é passado ao
`Tester` responsável por esse critério.

## 2. O que mudou na saída?

Comparando `saida-antes.txt` e `saida-depois.txt`:

| | Antes (orçamento R$18.000) | Depois (orçamento R$15.000) |
|---|---|---|
| Currículo id=3 | `[DESCARTADO]` nome ausente | `[DESCARTADO]` nome ausente (sem mudança) |
| Bruno Rocha | `[REPROVADO]` 1 ano < mínimo 3 | `[REPROVADO]` 1 ano < mínimo 3 (sem mudança) |
| Clara Mendes | `[REPROVADO]` pretensão R$22.000 > R$18.000 | `[REPROVADO]` pretensão R$22.000 > R$15.000 (sem mudança) |
| **Diego Faria** | **Aprovado** (score 75%) | **`[REPROVADO]` pretensão R$16.000 > R$15.000** |
| Total aprovados | **3** (Ana, Elena, Diego) | **2** (Ana, Elena) |

A única diferença real entre as duas execuções é que Diego Faria, cuja
pretensão salarial (R$16.000) estava dentro do orçamento antigo mas fora do
novo, passou de aprovado para reprovado. Nenhum outro candidato, nenhuma
mensagem de descarte/reprovação anterior e nenhuma etapa do pipeline mudaram
de comportamento.

## 3. Qual responsabilidade arquitetural isso demonstra?

A mudança foi feita em **um único parâmetro de entrada** (`salario_maximo`),
consumido por **um único filtro** (`FiltroPorPretensaoSalarial`), e o efeito
ficou **inteiramente contido** nesse ponto do pipeline:

- O **Producer** (`LeitorDeCurriculos`) continuou gerando os mesmos 6
  objetos `Curriculo` — ele não sabe nada sobre critérios de aprovação.
- O **Tester** de validação de dados (`ValidadorDeCurriculo`) e o
  **Transformer** de normalização (`NormalizadorDeCampos`) continuaram
  produzindo exatamente a mesma saída — eles não têm nenhuma dependência do
  orçamento da vaga.
- O **Tester** de experiência mínima (`FiltroPorExperienciaMinima`) também
  não foi afetado, pois seu critério é outro (anos de experiência), e
  continuou reprovando somente Bruno Rocha.
- Apenas o **Tester de pretensão salarial** reagiu à mudança, porque é ele
  quem detém essa regra de negócio específica.
- O **Transformer** de score (`CalculadorDeScore`) e o **Consumer**
  (`RelatorioDeTriagem`) simplesmente processaram uma lista de candidatos com
  um elemento a menos — nenhum deles precisou ser alterado para refletir o
  novo orçamento.

Isso evidencia a responsabilidade central do estilo Pipes and Filters: **cada
filtro concentra uma única regra e é independente dos demais** (sem estado
compartilhado, sem conhecimento do que vem antes ou depois). Uma mudança de
critério de negócio fica isolada no filtro (`Tester`) dono daquela regra, e o
restante do pipeline — Producer, outros Testers, Transformers e Consumer —
continua funcionando sem qualquer alteração. Isso também mostra que seria
possível reordenar, remover ou adicionar filtros sem que os demais precisem
ser modificados, já que a única "comunicação" entre eles é a lista de dados
que passa de um `processar()` para o próximo.
