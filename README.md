# EV-Challenge-Energias-sustentaveis-02

# GreenCharge 

**EV Challenge 2026 — Energias Renováveis e Sustentabilidade — FIAP**

---

## Equipe

| Nome             | RM       |
| ---------------- | -------- |
| Bryan Lugli      | RM571350 |
| Beckman Lugli    | RM573442 |
| Guilherme Xavier | RM573053 |

---

## O que é o GreenCharge

A ideia surgiu de um problema simples: eletropostos que ficam ligados consumindo energia da rede mesmo quando tem sol batendo nos painéis solares do lado. Isso acontece porque a maioria dos postos não tem nenhum software que conecte a geração solar com a decisão de quando e como recarregar os veículos.

O GreenCharge é esse software. Ele lê a geração solar disponível em tempo real, vê quantos veículos estão conectados e decide quanto usar de solar e quanto puxar da rede. Simples assim no conceito, mas com um impacto real na conta de luz e no CO₂ emitido.

---

## Sprint 2 — O que fizemos

Nessa sprint a ideia saiu do papel. Construímos uma simulação funcional em Python que roda o núcleo do sistema: o motor de decisão que prioriza energia solar na recarga dos veículos.

Como não temos acesso às credenciais reais da API da GoodWe ainda, simulamos a geração solar usando o Clear Sky Model — um modelo físico real, baseado na posição do sol calculada pela geometria astronômica. Não é um número aleatório: é a mesma equação usada em engenharia solar para estimar produção de painéis.

A simulação rodou 30 dias inteiros, hora por hora, com padrões reais de demanda (mais carros carregando de manhã cedo e no fim da tarde, madrugada quase vazia). No total, 720 registros gerados.

O que está funcionando:

| O que | Como |
|---|---|
| Modelo de geração solar | Clear Sky Model calibrado para São Paulo (-23,5°) |
| Lógica de prioridade solar | Solar primeiro, rede só no que faltar |
| Simulação de 30 dias | 720 registros hora a hora |
| Cálculo de CO₂ evitado | Fator de emissão do SIN (MCTI 2023) |
| Cálculo de economia | Tarifa ANEEL referência SP |
| Exportação dos dados | CSV com histórico completo + JSON com relatório |

---

## Como o sistema funciona

### A arquitetura

```
  ENTRADA                     PROCESSAMENTO                SAÍDA
  ───────                     ─────────────                ─────

  API GoodWe ──────────────▶  Motor de Decisão  ────────▶  Dashboard Web
  (produção)    geração kW    ─────────────────            (Node.js)
                              1. Lê geração solar
  Modelo Clear Sky ────────▶  2. Verifica EVs conectados
  (Sprint 2 / POC)            3. Decide fonte de energia   Relatório
                              4. Registra tudo  ────────▶  JSON / CSV

  Sensores EVSE ───────────▶  Controlador OCPP  ────────▶  Alertas
  (veículos conectados)
```

Em produção, a entrada de dados vem direto da API SEMS da GoodWe. Na POC, substituímos essa leitura pelo modelo matemático, mas a lógica de decisão é exatamente a mesma que rodaria no sistema real.

### A lógica de decisão (o coração do sistema)

A cada hora, o sistema faz uma pergunta simples:

```
Tem veículo conectado?
│
Não → não faz nada, sem consumo
│
Sim → Calcula demanda total (N carregadores × 7,4 kW)
      │
      Solar ≥ Demanda? ──Sim──▶ Usa 100% solar, rede = zero
      │
      Não ──────────────────▶ Usa tudo que tem de solar
                               + complementa com a rede
                               │
                               Registra: kWh solar, kWh rede,
                               CO₂ evitado, economia em R$
```

### Parâmetros usados na simulação

| Parâmetro | Valor | De onde veio |
|---|---|---|
| Capacidade do painel | 5 kWp | Referência GoodWe para instalações pequenas |
| Potência por carregador | 7,4 kW | AC monofásico, padrão no Brasil |
| Eficiência do painel | 18% | Monocristalino típico do mercado |
| Fator de emissão CO₂ | 0,0897 kg/kWh | MCTI — Inventário Nacional GEE 2023 |
| Tarifa de energia | R$ 0,85/kWh | ANEEL — referência SP 2024 |
| Latitude | -23,5° | São Paulo |

---

## Resultados da simulação

```
  RELATÓRIO — MAIO 2026 (30 dias)
  ════════════════════════════════════════════
  Energia consumida no total:   6.127,20 kWh
  Veio de energia solar:          561,84 kWh
  Veio da rede elétrica:        5.565,36 kWh
  Aproveitamento solar:               9,2%
  ────────────────────────────────────────────
  CO₂ evitado no mês:              50,40 kg
  Projeção para 12 meses:           0,60 t
  ────────────────────────────────────────────
  Economia na conta de luz:      R$ 477,56
  ════════════════════════════════════════════
```

**Sobre o percentual de 9,2%:** vale explicar porque esse número parece baixo. Com 4 carregadores de 7,4 kW cada, a demanda de pico é de 29,6 kW. Um painel de 5 kWp consegue entregar no máximo 5 kW. Ou seja, o painel não é grande o suficiente para atender toda a demanda — e isso é realista. Se aumentar o painel para 12 kWp, o aproveitamento solar chega perto dos 40% que estimamos na Sprint 1. O código aceita qualquer valor, é só alterar o parâmetro `CAPACIDADE_PAINEL_KWP`.

### Amostra dos dados gerados

| data | hora | veiculos | solar (kW) | kWh solar | kWh rede | CO₂ evitado (kg) | economia (R$) |
|---|---|---|---|---|---|---|---|
| 2026-05-01 | 08h | 3 | 2,841 | 2,841 | 19,359 | 0,2549 | 2,41 |
| 2026-05-01 | 12h | 4 | 4,127 | 4,127 | 25,473 | 0,3702 | 3,51 |
| 2026-05-01 | 14h | 2 | 3,654 | 3,654 | 11,146 | 0,3278 | 3,11 |
| 2026-05-01 | 18h | 4 | 1,203 | 1,203 | 28,397 | 0,1079 | 1,02 |

---

## Por que escolhemos cada tecnologia

**Python** foi a escolha óbvia pro núcleo do sistema. É a linguagem padrão em projetos de automação e IoT, e o código roda em qualquer lugar — computador, servidor ou Raspberry Pi ligado ao inversor. Não precisamos instalar nada além do Python em si.

**Clear Sky Model (Bird, 1984)** é um modelo físico consagrado em engenharia solar. Em vez de inventar uma curva de geração, usamos a geometria astronômica real: declinação do sol, ângulo horário, altitude solar. Isso garante que a simulação respeita os limites físicos — o sol não gera energia à meia-noite, e gera mais ao meio-dia. Adicionamos uma variação aleatória de ±20% pra simular dias nublados.

**CSV e JSON** para exportar os dados porque qualquer pessoa consegue abrir e auditar sem precisar de software específico. Em produção, o banco seria PostgreSQL, mas para a POC não faz sentido adicionar essa dependência.

**OCPP** como protocolo de comunicação com os carregadores porque é o padrão internacional. Qualquer eletroposto do mercado brasileiro já fala OCPP — então o GreenCharge não precisa de hardware proprietário pra funcionar.

---

## Conexão com energias renováveis e sustentabilidade

O sistema aplica três conceitos que vimos durante o semestre:

**Autoconsumo fotovoltaico:** o GreenCharge maximiza o uso da energia gerada no próprio local antes de recorrer à rede. Isso reduz tanto o custo quanto a dependência de geração centralizada, que no Brasil ainda mistura hidrelétricas com termelétricas a gás e carvão.

**Demand response:** a carga é ajustada à oferta de energia limpa disponível. Quando tem sol, o sistema carrega mais. Quando não tem, pode adiar ou reduzir a carga. É um dos princípios das smart grids.

**Rastreabilidade ambiental:** usando o Fator de Emissão do SIN (publicado pelo MCTI), conseguimos transformar kWh solar em kg de CO₂ evitado. Isso não é enfeite — é o que empresas precisam para relatório ESG e o que o governo vai exigir cada vez mais.

ODS atendidos:

| ODS | Como o GreenCharge contribui |
|---|---|
| ODS 7 — Energia limpa e acessível | Prioriza energia solar nos eletropostos |
| ODS 9 — Indústria e inovação | Software que melhora infraestrutura existente sem novo hardware |
| ODS 11 — Cidades sustentáveis | Mobilidade elétrica mais eficiente e limpa |
| ODS 13 — Ação climática | CO₂ evitado mensurável e auditável |

---


