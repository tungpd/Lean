# LEAN Algorithmic Trading Engine - AI Agent Instructions

## Project Overview

LEAN is an event-driven, institutional-grade algorithmic trading engine supporting multiple asset classes (equities, forex, crypto, options, futures) and brokerages. It runs both backtesting and live trading with a modular, pluggable architecture.

**Key Principle**: LEAN uses **configuration-driven polymorphism** - the `environment` setting in `config.json` determines which handler implementations are instantiated at runtime, enabling the same codebase to switch between backtesting and live trading seamlessly.

## Architecture Fundamentals

### Core Engine Components (Engine/)
The engine operates as an event loop coordinating pluggable handlers:

- **AlgorithmManager**: Main event loop in `Run()` method processing time slices, handling splits/dividends, managing warmup. Enforces time limits via `AlgorithmTimeLimitManager` to prevent infinite loops.
- **DataFeed**: Subscription-based data streaming. Implementations:
  - `FileSystemDataFeed` for backtesting (reads from Data/ directory)
  - `LiveTradingDataFeed` for live (streams from brokerage/data queue handlers)
- **TransactionHandler**: Order execution with async order management:
  - `BacktestingTransactionHandler`: Instant fills with slippage models
  - `BrokerageTransactionHandler`: Real broker integration, handles order status updates
- **ResultHandler**: Results aggregation and reporting:
  - `BacktestingResultHandler`: Statistical analysis, charting
  - `LiveTradingResultHandler`: Real-time updates, notifications
- **SetupHandler**: Algorithm initialization and brokerage configuration
  - `BacktestingSetupHandler`: Basic initialization
  - `BrokerageSetupHandler`: Brokerage connection, account sync, live holdings restoration

### Algorithm Structure (Algorithm/)
**QCAlgorithm** is the base class split across partial classes by concern:
- `QCAlgorithm.cs`: Core algorithm lifecycle (Initialize, OnData, OnOrderEvent). Contains ~3800 lines - use partial class structure to navigate.
- `QCAlgorithm.Framework.cs`: Framework model integration (SetAlpha, SetPortfolioConstruction, SetExecution, SetRiskManagement)
- `QCAlgorithm.Trading.cs`: Order management APIs - `MarketOrder()`, `LimitOrder()`, `SetHoldings()` return `OrderTicket` for async tracking
- `QCAlgorithm.History.cs`: Historical data retrieval via `History()` methods
- `QCAlgorithm.Indicators.cs`: Technical indicator factory methods (SMA, EMA, RSI, MACD, etc.)
- `QCAlgorithm.Universe.cs`: Universe selection (AddEquity, AddOption, coarse/fine fundamental filtering)
- `QCAlgorithm.Python.cs`: Python interop wrappers for PythonNet bridge

### Algorithm Framework (Algorithm.Framework/)
Optional modular system for institutional-style algorithms with separation of concerns:
- **Alpha Models** (`Alphas/`): Generate `Insight` objects (price predictions with direction/magnitude/confidence/duration). Examples: `MacdAlphaModel`, `RsiAlphaModel`, `EmaCrossAlphaModel`, `ConstantAlphaModel`
- **Portfolio Construction** (`Portfolio/`): Convert insights to `PortfolioTarget[]`. Examples: `EqualWeightingPortfolioConstructionModel`, `MeanVarianceOptimizationPortfolioConstructionModel`, `InsightWeightingPortfolioConstructionModel`
- **Execution Models** (`Execution/`): Execute targets to orders. Examples: `ImmediateExecutionModel`, `VolumeWeightedAveragePriceExecutionModel`, `StandardDeviationExecutionModel`
- **Risk Management** (`Risk/`): Cancel/reduce positions based on risk rules (drawdown limits, volatility)
- **Universe Selection** (`Selection/`): Filter securities for alpha generation via `CoarseFundamentalUniverseSelectionModel`, `FineFundamentalUniverseSelectionModel`, `ManualUniverseSelectionModel`
- **Critical Pattern**: Models must NOT log/chart directly - they emit events/insights; the algorithm handles output. This keeps models reusable and testable.

### Data Layer (Common/)
- **BaseData**: All market data types inherit from this (TradeBar, QuoteBar, Tick)
- **Symbol**: Canonical security identifier with market/security type metadata
- **Securities**: Security objects manage holdings, margin, fills (Equity, Option, Future, Forex, Crypto)
- **Orders**: Order types (Market, Limit, StopMarket, StopLimit, MarketOnOpen/Close, ComboMarket)

## Key Development Patterns

### Configuration-Driven Polymorphism
LEAN's architecture centers on `Launcher/config.json` which controls which implementations are loaded:
- **Environment Switcher**: Change `"environment"` setting to switch handler stack:
  - `"backtesting"`: FileSystemDataFeed + BacktestingTransactionHandler + BacktestingResultHandler
  - `"live-paper"`: LiveTradingDataFeed + BacktestingTransactionHandler (paper trading)
  - `"live-interactive"`: LiveTradingDataFeed + BrokerageTransactionHandler + InteractiveBrokersBrokerage
  - `"live-binance"`, `"live-zerodha"`, etc.: Brokerage-specific environments
- **Algorithm Configuration**: 
  - `algorithm-type-name`: Class name (e.g., "BasicTemplateFrameworkAlgorithm")
  - `algorithm-language`: "CSharp" or "Python"
  - `algorithm-location`: DLL path or .py file path
- **Handler Pattern**: All major components implement interfaces (`IDataFeed`, `ITransactionHandler`, `IResultHandler`, `ISetupHandler`). The config file specifies concrete implementations at runtime.

### Algorithm Development
```csharp
public class MyAlgorithm : QCAlgorithm
{
    public override void Initialize()
    {
        SetStartDate(2020, 1, 1);
        SetCash(100000);
        AddEquity("SPY", Resolution.Daily);
        // Framework approach:
        SetUniverseSelection(new ManualUniverseSelectionModel(Symbol.Create("SPY", SecurityType.Equity, Market.USA)));
        SetAlpha(new MyAlphaModel());
    }

    public override void OnData(Slice data)
    {
        // Classic approach - direct trading
        if (!Portfolio.Invested)
        {
            SetHoldings("SPY", 1.0);
        }
    }
}
```

### Testing Philosophy
- **Regression Tests**: Algorithms implement `IRegressionAlgorithmDefinition` with expected statistics
  - Must define: `AlgorithmStatus`, `CanRunLocally`, `Languages`, `DataPoints`, `AlgorithmHistoryDataPoints`, `ExpectedStatistics`
  - Example: `BasicTemplateFrameworkAlgorithm.cs` shows complete implementation pattern
- **NUnit**: Standard test framework with `[TestFixture]` and `[Test]` attributes
- Tests in `Tests/` mirror source structure (Tests/Engine/, Tests/Algorithm/, etc.)
- Run tests: `dotnet test` or use VS Code Test Explorer
- **Test Categories**: `[Category("RegressionTests")]` for algorithm validation, `[Category("TravisExclude")]` for long-running tests

## Development Workflows

### Building
```bash
dotnet build                                    # Standard build
dotnet build --no-incremental                   # Clean rebuild
dotnet clean                                    # Clean build artifacts
```
Use VS Code tasks (Ctrl+Shift+B): `build`, `rebuild`, `clean`

**Note**: Project targets .NET 9.0 framework.

### Running Algorithms
1. Edit `Launcher/config.json`:
   - Set `algorithm-type-name` (e.g., "BasicTemplateFrameworkAlgorithm")
   - Set `algorithm-language` ("CSharp" or "Python")
   - Set `algorithm-location` (DLL path or .py file path)
2. Run: `cd Launcher/bin/Debug && dotnet QuantConnect.Lean.Launcher.dll`
3. Or use VS Code launch configuration (F5) with "Launch" profile
4. Config changes require rebuild but not always a full rebuild

### Python Integration
- Python algorithms inherit from `QCAlgorithm` in .py files
- Algorithm.Python/ contains examples (use snake_case: `def initialize(self):`, `def on_data(self, data):`)
- Requires Python 3.11, configured via `python-venv` in config.json
- Python.NET bridges C# and Python runtime
- Debug Python: Set `"debugging": true` and `"debugging-method": "DebugPy"` in config.json, then use "Attach to Python" launch configuration (port 5678)

### Docker Development
- **lean CLI**: Recommended for algorithm development (`pip install lean`)
- **Dev Container**: Full environment in `.devcontainer/` for contributors
- Images: `quantconnect/lean:latest`, `quantconnect/research:latest`

### VS Code Development Setup
- **Tasks** (Ctrl+Shift+P > Tasks: Run Task): `build`, `rebuild`, `clean`, `start research`
- **Launch Configurations** (F5): "Launch" builds and runs algorithm, "Attach to Python" for debugging Python algorithms
- **Extensions**: C# Dev Kit, Python (for Python algorithms), C# (for IntelliSense)
- **Settings**: 4-space soft tabs, Microsoft C# formatting

### Brokerage Integration Patterns
- **Environment Configuration**: `live-{brokerage}` environments in `config.json` (e.g., `live-interactive`, `live-zerodha`, `live-binance`, `live-kraken`, `live-oanda`)
- **Handler Pattern**: Each brokerage implements `BrokerageTransactionHandler` and data queue handler
- **Configuration Keys**: Brokerage-specific settings (API keys, accounts) in `config.json`
- **Paper Trading**: `live-paper` environment uses `BacktestingTransactionHandler` for risk-free testing

### ToolBox Utilities
- **Data Management**: Command-line tools for downloading and converting market data
- **Location**: `ToolBox/` directory with over 15 utilities
- **Usage**: `--app=<tool-name>` is the required parameter (e.g., `--app=RandomDataGenerator`)
- **Common Tools**: Data downloaders (GDAX, IB, Bitfinex), converters (AlgoSeek, Kaiko, QuantQuote), CoarseUniverseGenerator
- Run: `dotnet QuantConnect.ToolBox.dll --app=<tool> --help` for tool-specific parameters

## Project Conventions

### Coding Standards
- **Microsoft C# Guidelines**: Soft tabs (4 spaces), PascalCase for public members
- **Partial Classes**: QCAlgorithm split by concern across multiple files
- **No Logging in Framework Modules**: Alpha/Portfolio/Execution models should be silent (users handle logging)
- **Documentation Attributes**: `[DocumentationAttribute(TradingAndOrders)]` for API grouping

### File Organization
- **Algorithm.CSharp/**: Example algorithms (many are regression tests)
- **Common/**: Shared data structures, no engine logic
- **Engine/**: Core execution engine, no algorithm-specific logic
- **Brokerages/**: Brokerage integrations (IB, GDAX, Oanda, etc.)
- **Indicators/**: Technical analysis indicators
- **Tests/**: Mirrors source structure with NUnit tests

### Naming Patterns
- Regression algorithms: `*RegressionAlgorithm.cs` implementing `IRegressionAlgorithmDefinition`
- Handlers: `I*Handler` interface, implementations end with handler type (e.g., `BacktestingResultHandler`)
- Models: Framework models end with model type (e.g., `EqualWeightingPortfolioConstructionModel`)

## Common Pitfalls & Solutions

### Time Zone Handling
- **Algorithm.Time**: Always in algorithm's time zone (default: Eastern)
- **Algorithm.UtcTime**: UTC time
- Use `ConvertToUtc()` / `ConvertFromUtc()` for conversions

### Data Subscription
- `AddEquity()` returns `Security` object - use for reference
- Data arrives in `OnData(Slice data)` where `data.Bars`, `data.QuoteBars`, `data.Ticks` are keyed by Symbol
- Warmup: Set warmup period with `SetWarmUp(TimeSpan)` or `SetWarmUp(barCount, resolution)`

### Order Management
- Orders are asynchronous - check `OrderTicket.Status` or handle in `OnOrderEvent()`
- `SetHoldings(symbol, percentage)` calculates order quantity automatically
- Framework approach: Portfolio models emit `PortfolioTarget[]`, execution models convert to orders

### Debugging and Performance
- **AlgorithmManager.TimeLimit**: Monitors execution time per algorithm loop
- **PerformanceTrackingTool**: Tracks DataPoints and AlgorithmHistoryDataPoints for regression validation
- **Debugpy**: For Python algorithm debugging (set `debugging-method` to "DebugPy" in config)
- **Time Loop Limits**: Configurable via `algorithm-manager-time-loop-maximum` for preventing infinite loops

## Critical Files to Reference

- `Algorithm/QCAlgorithm.cs`: Core algorithm API surface
- `Engine/AlgorithmManager.cs`: Event loop and time slice processing
- `Launcher/config.json`: Configuration schema and environment definitions
- `Algorithm.CSharp/BasicTemplateFrameworkAlgorithm.cs`: Framework usage example
- `Algorithm.CSharp/OrderTicketDemoAlgorithm.cs`: Order management patterns
- `CONTRIBUTING.md`: Git workflow and contribution guidelines

## Contributing Checklist

1. **Branch Naming**: `bug-<issue#>-description` or `feature-<issue#>-description`
2. **Tests Required**: All PRs need unit tests demonstrating functionality
3. **No Direct Logging**: Framework modules should not log/chart (users control this)
4. **Regression Tests**: Algorithm changes need expected statistics in `IRegressionAlgorithmDefinition`
5. **Code Style**: 4-space soft tabs, Microsoft C# conventions

## Useful Commands

```bash
# Build and run
dotnet build
cd Launcher/bin/Debug && dotnet QuantConnect.Lean.Launcher.dll

# Run tests
dotnet test                                          # All tests
dotnet test --filter "FullyQualifiedName~Engine"     # Engine tests only

# Lean CLI (algorithm development)
lean project-create "MyAlgorithm"
lean backtest "MyAlgorithm"
lean research                                        # Start Jupyter
```

## Additional Resources

- [LEAN Documentation](https://www.lean.io/docs/)
- [Algorithm Framework Guide](https://www.quantconnect.com/docs/algorithm-framework/overview)
- [QuantConnect Forum](https://www.quantconnect.com/forum)
- [Discord Community](https://www.quantconnect.com/discord)
