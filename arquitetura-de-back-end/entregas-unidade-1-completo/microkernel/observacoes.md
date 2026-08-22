# Observações — Experimento 3: Microkernel

## Sistema utilizado

Sistema de Faturamento Multi-Estado, organizado segundo o estilo Microkernel:

- `nucleo.py` → **Core System** (`CoreFaturamento`) + **Plugin Registry**
  (`PluginRegistry`) + **Contrato** (`PluginFaturamento`, um `Protocol`)
- `dominio.py` → objetos de dados (`Fatura`, `Cliente`, `ItemFatura`,
  `ResultadoEmissao`)
- `plugins/` → plugins independentes, cada um registrado em uma categoria
  (`impostos`, `frete`, `notificacao`):
  - `impostos_sp.py` → `ImpostoSPPlugin` (ICMS-SP), `ISSSPPlugin` (ISS-SP)
  - `impostos_rj.py` → `ImpostoRJPlugin` (ICMS-RJ)
  - `frete.py` → `FreteCorrespondenciaPlugin`
  - `notificacao.py` → `NotificacaoEmailPlugin`
- `main.py` → registra os plugins no núcleo, emite 4 faturas de exemplo (SP,
  RJ, SP com frete grátis, e MG — este último com um plugin criado e
  registrado em tempo de execução, sem tocar no núcleo)

Execução: `python3 main.py`

---

## 1. O que eu alterei?

Alterei **um plugin de imposto**, `ImpostoRJPlugin` (`plugins/impostos_rj.py`),
mudando sua alíquota de ICMS:

```python
# ANTES
class ImpostoRJPlugin:
    nome = "ICMS-RJ"
    ALIQUOTA = 0.20

# DEPOIS
class ImpostoRJPlugin:
    nome = "ICMS-RJ"
    ALIQUOTA = 0.25  # aumentada de 20% para 25%
```

Não toquei no núcleo (`nucleo.py`), no contrato `PluginFaturamento`, na ordem
de execução das categorias (`ORDEM_CATEGORIAS`), nem em nenhum outro plugin
(`ImpostoSPPlugin`, `ISSSPPlugin`, `FreteCorrespondenciaPlugin`,
`NotificacaoEmailPlugin`).

## 2. O que mudou na saída?

`diff saida-antes.txt saida-depois.txt` mostra que a única diferença entre as
duas execuções está na **Fatura #1002 (Rio de Janeiro)**:

```diff
<   [Email → nf@distribrj.com] Fatura #1002 emitida para Distribuidora Rio. Total: R$10,080.00
---
>   [Email → nf@distribrj.com] Fatura #1002 emitida para Distribuidora Rio. Total: R$10,500.00

<     ICMS-RJ: R$1,680.00
---
>     ICMS-RJ: R$2,100.00

<     TOTAL: R$10,080.00
---
>     TOTAL: R$10,500.00
```

O valor bruto da fatura RJ é R$8.400,00. Com a alíquota antiga (20%), o
ICMS-RJ era R$1.680,00 e o total R$10.080,00. Com a alíquota nova (25%), o
ICMS-RJ passou a R$2.100,00 e o total a R$10.500,00.

Todas as demais faturas — #1001 (São Paulo), #1003 (São Paulo, frete grátis)
e #1004 (Minas Gerais, plugin novo registrado em tempo de execução) —
permaneceram **byte a byte idênticas** nas duas execuções, incluindo os
valores de ICMS-SP, ISS-SP, ICMS-MG, frete e notificações.

## 3. Qual responsabilidade arquitetural isso demonstra?

A alteração foi feita em **um único plugin**, dono exclusivo da regra fiscal
do estado do Rio de Janeiro, e o efeito ficou **inteiramente contido** nas
faturas daquele estado:

- O **núcleo** (`CoreFaturamento`) não sabe o que é "ICMS" nem em que estado
  ele se aplica — ele só sabe que existe uma lista de plugins na categoria
  `"impostos"` e que deve chamar `processar(fatura, resultado)` em cada um,
  na ordem `impostos → frete → notificacao`. Por isso não precisou mudar.
- O **contrato** (`PluginFaturamento`, um `Protocol`) continua sendo
  respeitado sem nenhuma alteração — ele define apenas a assinatura
  `processar(fatura, resultado) -> resultado`, e não conhece regras internas.
- Os **plugins de SP** (`ImpostoSPPlugin`, `ISSSPPlugin`) e o **plugin de
  MG** (criado dinamicamente em `main.py`) verificam
  `fatura.cliente.estado != "SP"` (ou `"MG"`) e retornam o resultado
  inalterado quando não é o seu estado — por isso a fatura RJ nunca passou
  por eles de forma "ativa", e eles não têm nenhuma dependência da alíquota
  do RJ.
- O **plugin de frete** e o **plugin de notificação** dependem apenas do
  `valor_bruto` e do `resultado.valor_total` já calculado — eles não sabem
  como os impostos foram calculados, apenas consomem o resultado agregado.

Isso evidencia a responsabilidade central do estilo Microkernel: o **núcleo
permanece estável e genérico**, cada **regra fiscal fica isolada em seu
próprio plugin**, e a extensão ou modificação de uma regra de um estado
específico **não exige alterar o núcleo nem os plugins de outros estados**
— exatamente o mesmo princípio que o próprio `main.py` já demonstra ao
adicionar suporte a MG em tempo de execução sem tocar em `CoreFaturamento`.
A ordem das categorias (`impostos → frete → notificacao`) também não foi
afetada: como o `FreteCorrespondenciaPlugin` e o `NotificacaoEmailPlugin`
leem o resultado já pronto (via `resultado.valor_bruto` e
`resultado.valor_total`), a mudança de alíquota de um imposto se propaga
automaticamente para o frete e a notificação daquela fatura, sem que esses
plugins precisem saber por quê o valor mudou.
