# Blockscout W Chain Backend

This repository is a fork of [Blockscout](https://github.com/blockscout/blockscout), customized for **W Chain**. It serves as the backend explorer, providing an API and web interface to inspect the W Chain network.

## 🏗 Architecture

The backend follows an umbrella application structure, where the **Indexer** fetches data from the W Chain Node and stores it in **PostgreSQL**. The **Web** and **API** applications serve this data to users.

```mermaid
graph TD
    User[User / Frontend] -->|HTTP/WebSocket| Web[Blockscout Web]
    User -->|API Requests| API[Blockscout API]
    
    subgraph "Blockscout Backend"
        Web
        API
        Indexer[Indexer Process]
    end
    
    Indexer -->|Fetch Blocks/Txs| Node["W Chain Node (RPC)"]
    Indexer -->|Store Data| DB[(PostgreSQL)]
    
    Web -->|Query Data| DB
    API -->|Query Data| DB
    
    Web -->|Real-time Updates| Redis[("Redis Pub/Sub")]
    Indexer -->|Publish Events| Redis
    
    classDef external fill:#f9f,stroke:#333,stroke-width:2px;
    class Node,User external
```

### Indexing Flow

Blockscout uses two modes of indexing:
1.  **Real-time**: Listens for new blocks via RPC.
2.  **Catch-up**: Fills in missing historical blocks.

```mermaid
sequenceDiagram
    participant Node as W Chain Node
    participant Fetcher as Block Fetcher
    participant Parser as Data Parser
    participant DB as PostgreSQL
    
    loop Real-time Indexing
        Fetcher->>Node: Poll for new block
        Node-->>Fetcher: Block Data + Txs
        Fetcher->>Parser: Parse Block & Transactions
        Note right of Parser: Handles W Chain's "txType"
        Parser->>DB: Insert Block, Txs, Addresses
    end
    
    loop Catch-up Indexing
        Fetcher->>DB: Check missing blocks
        DB-->>Fetcher: List of missing ranges
        Fetcher->>Node: Batch request blocks
        Node-->>Fetcher: Batch response
        Fetcher->>Parser: Parse
        Parser->>DB: Bulk Insert
    end
```

## 🧩 W Chain Modifications

This fork includes specific adaptations for W Chain:

*   **RPC Handling**: Patched `apps/ethereum_jsonrpc` to support the custom `txType` field returned by W Chain nodes (preventing `FunctionClauseError` crashes).
*   **Theming**: Custom branding colors defined in `apps/block_scout_web/assets/css/theme/_wanchain_variables.scss`.
*   **Configuration**: Tuned for W Chain's Chain ID (171717) and block times.

## 🚀 Production Deployment

**Note**: This guide assumes a direct host installation on **Ubuntu 20.04/22.04**. For full details, refer to the [Deployment Guide](BLOCKSCOUT_DEPLOYMENT_GUIDE.md).

### 1. Prerequisites

Ensure the following versions are installed. We recommend using `asdf` for version management.

*   **Erlang**: 27.x
*   **Elixir**: 1.17.x (OTP 27)
*   **PostgreSQL**: 14
*   **Redis**: 6+
*   **Node.js**: 18.x (for asset building)

System packages:
```bash
sudo apt install -y build-essential git curl automake libtool inotify-tools libgmp-dev make g++
```

### 2. Database Setup

```bash
# Create user and database
sudo -u postgres psql -c "CREATE USER blockscout WITH PASSWORD 'your_password';"
sudo -u postgres psql -c "CREATE DATABASE blockscout;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE blockscout TO blockscout;"
# Grant schema permissions
sudo -u postgres psql -d blockscout -c "GRANT ALL ON SCHEMA public TO blockscout;"
```

### 3. Installation

```bash
# Clone the repository
git clone https://github.com/blockscout/blockscout blockscout-backend
cd blockscout-backend

# Install Elixir dependencies
mix do deps.get, local.rebar --force, deps.compile

# Compile the application
mix compile

# Build frontend assets
cd apps/block_scout_web/assets && npm install && node_modules/webpack/bin/webpack.js --mode production && cd -
cd apps/explorer && npm install && cd -
mix phx.digest
```

### 4. Configuration

Create a `prod.env` file with the following W Chain configuration:

```bash
# Database
DATABASE_URL=postgresql://blockscout:your_password@localhost:5432/blockscout
POOL_SIZE=200

# Network
NETWORK="W Chain"
CHAIN_ID=171717
BLOCKSCOUT_HOST=scan.w-chain.com
BLOCKSCOUT_PROTOCOL=https
COIN=W
COIN_NAME="W Chain"

# RPC (W Chain Node)
ETHEREUM_JSONRPC_VARIANT=geth
ETHEREUM_JSONRPC_HTTP_URL=https://rpc.w-chain.com
ETHEREUM_JSONRPC_WS_URL=wss://rpc.w-chain.com/ws

# Indexer Tuning
INDEXER_MEMORY_LIMIT=16G
INDEXER_DISABLE_BLOCK_REWARD_FETCHER=true
INDEXER_INTERNAL_TRANSACTIONS_CONCURRENCY=10

# API
API_V2_ENABLED=true
PORT=3001
SECRET_KEY_BASE=YOUR_GENERATED_SECRET # Generate with `mix phx.gen.secret`
MIX_ENV=prod
```

### 5. Running the Service

It is recommended to run Blockscout as a systemd service.

**Startup Script (`run_blockscout.sh`):**
```bash
#!/bin/bash
export PATH=$PATH:$HOME/.asdf/shims
cd /opt/blockscout-backend
if [ -f "prod.env" ]; then
  export $(grep -v '^#' prod.env | xargs)
fi
# Run migrations before starting
mix ecto.migrate
exec mix phx.server
```

**Systemd Unit (`/etc/systemd/system/blockscout-backend.service`):**
```ini
[Unit]
Description=Blockscout Backend
After=network.target postgresql.service redis-server.service

[Service]
User=root
WorkingDirectory=/opt/blockscout-backend
ExecStart=/opt/blockscout-backend/run_blockscout.sh
Restart=always
MemoryMax=16G

[Install]
WantedBy=multi-user.target
```

## 🛠 Development

To run the explorer locally for development:

1.  **Configure DB**: Update `config/dev.exs` with your local DB credentials.
2.  **Install Deps**: `mix deps.get`
3.  **Setup DB**: `mix do ecto.create, ecto.migrate`
4.  **Install Node Deps**:
    *   `cd apps/block_scout_web/assets && npm install`
5.  **Start Server**: `mix phx.server`

The explorer will be available at `http://localhost:4000`.

## 🔍 Troubleshooting

*   **`FunctionClauseError` in Indexer**: Ensure you are using this W Chain fork which handles the `txType` field.
*   **Missing Assets**: Run `mix phx.digest` if CSS/JS is missing in production.
*   **Slow Indexing**: Increase `INDEXER_..._CONCURRENCY` values in `prod.env`.
*   **Database Connection**: Check `DATABASE_URL` matches your Postgres credentials.
