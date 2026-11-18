# nox-trading-export

Resumo diário, **só números**, de um registo de trading em papel (paper trading) e da infraestrutura de recolha de dados.

- Gerado automaticamente pelo servidor `nox-server` todos os dias por volta das 06:30 UTC.
- **Sem segredos:** sem chaves, credenciais, wallets, endereços de tokens, IPs, caminhos ou nomes de utilizador. Antes de cada push um verificador bloqueia o envio se encontrar algo parecido com um segredo (valores de configuração secretos, sequências base58/hex com 32 ou mais caracteres, blocos de chave privada PEM, caminhos absolutos do servidor, endereços IP).
- Não é aconselhamento financeiro. Os livros são simulados (paper), com regras congeladas antes do início.

## Ficheiros

| Ficheiro | Conteúdo |
|---|---|
| `latest.json` | Último resumo. |
| `history/AAAA-MM-DD.json` | Cópia diária (data UTC de `generated_at`). |

## Schema (`schema_version: 2`)

Datas em `AAAA-MM-DD`, instantes em ISO 8601 UTC (`...Z`). Montantes em EUR. Campos sem dados ficam a `0`, `[]` ou `null`: nunca são omitidos. Se uma secção não puder ser lida, aparece com `"erro": "<tipo>"`. Exceção: `forward.livros["B-short"].funding` fica a `null` quando a API externa não responde (ver abaixo).

**Mudanças na versão 2** (só acrescentos, só observação; as regras e decisões do registo forward não mudam): `regime` no topo; em cada livro, as listas `sinais` e `trades` e o campo `regime_entrada`; `funding` no livro `B-short`.

### Topo

| Campo | Descrição |
|---|---|
| `schema_version` | Versão deste formato (muda só se campos existentes mudarem de significado). |
| `generated_at` | Instante da geração (UTC). |
| `host` | Sempre `nox-server`. |
| `versoes.forward` | Identificador do código congelado do registo forward (16 hex do sha256 do ficheiro de hashes). |
| `versoes.forward_codigo_integro` | `true` se todos os ficheiros congelados continuam iguais ao registado. |
| `versoes.universe` | 16 hex do sha256 da lista de ativos. |
| `versoes.scanner` | Versão do código do scanner. |
| `regime` | Etiqueta de regime do mercado cripto no último dia com vela de BTC (ver secção `regime`). |
| `alertas` | Lista de frases curtas geradas por regras simples (vazia = tudo normal). |

### `forward` — livros em papel

`forward.livros` tem uma entrada por livro: `C` (ETFs, setup C1), `B` (cripto, B2), `B-short` (cripto short experimental, B3).

| Campo | Descrição |
|---|---|
| `setup`, `data_inicio` | Setup do livro e primeira vela do registo forward. |
| `ultimo_dia_processado`, `dias_processados` | Último dia de vela processado e total de dias. |
| `capital_inicial`, `equity`, `caixa` | Capital inicial; capital marcado a mercado; capital realizado (inicial + resultados fechados − custos pagos). |
| `pnl_aberto`, `nocional_aberto`, `retorno_pct` | Resultado latente das posições abertas, nocional a preço de entrada, retorno total em %. |
| `posicoes_abertas[]` | `ativo` (símbolo), `setup`, `direcao` (long/short), `entrada`, `stop`, `alvo`, `data_sinal`, `data_entrada`, `regime_entrada`. |
| `trades[]` | Cada trade fechado: `ativo`, `setup`, `direcao`, `data_sinal`, `data_entrada`, `data_saida`, `motivo_saida` (stop/alvo/tempo), `dias`, `r_liquido` (R líquido de custos, sem funding), `pnl_eur`, `regime_entrada`. |
| `sinais[]` | Cada sinal processado: `data_sinal`, `ativo`, `decisao` (aceite/rejeitado), `motivo` (da rejeição), `regime_entrada`. |
| `trades_fechados` | `n`, `r_medio` (R líquido de custos), `win_rate_pct`, `resultado_liquido_eur` (depois de custos), `custos_eur`, `drawdown_max_eur`, `drawdown_max_pct` (curva diária). |
| `hoje` | `sinais_gerados` e `sinais_aceites` processados hoje. |
| `desde_corrida_anterior` | Trades abertos e fechados pela última corrida do forward. |
| `trades_por_setup` | Por setup: `fechados`, `abertos`, `progresso_trades_pct` (face a 30 trades fechados). |
| `progresso` | `semanas_decorridas`, `semanas_alvo` (8), `progresso_semanas_pct`, `trades_alvo_por_setup` (30), `criterios_cumpridos` (8 semanas **e** 30 trades fechados por setup — só então se avalia o resultado). |

Também: `forward.avisos_total`, `forward.avisos_hoje` (avisos de dados registados pelo motor).

`regime_entrada` é o objeto `regime` (mesmos campos) do **dia do sinal**, ou seja, da última vela de BTC fechada antes da entrada (sem olhar para o futuro). Fica `null` se não houver dados.

### Forward v2 (desde 17/11/2026): C1 arquivado, C1R e C1R-placebo

Só acrescentos. Nenhum campo existente mudou de nome ou significado.

- `versoes.forward_rotulo`: rótulo da versão do código congelado (`"v2"`).
- `forward.livros.<livro>.estado`: `"ativo"` ou `"arquivado"`. `forward.livros.C.arquivado_desde` é a primeira vela sem entradas novas do C1. A posição que estava aberta fecha pelas regras do C1.
- `forward.livros_novos`: tem os livros `C1R` (C1-ATR alargado: 20 ETFs + 130 ações, filtro SPY acima da SMA200, stop 1,5×ATR14, alvo 2R, risco de 1% do capital da Conta C) e `C1R-placebo` (sombra: mesmas regras, entradas sorteadas, capital fictício que não conta para a Conta C).
  - Têm os mesmos campos de um livro de `forward.livros`, com uma exceção: `sinais_ultimo_dia` (só os sinais do último dia processado) substitui a lista `sinais` completa.
  - Campos extra: `estado` (`ativo`/`parado_drawdown`), `sombra` (bool), `seed` (no placebo; `null` no C1R) e `universo_n`.
  - Em `progresso`: `semanas_alvo` e `progresso_semanas_pct` ficam `null`, e `criterios_cumpridos` é ter 30 trades fechados.
- `forward.conta_c`: Conta C combinada (capital inicial + P&L do C1 + P&L do C1R).
  - Campos: `capital_inicial`, `caixa`, `equity`, `pnl_c1`, `pnl_c1r`, `nocional_aberto`.
  - `drawdown_pct`: maior queda em % desde 17/11, com pico inicial no capital de 16/11.
  - `drawdown_limite_pct`: 15.
  - `paragem`: `true` quando o drawdown passou o limite e o C1R e o placebo deixaram de abrir posições.
- `forward.filtro_regime_spy`: `data`, `spy_fecho`, `spy_sma200`, `acima` (o C1R só abre posições novas com `acima: true`).
- `forward.avaliacao_c1r`: avaliação pré-registada ao fim de 30 trades fechados do C1R.
  - Campos: `trades_fechados`, `trades_alvo` (30), `pronta`, `data_avaliacao`, `r_medio`, `ic90_lo`, `ic90_hi` (bootstrap por dia de entrada), `r_medio_placebo`, `retorno_c1r_pct`, `retorno_spy_pct` (comprar e manter SPY desde 17/11).
  - `passa` fica `null` até haver 30 trades. Depois é `true` só se o IC 90% estiver acima de 0, o R médio for maior que o do placebo e o retorno for ≥ ao do SPY.
- Alertas novos: `C1R: paragem por drawdown (Conta C acima de 15%)`, `C1R: avaliacao pronta (30 trades fechados)`, `filtro SPY indisponivel`, `forward v2: erro em <secção>`.

### `forward.livros["B-short"].funding` — funding real (Hyperliquid)

Só observação: o livro continua a usar a sua suposição fixa de funding; isto mede o que um short perpétuo teria realmente recebido ou pago.

| Campo | Descrição |
|---|---|
| `por_posicao[]` | Uma entrada por posição aberta e por trade fechado do livro (ver abaixo). |
| `total_eur` | Soma do funding em EUR das posições com dados. |
| `total_r` | Soma do funding em R das posições com dados. |
| `fonte` | Sempre `hyperliquid`. |
| `sem_dados` | Símbolos sem mercado perpétuo ativo na Hyperliquid (não entram nos totais). |

Campos de `por_posicao[]`: `ativo`, `moeda_hyperliquid` (ex.: SHIB → `kSHIB`), `estado` (`aberta`/`fechada`), `data_entrada`, `data_saida`, `horas` (horas na janela), `horas_com_taxa` (horas com taxa publicada; menor que `horas` se o mercado ainda não existia), `funding_eur`, `funding_r`, `r_bruto`, `r_liquido`. Posições sem dados levam `"sem_dados": true` e os valores a `null`.

Cálculo:

- **Fonte:** `POST https://api.hyperliquid.xyz/info` com `{"type": "fundingHistory"}`. É uma API pública, sem chave, e devolve uma taxa por hora.
- **Funding:** `funding_eur = Σ taxa_horária × nocional × (−direção)`. Um short **recebe** quando a taxa é positiva e paga quando é negativa.
  - O nocional é o de entrada, fixo.
  - Os preços são tratados como EUR, sem conversão cambial, como no resto do registo.
- **Janela:** de 00:00 UTC do dia de entrada (entrada à abertura) até ao fim do dia de saída, ou até à última hora fechada se a posição estiver aberta. A hora exata de saída dentro do dia não é conhecida, por isso conta-se o dia inteiro.
- **R:** `funding_r = funding_eur / risco da posição`.
  - `r_bruto`: R líquido de custos e sem funding (fechados: valor registado; abertos: R se fechasse ao último fecho).
  - `r_liquido = r_bruto + funding_r`.
- **Se a API falhar:** uma única tentativa por corrida. Se falhar, `funding` fica `null` e aparece um alerta; o resto do export sai normalmente.
- **Cache:** as taxas já obtidas ficam em cache local.

### `regime` — etiqueta de regime (BTC)

Calculada a partir das velas diárias de BTC, usando só velas até ao dia em causa.

| Campo | Descrição |
|---|---|
| `data` | Dia da vela de BTC usada. |
| `btc_vs_sma50`, `btc_vs_sma200` | Fecho `acima` ou `abaixo` da média simples de 50 / 200 fechos. |
| `var_20d_pct` | Variação do fecho face a 20 velas antes, em %. |
| `atr_pct` | ATR14 (Wilder) em % do fecho. |
| `tendencia` | `subida` se o fecho estiver acima das duas médias **e** `var_20d_pct > 0`; `descida` se estiver abaixo das duas **e** `var_20d_pct < 0`; `lateral` nos restantes casos. |
| `volatilidade` | `alta` se `atr_pct` for maior que a mediana do `atr_pct` dos últimos 250 dias (inclui o dia); senão `normal`. |

Com histórico insuficiente (menos de 200 velas para as médias, 264 para a volatilidade), os campos ficam `null`.

### `forward_ran`

`ultima_corrida` (UTC), `estado` (`ok`, `em_curso`, `falhou`, `recusou_codigo_alterado`, `outra_corrida_em_curso`, `nunca_correu`), `origem` (`systemd` ou `manual`), `correu_hoje`.

### `recolha` — descarregamento diário de velas

`ultima_corrida`, `fim`, `estado`, `modo`, `correu_hoje`, `systemd_resultado`, `ativos_total`, `ativos_com_dados_de_ontem` (última vela esperada: ontem na cripto, última sessão NYSE em ETFs/ações), `ativos_desatualizados` (só símbolos), `falhas`, `ativos_falhados`, `incompletos`, `ok_com_avisos`.

### `scanner` — recolha de lançamentos de tokens

`container_estado`, `container_saude`, `container_saudavel`, `tamanho_db_mb`, `snapshots_24h`, `tokens_vistos` (total), `tokens_24h`, `quote_budget_esgotado_24h` (snapshots sem preço USD por orçamento de consultas esgotado), `checkpoints_extra_24h` (`15m`, `30m`, `2h`, `4h`), `checkpoints_extra_24h_total`, `checkpoints_extra_saltados_24h`. Nunca inclui endereços de tokens.

### `backup`

`ultima_linha_em`, `resultado` (`ok`/`falha`), `integrity_check_ok`, `idade_horas` da última linha do log de backup da base do scanner.

## Alertas (regras)

forward não correu hoje · forward falhou/recusou · código forward alterado · recolha não correu hoje · N ativos sem dados · N ativos falhados · container não saudável · scanner sem snapshots em 24 h · último backup falhou / sem integrity_check · backup com mais de 26 h · avisos novos no forward · regime: dados de BTC indisponíveis · funding B-short: API Hyperliquid indisponível / erro a calcular / sem dados para <símbolos>.
