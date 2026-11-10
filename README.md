# Controle Revisional Volvo — Site interno + App GT Minas Motorista

**Solicitante:** Edson Santos (Indicadores) · **Data:** 10/11/2026 · **Tipo:** Melhoria · API / integrações · **Onde:** Site e App

Protótipo navegável: [`index.html`](index.html) (abas "Site interno" e "App do motorista").

## Objetivo

Tirar o Controle Revisional da planilha manual e alimentar **automaticamente, 1x por dia, com os dados do Volvo Connect**. Também queremos mostrar o **histórico de revisões Volvo** de cada placa: detalhado no site e resumido no app do motorista.

## Como é hoje

- O Edson sobe todo dia uma planilha Excel na aba Controle Revisional do site.
- O app mostra um único item (ex.: "73 – Revisão troca de filtro e óleo 40.000 km") com KM atual, KM e data da última revisão, próxima revisão e "Vencido".
- Não há histórico: não se sabe quantas vezes a placa foi à oficina, o que foi trocado, quantos dias ficou parada nem quem era o motorista.
- Hoje há campos confusos na tela do motorista (ex.: "KM vencidos 1.225" ao lado de "Faltam 0 km").

## Como vai ser

### Regras de negócio
1. **Só conta revisão feita em concessionária Volvo.** Revisão por fora ou de terceiros não entra.
2. **Nada manual para o motorista.** O app só exibe; não há botão de atualizar nem confirmação.
3. **Sem valores.** Não exibir custos, preços nem totais.
4. **Atualização automática 1x por dia** (sugestão: madrugada) + data/hora da última sincronização visível.

### Site interno (detalhado)
- Cards: revisões vencidas · vencendo (≤ 2.000 km ou ≤ 15 dias) · em dia · veículos com luz de alerta Volvo · média de dias parados na oficina.
- Tabela por placa: situação, KM atual, última revisão Volvo (data + KM), próxima revisão, barra de progresso e nº de revisões em 12 meses. Filtros por placa, situação, período, oficina e sistema.
- Ao clicar na placa, aparece a **linha do tempo de passagens na Volvo**:
  - data de entrada, data de saída e **dias parados**;
  - ordens de serviço (OS), KM na entrada, oficina e cidade;
  - **motorista vinculado à placa na data**;
  - se trocou peça ou não, com a lista de peças/operações (código, descrição, quantidade) agrupada por sistema (câmbio, motor, freio, combustível…).
- Indicadores de frota: placas que mais vão à oficina, sistemas com mais manutenção e tempo médio parado.

### App GT Minas Motorista (essencial)
- Destaque grande: **"Faltam X km" ou "Vencida há X km"** para a próxima revisão Volvo, com barra de progresso e data recomendada.
- Luzes de alerta Volvo ativas no caminhão (ex.: freio, óleo do motor).
- Nº de revisões Volvo do caminhão (12 meses) e data/oficina da última visita.
- "O que foi trocado na última revisão" em linguagem simples, por sistema.
- Histórico resumido: data, KM e o que foi feito.
- Rodapé: "Atualizado automaticamente pela Volvo em dd/mm/aaaa".
- Manter o selo "VENCIDO / VENCE EM X" no atalho Controle Revisional da tela inicial.

## Fontes de dados — levantamento feito em 10/11/2026 no Volvo Connect

| Dado | Onde está no Volvo Connect | Observação |
|---|---|---|
| Placa, VIN, chassi, modelo, motor, câmbio | Ativos › Detalhes do veículo | — |
| KM atual (odômetro) e horas de motor | Ativos › Detalhes do veículo | Atualizado várias vezes ao dia. Também disponível na API rFMS |
| Visitas à oficina (OS, data, KM, oficina) | **Histórico da oficina** (`volvoconnect.com/workshop`) | Ex. RMY9G32: 6 visitas em 12 meses |
| Peças/operações de cada visita | Histórico da oficina › Detalhes da visita | Código, descrição, quantidade, seção. **Não mostra valores.** Exporta PDF ou Excel/CSV **por visita** |
| Próxima manutenção recomendada | Ativos › lista de veículos, coluna "Próxima atividade" | Vem como **data** (ex.: 15/11/2026) |
| KM restante para a próxima revisão | API rFMS (`serviceDistance` em vehicle status/uptime) | **A confirmar** na integração que o site já usa |
| Luzes de alerta / falhas ativas | Ativos › Status atual (e API rFMS `tellTaleInfo`) | 44 veículos com luz acesa em 10/11 |
| Motorista | **Não existe na Volvo** (relatório por condutor = 0 condutores) | Buscar no sistema GT Minas: motorista vinculado à placa na data da visita |
| Data de saída / dias parados | **Não existe na Volvo** | Calcular: agrupar OS da mesma placa com até 5 dias de diferença numa "passagem" (entrada = 1ª OS, saída = última OS). Opcional: refinar com dias sem rodar (odômetro diário) |

### Exemplo real (RMY9G32 · FH13 · OS 3089093 · 08/05/2026 · 440.091 km · Treviso Betim – Pouso Alegre/MG)
Eixo principal da caixa de mudanças recondicionado (VO-4317300); conexão do resfriador de óleo da transmissão (VO-21380391); válvula de descarga do tanque de combustível (VO-21458307); válvula de alívio de pressão da bomba injetora (VO-2360700); mangueira, placa giratória, braçadeiras e presilhas.

## Integração — pedido aos programadores

1. **Verificar o acesso à API da Volvo** que o site já usa (provavelmente **rFMS**) e confirmar se ela traz `hrTotalVehicleDistance`, `serviceDistance` e `tellTaleInfo`.
2. **Solicitar à Volvo (Volvo Connect / Volvo Group Connected Solutions) acesso à API do Histórico da Oficina (workshop history / service history).** Se ainda não tivermos esse acesso, os programadores devem pedir. Pelo portal, os dados vêm de um serviço interno da Volvo (`ecs-workshophistory`), que não é público.
3. **Plano B, se a Volvo não liberar API de histórico:** rotina diária automatizada com login de serviço que baixa o Excel/CSV das **visitas novas** de cada placa no Histórico da oficina e grava no banco.
4. Job diário: (a) atualizar KM, `serviceDistance` e alertas de todos os veículos; (b) buscar visitas novas; (c) agrupar passagens e calcular dias parados; (d) cruzar motorista por placa e data; (e) recalcular a situação (vencida / vencendo / em dia).
5. Manter a planilha atual como contingência só até a integração estar validada.

### Modelo de dados sugerido
- `volvo_veiculo` (placa, vin, modelo, km_atual, horas_motor, service_distance_km, proxima_manutencao_data, alertas[], atualizado_em)
- `volvo_visita` (os, placa, data, km, oficina, cidade)
- `volvo_visita_item` (os, codigo, descricao, quantidade, sistema) — **sem valor**
- `volvo_passagem` (placa, entrada, saida, dias_parado, os[], motorista_id)

## Critérios de aceite
- [ ] Nenhuma ação manual do motorista; o app carrega os dados sozinho.
- [ ] Atualização automática diária, com data/hora da sincronização visível no site e no app.
- [ ] Só aparecem revisões Volvo; nenhum valor (R$) aparece.
- [ ] Site: linha do tempo por placa com entrada, saída, dias parados, motorista, KM, oficina e peças por sistema.
- [ ] App: km para a próxima revisão, nº de revisões, o que trocou na última e histórico resumido.
- [ ] Conferência: RMY9G32 mostra 6 visitas em 12 meses e a OS 3089093 com 10 itens.

Relacionada: demanda **#23 – ITEM REVISIONAL** (API/integrações).
