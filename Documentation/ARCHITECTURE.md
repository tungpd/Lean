# LEAN Algorithmic Trading Engine - Architecture Visualization

## Core Engine Architecture

```mermaid
graph TB
    subgraph "Algorithm Layer"
        A[QCAlgorithm] --> AF[Algorithm Framework]
        A --> AC[Algorithm.CSharp]
        A --> AP[Algorithm.Python]

        subgraph "Algorithm Framework"
            AF --> AM[Alpha Models]
            AF --> PC[Portfolio Construction]
            AF --> EM[Execution Models]
            AF --> RM[Risk Management]
            AF --> US[Universe Selection]
        end
    end

    subgraph "Engine Layer"
        E[AlgorithmManager] --> DF[DataFeed]
        E --> TH[TransactionHandler]
        E --> RH[ResultHandler]
        E --> SH[SetupHandler]
        E --> RT[RealTimeHandler]
    end

    subgraph "Data Layer"
        DL[Common] --> BD[BaseData]
        DL --> SY[Symbol]
        DL --> SC[Securities]
        DL --> OR[Orders]
    end

    subgraph "Brokerage Layer"
        BL[Brokerages] --> IB[Interactive Brokers]
        BL --> BI[Binance]
        BL --> CO[Coinbase]
        BL --> OT[Other Brokerages...]
    end

    subgraph "Infrastructure"
        I[Configuration] --> CF[config.json]
        I --> LG[Logging]
        I --> MS[Messaging]
        I --> QU[Queues]
    end

    A --> E
    E --> DL
    E --> BL
    E --> I

    subgraph "Testing Layer"
        T[Tests] --> RT[Regression Tests]
        T --> UT[Unit Tests]
    end

    E --> T
    A --> T
```

## Data Flow Architecture

```mermaid
sequenceDiagram
    participant Config as config.json
    participant Setup as SetupHandler
    participant Algo as QCAlgorithm
    participant Engine as AlgorithmManager
    participant DataFeed
    participant Brokerage
    participant Result as ResultHandler

    Config->>Setup: Load environment config
    Setup->>Algo: Initialize algorithm
    Algo->>Engine: Register algorithm
    Engine->>DataFeed: Subscribe to data
    Engine->>Brokerage: Connect brokerage

    loop Event Loop
        DataFeed->>Engine: New data slice
        Engine->>Algo: OnData(slice)
        Algo->>Engine: Generate orders
        Engine->>Brokerage: Execute orders
        Brokerage->>Engine: Order updates
        Engine->>Algo: OnOrderEvent
        Engine->>Result: Update results
    end
```

## Framework Model Architecture

```mermaid
graph LR
    subgraph "Algorithm Framework"
        US[Universe Selection] --> AM[Alpha Model]
        AM --> PC[Portfolio Construction]
        PC --> EM[Execution Model]
        EM --> RM[Risk Management]
    end

    subgraph "Data Flow"
        US --> Insights[Portfolio Insights]
        Insights --> Targets[Portfolio Targets]
        Targets --> Orders[Orders]
        Orders --> RM
    end

    subgraph "Engine Coordination"
        Engine[AlgorithmManager] --> US
        Engine --> AM
        Engine --> PC
        Engine --> EM
        Engine --> RM
    end
```

## Component Relationships

```mermaid
classDiagram
    class IAlgorithm {
        +Initialize()
        +OnData(Slice)
        +OnOrderEvent(OrderEvent)
    }

    class QCAlgorithm {
        +SetStartDate()
        +SetCash()
        +AddEquity()
        +SetHoldings()
        +MarketOrder()
    }

    class AlgorithmManager {
        +Run()
        +SetTime()
        +ProcessData()
    }

    class IDataFeed {
        +Subscribe()
        +Unsubscribe()
        +GetNextData()
    }

    class ITransactionHandler {
        +ProcessTransaction()
        +GetOpenOrders()
    }

    class IResultHandler {
        +StoreResult()
        +SendStatus()
    }

    IAlgorithm <|-- QCAlgorithm
    AlgorithmManager --> IAlgorithm
    AlgorithmManager --> IDataFeed
    AlgorithmManager --> ITransactionHandler
    AlgorithmManager --> IResultHandler
```

## Environment Configuration Architecture

```mermaid
graph TD
    subgraph "Environment Types"
        BT[backtesting] --> BTH[BacktestingTransactionHandler]
        BT --> FDF[FileSystemDataFeed]
        BT --> BRH[BacktestingResultHandler]

        LP[live-paper] --> BTH
        LP --> LTD[LiveTradingDataFeed]
        LP --> LTR[LiveTradingResultHandler]

        LI[live-interactive] --> BRH[BrokerageTransactionHandler]
        LI --> LTD
        LI --> LTR
        LI --> IB[InteractiveBrokersBrokerage]
    end

    subgraph "Configuration"
        CF[config.json] --> ENV[environment]
        CF --> ATN[algorithm-type-name]
        CF --> AL[algorithm-language]
        CF --> ALOC[algorithm-location]
    end

    ENV --> BT
    ENV --> LP
    ENV --> LI
```

## Testing Architecture

```mermaid
graph TB
    subgraph "Test Categories"
        RT[Regression Tests] --> ART[Algorithm Regression Tests]
        RT --> FRT[Framework Regression Tests]

        UT[Unit Tests] --> ET[Engine Tests]
        UT --> AT[Algorithm Tests]
        UT --> BT[Brokerage Tests]
    end

    subgraph "Test Infrastructure"
        NUnit --> TestRunner
        TestRunner --> ART
        TestRunner --> FRT
        TestRunner --> ET
        TestRunner --> AT
        TestRunner --> BT
    end

    subgraph "Test Data"
        TD[Test Data] --> HD[Historical Data]
        TD --> MD[Market Data]
        TD --> OD[Order Data]
    end

    ART --> TD
    FRT --> TD
```

## Key Architectural Patterns

### Handler Pattern

- Each major component (DataFeed, TransactionHandler, ResultHandler) implements an interface
- Different implementations for backtesting vs live trading
- Configurable via `config.json` environments

### Event-Driven Architecture

- AlgorithmManager processes time slices in event loop
- Data arrives asynchronously through DataFeed
- Orders processed asynchronously through TransactionHandler
- Results updated continuously through ResultHandler

### Modular Framework System

- Optional framework components can be mixed and matched
- Models emit events rather than direct calls
- Engine coordinates between models
- Framework models are silent (no logging/charting)

### Configuration-Driven Behavior

- Single `config.json` controls all behavior
- Environment switching changes handler implementations
- Brokerage-specific configurations
- Algorithm selection and parameters

### Partial Class Architecture

- QCAlgorithm split across multiple files by concern
- `QCAlgorithm.cs` - Core lifecycle
- `QCAlgorithm.Trading.cs` - Order APIs
- `QCAlgorithm.Framework.cs` - Framework integration
- `QCAlgorithm.History.cs` - Historical data
- `QCAlgorithm.Indicators.cs` - Technical indicators
- `QCAlgorithm.Universe.cs` - Universe selection

This architecture enables LEAN to support both simple algorithmic trading strategies and complex institutional-grade portfolio management systems through its modular, pluggable design.
