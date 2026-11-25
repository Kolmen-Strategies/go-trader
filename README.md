# go-trader

Kolmen Strategies fork of [richkuo/go-trader](https://github.com/richkuo/go-trader). A Go scheduler invokes Python strategy/data scripts, maintains SQLite paper/position state, applies risk checks, exposes an HTTP status API, and sends optional Discord/Telegram notifications. Selected platform paths can submit real orders; platform coverage is not uniform.

## Feature verification — 2026-11-25

Audited source: `b214f23`. “Implemented” describes source wiring. “Tests passed” refers to local automated tests, generally using fixtures/mocks; no broker account, real order, notifier recipient or deployed service was exercised. Missing advertised behavior and validation gaps are requirements in [ROADMAP.md](ROADMAP.md).

| Capability | Code / actual availability | Test evidence |
|---|---|---|
| Periodic strategy scheduling and Python subprocess execution | `scheduler/main.go`, `executor.go`, `strategy_interval.go`; script timeouts/concurrency limits | Go executor/interval/main tests passed; a running exchange-connected scheduler was not tested |
| Interactive and JSON config wizard | `scheduler/init.go`; defaults to a small paper setup | `init_test.go`, `config_test.go` passed; noninteractive generated config smoke checked |
| Config migration and safe hot reload | `scheduler/config_migration.go`, `config_reload.go`; current version **12** | Migration/reload tests passed. DM prompts only cover the fields in `configFieldRegistry`, not every field introduced since version 3 |
| Spot/perps/futures entry strategies and composed partial exits | `shared_strategies/open/registry.py`, `shared_strategies/close/registry.py`, `scheduler/strategy_composition.go` | Registry, signal, composition and partial-close tests passed. Counts vary by platform; inspect the registry instead of relying on a fixed “50+” deployment claim |
| Options strategies and theta harvesting | `shared_strategies/options/`, `shared_scripts/check_options.py`, `scheduler/options.go` | Options tests passed (including an explicit additional test invocation); Go theta/options tests passed. Scheduler options bookkeeping is simulated, not live order execution |
| Higher-timeframe and ADX/DI regime entry filters | `shared_tools/htf_filter.py`, `regime.py`, `scheduler/regime.go` | HTF/regime/wiring tests passed; regime support does not extend to options backtests |
| Binance US spot / Luno spot | `platforms/binanceus/adapter.py`, `platforms/luno/adapter.py`, `shared_scripts/check_strategy.py`; market-data adapters and simulated scheduler trading | Adapter/unit tests passed; no live order branch for these platforms in `scheduler/main.go` |
| Deribit / IBKR options | `platforms/deribit/`, `platforms/ibkr/`, `shared_scripts/check_options.py`; exchange quotes or pricing fallback and simulated options accounting | Adapter/pricing/options tests passed; `ibkr/paper_adapter.py` is separate TWS code, not proof of scheduler live options. TWS/live quotes not tested |
| Hyperliquid perpetuals | `shared_scripts/check_hyperliquid.py`, `platforms/hyperliquid/adapter.py`, Go perps/manual execution | Paper and real-order code paths exist; adapter, execution, fills, fee and reconciliation tests passed with mocks/local HTTP fixtures. No live venue test |
| TopStep futures | `shared_scripts/check_topstep.py`, `platforms/topstep/adapter.py`, Go futures execution | Yahoo Finance paper-data and TopStepX real-order paths exist; unit/close tests passed. Real TopStep account/data untested |
| Robinhood crypto / options | `shared_scripts/check_robinhood.py`, `platforms/robinhood/adapter.py`, `check_options.py` | Crypto has a scheduler real-order path; options scheduler uses simulated accounting even when live chains are requested. Adapter/crypto-close tests passed; real login/MFA/options orders untested |
| OKX spot / swaps / options | `shared_scripts/check_okx.py`, `platforms/okx/adapter.py`, `check_options.py` | Spot/swaps have scheduler real-order paths; options use simulated scheduler accounting. Adapter/close/mark tests passed; real or sandbox exchange calls untested |
| Paper accounting, fees/slippage, positions/trades and persistence | `scheduler/portfolio.go`, `fees.go`, `state.go`, `db.go`, `shared_tools/storage.py` | Go persistence/trade/fee tests and Python storage tests passed; estimated fees are model assumptions, not venue fee guarantees |
| Kill switch, per-strategy circuit breaker, notional cap, consecutive-loss pause | `scheduler/risk.go`, `kill_switch_close.go`, per-platform closers | Risk/close/operator-required tests passed. Automatic real flattening is platform-specific; OKX spot and Robinhood options require operator handling in circuit-breaker paths |
| Hyperliquid fixed/trailing stops, ATR stops and tiered reduce-only take profits | `scheduler/config.go`, `hyperliquid_protection.go`, `hyperliquid_trailing_stop.go`; default stop distance is **1× ATR** when stop fields are omitted (`default_stop_loss_atr_mult`, set 0 to opt out) | Protection/trailing/manual tests passed; no live trigger placement or restart test against the venue |
| Shared-wallet/coin reconciliation, margin/leverage and manual open/close | `scheduler/shared_wallet.go`, `hyperliquid_balance.go`, `manual.go`, `manual_protection.go` | Ownership, shared-wallet, manual-protection and fill tests passed; independent virtual strategies still share the venue's net position/leverage per coin |
| Dashboard/status/health/history and strategy summaries | `scheduler/server.go`, `leaderboard.go`, `sharpe.go`, `correlation.go` | Local HTTP/leaderboard/Sharpe/correlation tests passed; no authenticated web trading UI. Correlation warnings require explicit config enablement |
| Discord/Telegram notifications, summary frequency and update prompts | `scheduler/discord.go`, `telegram.go`, `notifier.go`, `summary_frequency.go`, `updater.go` | Formatter/routing/updater tests passed; no real message delivery, DM reply, self-update or service restart tested |
| TradingView export and Hyperliquid fee backfill | `scheduler/tradingview_export.go`, `backfill_hl_fees.go` | Export/backfill tests passed. Backfill is dry-run unless `--apply`; external import and historical venue reconciliation untested |
| Historical backtest, optimization, options/theta reporting | `backtest/backtester.py`, `optimizer.py`, `run_backtest.py`, `backtest_options.py`, `backtest_theta.py`, `reporter.py` | Backtest fixture/parity/fee/report/CLI tests passed; optimization quality, real historical data and strategy profitability not established |
| Linux service installation and multi-instance operation | `scripts/install-service.sh`, `systemd/` units | Source reviewed; systemd installation/reboot/isolation not exercised |

### Checks performed

Environment: Go 1.27.4 and Python 3.12 on Linux. The Go module declares 1.26.2; Python project declares 3.12+.

- `go -C scheduler test ./...` — **passed** (`trading-scheduler`).
- `uv sync --extra test`, then `.venv/bin/python -m pytest -q` — **829 passed**; warnings include deprecated datetime APIs. Default test paths do not include every test directory.
- `.venv/bin/python -m pytest -q shared_scripts shared_strategies/options` — **166 passed**, an additional run outside the default paths.
- Version-stamped Go build and JSON config generation — see setup below; no scheduler cycle against a live data source was run.

These checks verify specific local behaviors, not production account compatibility, profitability or live risk controls. CI definitions are in [.github/workflows/ci.yml](.github/workflows/ci.yml); their presence alone does not establish a successful hosted run.

## Setup and local commands

```bash
git clone https://github.com/Kolmen-Strategies/go-trader.git
cd go-trader
uv sync --extra test
# Build from the repository root; requires Go 1.26.2 or later.
go -C scheduler build -ldflags "-X main.Version=$(git describe --tags --always --dirty)" -o ../go-trader .
./go-trader init
# Noninteractive alternative that writes a paper configuration:
./go-trader init --json '{"assets":["BTC"],"enableSpot":true,"spotStrategies":["sma_crossover"],"spotCapital":1000,"spotDrawdown":10}' --output scheduler/config.json
```

Use the generated config and review its modes before running. Live execution is selected in strategy arguments on supported branches; the fact that an adapter accepts credentials does not establish a live scheduler path. Python scripts run through `.venv/bin/python3`; install dependencies in the repository venv. See [pyproject.toml](pyproject.toml) for direct dependencies, including `ccxt`, Hyperliquid SDK, NumPy, pandas, robin-stocks, pyotp and yfinance. Optional TWS functionality has additional imports such as `ib_insync` outside that list.

Configuration fields/defaults and validation are defined in [scheduler/config.go](scheduler/config.go); generation is in [scheduler/init.go](scheduler/init.go), and examples are in [scheduler/config.example.json](scheduler/config.example.json). Use the current generator rather than copying old version-10 JSON. Important options include:

- `db_file`, `log_dir`, `status_port` (default 8099), per-strategy `interval_seconds`.
- Strategy `type`: `spot`, `options`, `perps`, `futures`, `manual`; platform, script/args, paper/live mode, capital and drawdown settings.
- `open_strategy`, ordered `close_strategies` (maximum close fraction wins), parameters, higher-timeframe and regime filters.
- Portfolio kill-switch/notional settings, per-platform risk, correlation settings, and `risk_free_rate` for reported Sharpe calculations.
- Hyperliquid margin mode (default isolated), exchange/sizing leverage, optional fixed or trailing price/ATR stops and tiered take profits. Positive stop modes are mutually exclusive; shared-coin strategies are not isolated exchange accounts.
- Optional Discord/Telegram channels, summary frequency and `auto_update` (`off`, `daily`, `heartbeat`). Self-update and config-DM behavior require a configured notifier; not tested in this audit.

Store credentials outside tracked files; required names are documented in each platform adapter and [scheduler/config.example.json](scheduler/config.example.json). Paper data requests can still contact public market APIs.

For an intentionally configured deployment, `./go-trader --config scheduler/config.json --once` runs one cycle. It was **not** used as an audit check because configured live strategies can place orders. Service installation is described in [scripts/install-service.sh](scripts/install-service.sh) and [systemd/](systemd/). Distinct instances need their own working files/configuration/database and available status port; the binary alone is not a complete instance.

HTTP monitoring uses `GET /health`, `/status` and `/history` on the configured status port. Notifications, paper bookkeeping and exchange state can diverge; inspect reconciliation warnings. **Do not delete a state database to reset live positions**: deleting local state does not close exchange holdings. Preserve backups and reconcile first.

## Repository layout

- `scheduler/`: Go entry points, risk, order dispatch, SQLite, HTTP, notifications, migration, export and tests.
- `shared_scripts/`: Python check/execute/close/fetch entry points used by Go.
- `platforms/`: eight exchange/data adapter directories.
- `shared_strategies/`: entry, exit and options strategies; registry is the source of supported strategy names.
- `shared_tools/`: pricing, data, storage, ATR/regime and shared exchange helpers.
- `backtest/`: simulated historical engines, optimizer, reports and tests.
- `scripts/`, `systemd/`, `.github/workflows/`: deployment and CI tooling.
- [AGENTS.md](AGENTS.md), [SKILL.md](SKILL.md): upstream contributor/setup context; setup instructions may refer to the upstream repository.

## Provenance and license

This is a Kolmen fork; upstream release badges, community channels and setup guides do not certify this fork's runtime. Upstream community: [richkuo/go-trader](https://github.com/richkuo/go-trader). License: [MIT](LICENSE). Trading carries financial risk; no performance or loss-prevention guarantee follows from the test results.
