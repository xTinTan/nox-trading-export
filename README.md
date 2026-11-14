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

## Schema (`schema_version: 1`)

Datas em `AAAA-MM-DD`, instantes em ISO 8601 UTC (`...Z`). Montantes em EUR. Campos sem dados ficam a `0`, `[]` ou `null`: nunca são omitidos. Se uma secção não puder ser lida, aparece com `"erro": "<tipo>"`.

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
| `alertas` | Lista de frases curtas geradas por regras simples (vazia = tudo normal). |

### `forward` — livros em papel

`forward.livros` tem uma entrada por livro: `C` (ETFs, setup C1), `B` (cripto, B2), `B-short` (cripto short experimental, B3).

| Campo | Descrição |
|---|---|
| `setup`, `data_inicio` | Setup do livro e primeira vela do registo forward. |
| `ultimo_dia_processado`, `dias_processados` | Último dia de vela processado e total de dias. |
| `capital_inicial`, `equity`, `caixa` | Capital inicial; capital marcado a mercado; capital realizado (inicial + resultados fechados − custos pagos). |
| `pnl_aberto`, `nocional_aberto`, `retorno_pct` | Resultado latente das posições abertas, nocional a preço de entrada, retorno total em %. |
| `posicoes_abertas[]` | `ativo` (símbolo), `setup`, `direcao` (long/short), `entrada`, `stop`, `alvo`, `data_sinal`, `data_entrada`. |
| `trades_fechados` | `n`, `r_medio` (R líquido de custos), `win_rate_pct`, `resultado_liquido_eur` (depois de custos), `custos_eur`, `drawdown_max_eur`, `drawdown_max_pct` (curva diária). |
| `hoje` | `sinais_gerados` e `sinais_aceites` processados hoje. |
| `desde_corrida_anterior` | Trades abertos e fechados pela última corrida do forward. |
| `trades_por_setup` | Por setup: `fechados`, `abertos`, `progresso_trades_pct` (face a 30 trades fechados). |
| `progresso` | `semanas_decorridas`, `semanas_alvo` (8), `progresso_semanas_pct`, `trades_alvo_por_setup` (30), `criterios_cumpridos` (8 semanas **e** 30 trades fechados por setup — só então se avalia o resultado). |

Também: `forward.avisos_total`, `forward.avisos_hoje` (avisos de dados registados pelo motor).

### `forward_ran`

`ultima_corrida` (UTC), `estado` (`ok`, `em_curso`, `falhou`, `recusou_codigo_alterado`, `outra_corrida_em_curso`, `nunca_correu`), `origem` (`systemd` ou `manual`), `correu_hoje`.

### `recolha` — descarregamento diário de velas

`ultima_corrida`, `fim`, `estado`, `modo`, `correu_hoje`, `systemd_resultado`, `ativos_total`, `ativos_com_dados_de_ontem` (última vela esperada: ontem na cripto, última sessão NYSE em ETFs/ações), `ativos_desatualizados` (só símbolos), `falhas`, `ativos_falhados`, `incompletos`, `ok_com_avisos`.

### `scanner` — recolha de lançamentos de tokens

`container_estado`, `container_saude`, `container_saudavel`, `tamanho_db_mb`, `snapshots_24h`, `tokens_vistos` (total), `tokens_24h`, `quote_budget_esgotado_24h` (snapshots sem preço USD por orçamento de consultas esgotado), `checkpoints_extra_24h` (`15m`, `30m`, `2h`, `4h`), `checkpoints_extra_24h_total`, `checkpoints_extra_saltados_24h`. Nunca inclui endereços de tokens.

### `backup`

`ultima_linha_em`, `resultado` (`ok`/`falha`), `integrity_check_ok`, `idade_horas` da última linha do log de backup da base do scanner.

## Alertas (regras)

forward não correu hoje · forward falhou/recusou · código forward alterado · recolha não correu hoje · N ativos sem dados · N ativos falhados · container não saudável · scanner sem snapshots em 24 h · último backup falhou / sem integrity_check · backup com mais de 26 h · avisos novos no forward.
