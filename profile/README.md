# Source Cooperative: A Neutral, Non-Profit Data-Sharing Utility

Source Cooperative is a platform designed to facilitate data sharing and collaboration. It provides a robust infrastructure for managing repositories, user accounts, and data connections, enabling efficient and secure data exchange.

The project consists of two main components:
1. A web application (`source.coop`) that serves as the user interface and API for managing repositories and accounts.
2. A data proxy service (`data.source.coop`) that handles data access and storage operations.

## Repository Structure

```
.
├── data.source.coop/
│   ├── Dockerfile
│   ├── scripts/
│   │   ├── build-push.sh
│   │   ├── deploy.sh
│   │   ├── run.sh
│   │   └── tag-release.sh
│   └── src/
│       ├── apis/
│       ├── backends/
│       ├── main.rs
│       └── utils/
└── source.coop/
    ├── cli/
    │   └── src/
    │       ├── commands/
    │       └── index.ts
    ├── scripts/
    │   ├── bucket_policy.json
    │   ├── db_start.sh
    │   └── db_stop.sh
    └── src/
        ├── api/
        ├── components/
        ├── lib/
        ├── pages/
        └── utils/
```

Key Files:
- `data.source.coop/Dockerfile`: Defines the Docker image for the data proxy service.
- `source.coop/src/pages/api/`: Contains API route handlers for the web application.
- `source.coop/src/components/`: React components for the user interface.
- `source.coop/cli/src/index.ts`: Entry point for the command-line interface.

## Usage Instructions
For Usage instrcutions on below sections, please refer to correspoding readme file on source.coop and data.source.coop.

### Installation
### Running the Application
### Configuration
### Testing
### Deployment


## Data Flow

1. User interacts with the web application (source.coop)
2. Web application sends API requests to the backend (/api routes)
3. Backend processes requests, interacting with DynamoDB for data storage
4. For data access operations, the backend communicates with the data proxy service
5. Data proxy service interacts with the configured data storage (S3, Azure, etc.)
6. Results are returned to the web application and displayed to the user

```
User -> Web App -> API Routes -> DynamoDB
                -> Data Proxy -> Data Storage (S3, Azure)
```

## Infrastructure

The project uses the following AWS resources:

- DynamoDB:
  - accounts: Stores user and organization account information
  - repositories: Stores repository metadata
  - api-keys: Stores API keys for authentication
  - memberships: Stores membership information for accounts and repositories
  - data-connections: Stores data connection configurations

- S3:
  - source-cooperative: Main bucket for storing repository data

- ECS:
  - SourceCooperative-Prod cluster: Runs the data proxy service

- ECR:
  - source-data-proxy: Stores Docker images for the data proxy service
