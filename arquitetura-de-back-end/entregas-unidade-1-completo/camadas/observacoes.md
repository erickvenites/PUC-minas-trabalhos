# Observações — Experimento 1: Camadas (Layers)

## Sistema utilizado

Sistema de Agendamento de Clínica Médica, organizado em 4 arquivos que representam
as camadas clássicas do estilo:

- `apresentacao.py` → **Camada de Apresentação** (`AgendaController`, simula endpoints HTTP)
- `servicos.py` → **Camada de Negócios** (`AgendamentoServico`, `RelatorioServico`, regras)
- `dominio.py` → **Camada de Domínio** (entidades, value objects, contratos/ABC de repositório)
- `repositorios.py` → **Camada de Dados** (implementações em memória dos repositórios)

Execução: `python3 main.py`

---

## 1. O que eu alterei?

Alterei o método `Horario.conflita_com()`, na **camada de Domínio** (`dominio.py`):

```python
# ANTES
def conflita_com(self, outro: Horario) -> bool:
    return self.inicio < outro.fim and self.fim > outro.inicio

# DEPOIS
def conflita_com(self, outro: Horario) -> bool:
    return self.inicio <= outro.fim and self.fim >= outro.inicio
```

Antes da mudança, dois horários "encostados" — um começando exatamente no
instante em que o outro termina (ex.: uma consulta das 14:00–14:20 e outra das
14:20–14:40) — **não** eram considerados conflito, pois a comparação usava
desigualdades estritas (`<` / `>`). Depois da mudança, passaram a usar `<=` / `>=`,
então esse caso "encostado" passa a ser tratado como conflito (regra de negócio
que passa a exigir uma folga mínima entre consultas do mesmo médico).

Também adicionei, em `main.py`, um novo cenário de demonstração (item 10) que
tenta agendar uma consulta para o Dr. João começando exatamente às 14:20 —
horário em que sua consulta anterior termina — apenas para expor o efeito da
alteração na saída. Nenhuma outra regra de negócio foi tocada.

## 2. O que mudou na saída?

Comparando `saida-antes.txt` e `saida-depois.txt`:

- Os itens 1 a 9 permanecem **idênticos** nas duas execuções — a alteração não
  afeta nenhum dos cenários originais, porque nenhum deles testava horários
  exatamente encostados.
- O novo item 10 (presente só em `saida-depois.txt`, já que o cenário foi
  adicionado junto com a alteração) mostra o efeito da mudança:

  ```
  10. [EXPERIMENTO] Agendamento 'encostado' (começa exatamente onde outra termina)
  HTTP 409 CONFLICT → {'erro': 'Dr(a). Dr. João Costa já tem consulta das 14:00 às 14:20.'}
  ```

  Com o código **original** (`inicio < outro.fim`), essa mesma tentativa teria
  retornado `HTTP 201 CREATED`, pois `14:20 < 14:20` é falso e o método
  `conflita_com` retornaria `False`. Com a regra alterada (`inicio <= outro.fim`),
  `14:20 <= 14:20` é verdadeiro, o conflito é detectado e o agendamento é
  rejeitado com `HTTP 409`.

## 3. Qual responsabilidade arquitetural isso demonstra?

A alteração foi feita em **uma única linha**, dentro do Value Object `Horario`,
que pertence à **camada de Domínio**. Mesmo assim, o efeito se propagou até a
**camada de Apresentação** (o código HTTP retornado por `AgendaController`
mudou de 201 para 409), sem que **nenhuma linha** de `servicos.py`,
`repositorios.py` ou `apresentacao.py` precisasse ser alterada.

Isso evidencia exatamente a responsabilidade de cada camada no estilo Camadas:

- A **regra de conflito de horário em si** (o que conta como sobreposição)
  é uma responsabilidade do **Domínio**, não da Apresentação nem dos
  Repositórios.
- A **camada de Negócios** (`AgendamentoServico.agendar`) apenas *usa* o
  contrato `horario.conflita_com(...)` — ela não sabe (e não precisa saber)
  como a comparação é feita internamente. Por isso não precisou mudar.
- A **camada de Apresentação** (`AgendaController.post_consulta`) apenas
  traduz a exceção `ConflitodeAgendaError` em um `HTTP 409` — ela reage ao
  resultado da regra, mas não a implementa. Por isso também não precisou
  mudar.
- A **camada de Dados** (repositórios em memória) apenas guarda e recupera
  consultas; ela é completamente indiferente à definição de "conflito".

Ou seja, o experimento demonstra na prática o princípio de **separação de
responsabilidades por camada**: uma mudança de regra de negócio pontual fica
isolada na camada correta (Domínio), e as demais camadas continuam
funcionando sem alteração — o que também é uma evidência de baixo
acoplamento entre camadas e de que o sistema respeita o Open/Closed Principle
mencionado no próprio `main.py` (item 9), só que aplicado aqui à regra de
domínio em vez de à troca de repositório.
