# Eletrossíntese — motor de cálculo experimental

Aplicação local para registrar experimentos de eletrossíntese e substituir os
cálculos manuais repetitivos por um motor auditável: carga, quantidade de
elétrons, quantidade de matéria, reagente limitante, massa teórica, rendimento
e eficiência faradaica.

O projeto foi construído sobre um princípio: **o software não sabe química, e
não finge saber**. Ele conhece relações matemáticas universais. Coeficientes
estequiométricos, número de elétrons por mol e eficiência de corrente vêm do
seu protocolo. Quando um deles falta, o resultado correspondente aparece como
*não calculável*, com a lista do que falta — nunca como um número inventado.

---

## Sumário

- [Como executar](#como-executar)
- [O que o software faz](#o-que-o-software-faz)
- [Arquitetura](#arquitetura)
- [Estrutura de pastas](#estrutura-de-pastas)
- [Fórmulas e unidades](#fórmulas-e-unidades)
- [Cadastrar uma reação](#cadastrar-uma-reação)
- [Executar um experimento](#executar-um-experimento)
- [Interpretar os resultados](#interpretar-os-resultados)
- [Adicionar novas fórmulas](#adicionar-novas-fórmulas)
- [Testes](#testes)
- [Banco de dados](#banco-de-dados)
- [Limites do software](#limites-do-software)

---

## Como executar

Requer Python 3.11 ou superior.

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m app
```

A aplicação sobe em `http://127.0.0.1:8000` e abre o navegador
automaticamente. A documentação interativa da API fica em `/docs`.

Variáveis de ambiente reconhecidas:

| Variável | Padrão | Para quê |
|---|---|---|
| `ELAB_DATABASE_URL` | `sqlite:///data/experiments.db` | trocar de banco |
| `ELAB_HOST` | `127.0.0.1` | interface de rede |
| `ELAB_PORT` | `8000` | porta |
| `ELAB_OPEN_BROWSER` | `1` | `0` não abre o navegador |
| `ELAB_DATA_DIR` | `./data` | onde fica o banco SQLite |
| `ELAB_ECHO_SQL` | `0` | `1` imprime o SQL gerado |

---

## O que o software faz

**Cadastro** — nome, identificador, data, referência do protocolo, observações,
lista de reagentes com massa molar / massa / quantidade / concentração e
coeficiente, e o produto com massa molar, coeficiente e `z`.

**Cálculo** — carga (`Q = I·t` ou carga medida), quantidade de elétrons,
quantidade de matéria de cada reagente, reagente limitante, quantidade teórica
por duas rotas independentes, massa teórica, as duas definições de rendimento,
carga teórica, eficiência faradaica, carga por mol de substrato, elétrons por
mol de substrato e energia elétrica.

**Rastreamento** — cada resultado tem o botão *Como foi calculado?*, que abre a
derivação completa: dados → conversões → fórmula → substituição → resultado.

**Simulação** — sliders de corrente, tempo, massa, eficiência, `z` e
coeficiente, com varredura de um parâmetro e leitura em tempo real.

**Cenários** — várias condições avaliadas de uma vez, em tabela comparativa.

**Gráficos** — qualquer par de grandezas entre experimentos: rendimento × tempo,
rendimento × corrente, rendimento × carga, massa × carga, eficiência ×
rendimento, entre outros. Seleção de quais experimentos aparecem.

**Comparação** — dois ou mais experimentos lado a lado, com diferenças
absolutas e percentuais.

**Relatórios** — relatório experimental completo em PDF, e exportação em CSV,
JSON e Excel. Importação por CSV com modelo pronto.

**Dois níveis de visualização** — modo simples (massa inicial, massa obtida,
rendimento, corrente, tempo) e modo avançado (mols, carga, elétrons,
estequiometria, eficiência, fórmulas, conversões, cálculos intermediários).

---

## Arquitetura

```
   Navegador
      │  HTML + CSS + JS (sem cálculo científico)
      ▼
   API HTTP  ........................  app/backend/api
      │  traduz HTTP ↔ objetos de domínio
      ▼
   Serviços  ........................  app/backend/services
      │  orquestra cálculo + persistência + relatórios
      ├──────────────► Repositório ► SQLAlchemy ► SQLite/PostgreSQL
      ▼
   Motor de cálculo  ................  app/backend/calculations
      units → formulas → stoichiometry / electrochemistry → engine
```

Três regras estruturais, verificadas pelos testes:

1. **O motor de cálculo não importa nada de I/O.** Nem FastAPI, nem
   SQLAlchemy, nem Jinja. Pode ser usado num script ou num notebook.
2. **Nenhum cálculo científico acontece no navegador.** O JavaScript formata e
   desenha; todo número vem do Python.
3. **O rastro é produzido junto com o número**, pela mesma função. A interface
   não reconstrói derivações, então rastro e valor não podem divergir.

As três são verificadas por `tests/test_architecture.py`, que lê a árvore de
importações do motor e recusa qualquer dependência de framework web, banco ou
I/O — e varre o JavaScript atrás de constantes físicas e conversões de unidade
reimplementadas. Se alguém quebrar uma fronteira, a suíte falha no mesmo dia.

### Duas camadas de validação

O motor para no primeiro erro: ele não deve calcular com dado inválido. Quem
está preenchendo um formulário, porém, não deve descobrir um problema por vez.

| Camada | Onde | Comportamento |
|---|---|---|
| Guardas de domínio | `calculations/rules.py`, chamadas por `spec.py` | levantam na primeira violação |
| Validação de fronteira | `validators/experiment_validator.py` | percorre tudo e acumula |

`POST /api/validate` devolve a lista completa e **nunca** responde 422 — ele
existe para relatar problemas, não para recusá-los. Quando `/calculate` ou
`/experiments` falham por validação, a resposta 422 carrega a mesma lista no
campo `detail.problems`, e a interface a exibe inteira. A validação de
fronteira também detecta nomes de unidade inválidos, que o motor só descobriria
no meio de um cálculo, e distingue `error` (impede o cálculo) de `notice`
(apenas avisa, como concentração informada sem volume).

### Camadas do motor

| Módulo | Responsabilidade |
|---|---|
| `units.py` | registro de unidades, conversões exatas, tipo `Quantity` |
| `errors.py` | exceções de domínio |
| `trace.py` | passos do rastro e o tipo `Outcome` (calculado / não calculável) |
| `formatting.py` | arredondamento **apenas** para apresentação |
| `formulas.py` | catálogo simbólico (SymPy) com dimensão de cada símbolo |
| `spec.py` | documento de entrada do experimento e suas validações |
| `stoichiometry.py` | quantidade de matéria e reagente limitante |
| `electrochemistry.py` | carga, elétrons, eficiência, energia |
| `rules.py` | guardas numéricas reutilizáveis, com mensagem única por regra |
| `engine.py` | orquestra tudo e monta o conjunto de resultados |
| `simulation.py` | varreduras (NumPy), cenários e comparação |

---

## Estrutura de pastas

```
.
├── app/
│   ├── __main__.py            python -m app
│   ├── main.py                aplicação FastAPI
│   ├── config.py              configuração por variável de ambiente
│   ├── backend/
│   │   ├── api/               rotas e esquemas Pydantic
│   │   ├── calculations/      motor científico (sem I/O)
│   │   ├── database/          sessão e repositório
│   │   ├── models/            modelos SQLAlchemy
│   │   ├── services/          experimentos, relatórios, import/export
│   │   └── validators/        validação de fronteira (todos os erros de uma vez)
│   └── frontend/
│       ├── templates/         index.html
│       └── static/            css, js (módulos ES) e Chart.js embarcado
├── tests/                     191 testes
├── docs/calculations.md       todas as fórmulas, unidades e regras
├── data/                      banco SQLite (criado na primeira execução)
├── requirements.txt
└── pyproject.toml
```

Duas diferenças em relação à estrutura sugerida no pedido, ambas deliberadas:

- **`tests/` e `docs/` ficam na raiz, não dentro de `app/`.** Testes dentro do
  pacote seriam distribuídos junto com ele e importariam o código como
  submódulo, o que esconde erros de importação.
- **`data/` fica na raiz e é configurável.** Dados de execução não pertencem à
  árvore de código; misturá-los quebra qualquer empacotamento futuro.

---

## Fórmulas e unidades

A referência completa está em [`docs/calculations.md`](docs/calculations.md).
A versão viva está em `GET /api/formulas` e na tela **Referência → Fórmulas**.

Relações principais:

```
Q    = I · t                        carga elétrica
n_e  = Q / F                        quantidade de elétrons
n    = m / M                        quantidade de matéria
m    = n · M                        massa
n    = c · V                        quantidade a partir de solução
n_p  = n_L · (ν_p / ν_L)            rota estequiométrica
n_p  = (Q · η) / (z · F)            rota faradaica
Q_th = n_p · z · F                  carga teórica
η    = 100 · (n_exp · z · F) / Q    eficiência faradaica
Y_m  = 100 · m_exp / m_teórica      rendimento base massa
Y_n  = 100 · n_exp / n_teórica      rendimento base quantidade de matéria
```

Conversões automáticas, sempre visíveis no rastro:
`mA → A`, `µA → A`, `min → s`, `h → s`, `mg → g`, `µg → g`, `kg → g`,
`µmol → mmol → mol`, `mL → L`, `mAh → C`, `% → fração`.

`F = 96485,33212... C/mol` (CODATA 2018). Pode ser substituída por
experimento; o valor usado aparece em todo rastro que depende dele.

---

## Cadastrar uma reação

Não há reação embutida no código. Cadastrar uma reação é preencher seus
parâmetros a partir da sua equação balanceada.

1. **Experimento → aba Reagentes.** Um card por reagente. Para cada um informe
   massa molar e coeficiente estequiométrico, e a quantidade por uma destas
   vias: massa, quantidade de matéria, ou concentração + volume.
2. Escolha o **substrato de referência** (usado nas métricas por mol de
   substrato) e deixe o **limitante** em automático, salvo se você tiver
   motivo experimental para fixá-lo.
3. **Aba Produto e cálculo.** Massa molar, coeficiente estequiométrico e `z`
   (elétrons por mol de produto). O `z` depende do mecanismo: pegue-o do seu
   protocolo ou da literatura. Sem ele, o software simplesmente não calcula as
   métricas faradaicas — e diz isso.
4. Escolha a **definição de rendimento** e a **base da quantidade teórica**.

Para reaproveitar a configuração numa série de experimentos, salve e use
**Duplicar**.

---

## Executar um experimento

1. **Aba Eletroquímica.** Corrente e unidade, tempo e unidade. Opcionalmente
   tensão, carga medida por coulometria (que tem precedência sobre `I·t`) e
   eficiência de corrente admitida.
2. **Aba Produto.** Massa isolada — ou quantidade isolada, se você a mediu
   diretamente.
3. **Calcular.** Os resultados aparecem à direita.
4. **Salvar experimento.** Ele passa a aparecer no painel, nos gráficos, nas
   comparações e nas exportações.

Importação em lote: **Dados → Importar de CSV**. Baixe o modelo, preencha uma
linha por experimento e envie. Células vazias viram "não informado", nunca
zero. Linhas com problema são reportadas individualmente sem interromper a
importação das demais.

---

## Interpretar os resultados

A cor da borda esquerda de cada resultado codifica sua natureza:

| Marca | Significado |
|---|---|
| barra sólida teal | **valor calculado** por uma fórmula |
| barra âmbar | **medida experimental** que você informou |
| barra cinza | **não calculável** — com a lista do que falta |

Um valor calculado nunca é apresentado como resultado experimental, e
vice-versa. O mesmo vale no relatório em PDF, que separa as seções
"Medidas experimentais", "Resultados calculados" e "Parâmetros não
calculáveis".

Clique em **Como foi calculado?** em qualquer resultado para ver a derivação
completa. O modo avançado mostra ainda a tabela do reagente limitante com
`n`, `ν` e `ξ = n/ν` de cada reagente.

Os avisos em vermelho comparam apenas números já calculados. Eles apontam
inconsistências aritméticas — rendimento acima de 100 %, eficiência
impossível, carga insuficiente ou em excesso — e não fazem previsões químicas.

---

## Adicionar novas fórmulas

```python
# app/backend/calculations/formulas.py
MINHA_FORMULA = register(Formula(
    id="minha_formula",
    name="Nome legível",
    lhs="y", rhs="a*b/c",
    result_dimension="mass",
    symbols=(
        SymbolSpec("a", "amount", "descrição de a"),
        SymbolSpec("b", "molar_mass", "descrição de b"),
        SymbolSpec("c", "ratio", "descrição de c"),
    ),
    description="Quando esta relação vale.",
    reference="De onde ela vem.",
    nonzero=("c",),          # símbolos em denominador
))
```

Depois escreva um teste com um valor calculado à mão em
`tests/test_formulas.py`. Se a fórmula deve entrar no fluxo do experimento,
chame-a de `electrochemistry.py` ou `stoichiometry.py` devolvendo um
`Outcome`, e registre-a em `engine.run`. O catálogo da API, a tela de
referência e a documentação são gerados a partir do registro — não é preciso
tocar em mais nada.

---

## Testes

```bash
pytest                    # suíte completa
pytest tests/test_engine.py -v
pytest -k "yield or limiting"
```

Cobertura por área:

| Arquivo | O que verifica |
|---|---|
| `test_units.py` | conversões, ambiguidade, dimensões incompatíveis, precisão |
| `test_formulas.py` | cada fórmula contra valores calculados à mão, guardas de divisão por zero, símbolos errados |
| `test_stoichiometry.py` | quantidade de matéria por três vias, limitante, exclusões, limitante forçado |
| `test_engine.py` | caso de referência completo, dados ausentes, avisos, validações, precisão da cadeia |
| `test_simulation.py` | varreduras, cenários, comparação, pontos inválidos |
| `test_api.py` | CRUD, busca, painel, exportações, PDF, importação de CSV, integridade da persistência |
| `test_architecture.py` | fronteiras entre camadas e validação de fronteira |

O **caso de referência** usado como âncora (documentado no cabeçalho de
`tests/conftest.py`) foi calculado à mão:

```
Substrato : M = 100 g/mol, m = 100 mg     →  n     = 1,000e-3 mol
Produto   : M = 114 g/mol, ν = 1          →  n_th  = 1,000e-3 mol
                                             m_th  = 114 mg
Isolado   : 85,5 mg                       →  Y     = 75,000 %
Elétrica  : 50 mA × 60 min                →  Q     = 180 C
F = 96485 C/mol, z = 2                    →  n(e⁻) = 1,8655749e-3 mol
                                             Q_th  = 192,970 C
                                             η     = 80,404167 %
```

---

## Banco de dados

SQLite por padrão, em `data/experiments.db`. Os modelos foram escritos para
migrar sem reescrita:

- só tipos presentes nos dois bancos (`JSON`, `String`, `Float`, `DateTime`);
- datas em UTC com fuso explícito;
- nenhuma dependência de `rowid` ou `AUTOINCREMENT` do SQLite;
- o documento de entrada é guardado como JSON, e as grandezas usadas em busca,
  ordenação e gráficos são espelhadas em colunas escalares indexadas — sempre
  reescritas pelo motor a cada gravação, nunca editadas à mão.

Para PostgreSQL:

```bash
pip install "psycopg[binary]"
ELAB_DATABASE_URL="postgresql+psycopg://usuario:senha@host/banco" python -m app
```

Para evolução de esquema em produção, o passo seguinte é Alembic; os modelos
já estão compatíveis.

---

## Limites do software

Isto é uma ferramenta de **cálculo e registro**. Ela não:

- sugere condições experimentais como se fossem validadas;
- afirma que uma condição produzirá determinado resultado real;
- infere mecanismo, número de elétrons ou estequiometria;
- preenche valores desconhecidos com estimativas.

Os resultados da simulação e dos cenários são projeções aritméticas das
fórmulas com os parâmetros informados, e carregam essa ressalva na tela e na
resposta da API. A validação experimental continua sendo do laboratório.
