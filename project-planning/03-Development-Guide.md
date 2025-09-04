# Trading Portfolio Tracker - Development Guide

## Development Environment Setup

### Prerequisites

**System Requirements:**
- **Operating System**: macOS, Linux, or Windows 10/11 with WSL2
- **Memory**: Minimum 8GB RAM (16GB recommended for optimal performance)
- **Storage**: At least 10GB free space for Docker images and dependencies
- **CPU**: Multi-core processor (4+ cores recommended)

**Required Software:**
```bash
# Node.js (v18 or higher)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18

# Docker Desktop
# Download from: https://www.docker.com/products/docker-desktop/

# Git
# Download from: https://git-scm.com/downloads

# Visual Studio Code (recommended)
# Download from: https://code.visualstudio.com/
```

**VS Code Extensions (Recommended):**
```json
{
  "recommendations": [
    "ms-vscode.vscode-typescript-next",
    "ms-vscode.vscode-json",
    "bradlc.vscode-tailwindcss",
    "ms-vscode-remote.remote-containers",
    "ms-kubernetes-tools.vscode-kubernetes-tools",
    "humao.rest-client",
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "ms-vscode.vscode-docker"
  ]
}
```

### Project Initialization

**1. Clone and Setup Repository:**
```bash
# Create project directory
mkdir trading-portfolio-tracker
cd trading-portfolio-tracker

# Initialize Git repository
git init
git remote add origin <your-github-repo-url>

# Create workspace structure
mkdir -p services/{market-data-service,portfolio-service,analytics-service,notification-service}
mkdir -p frontend
mkdir -p shared/types
mkdir -p infrastructure/{kafka,monitoring,scripts}
mkdir -p docs
mkdir -p tests/{unit,integration,e2e}
```

**2. Root Package.json Setup:**
```json
{
  "name": "trading-portfolio-tracker",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "frontend",
    "services/*",
    "shared/*"
  ],
  "scripts": {
    "dev": "concurrently \"npm run dev:infrastructure\" \"npm run dev:services\" \"npm run dev:frontend\"",
    "dev:infrastructure": "docker-compose up -d zookeeper kafka mongodb redis postgres prometheus grafana",
    "dev:services": "concurrently \"npm run dev -w market-data-service\" \"npm run dev -w portfolio-service\" \"npm run dev -w analytics-service\" \"npm run dev -w notification-service\"",
    "dev:frontend": "npm run dev -w frontend",
    "build": "npm run build -ws",
    "test": "npm run test -ws",
    "test:integration": "npm run test:integration -ws",
    "lint": "npm run lint -ws",
    "clean": "npm run clean -ws && docker-compose down -v",
    "setup": "npm install && npm run setup -ws"
  },
  "devDependencies": {
    "concurrently": "^8.2.2",
    "husky": "^8.0.3",
    "lint-staged": "^15.0.2"
  }
}
```

**3. Docker Compose Configuration:**
```yaml
# docker-compose.yml
version: '3.8'

services:
  # Infrastructure Services
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data
      - zookeeper_logs:/var/lib/zookeeper/log

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
    volumes:
      - kafka_data:/var/lib/kafka/data

  mongodb:
    image: mongo:6.0
    container_name: mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: password
      MONGO_INITDB_DATABASE: portfolio_events
    volumes:
      - mongodb_data:/data/db
      - ./infrastructure/mongodb/init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro

  postgres:
    image: postgres:15
    container_name: postgres
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: portfolio_tracker
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./infrastructure/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: redis-server --requirepass password
    volumes:
      - redis_data:/data

  # Monitoring Services
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./infrastructure/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./infrastructure/monitoring/grafana/provisioning:/etc/grafana/provisioning
      - ./infrastructure/monitoring/grafana/dashboards:/var/lib/grafana/dashboards

  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    restart: unless-stopped
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      COLLECTOR_OTLP_ENABLED: true

volumes:
  zookeeper_data:
  zookeeper_logs:
  kafka_data:
  mongodb_data:
  postgres_data:
  redis_data:
  prometheus_data:
  grafana_data:

networks:
  default:
    name: portfolio_tracker_network
```

## Service Development Setup

### 1. Market Data Service

**Directory Structure:**
```
services/market-data-service/
├── src/
│   ├── config/
│   │   ├── database.ts
│   │   ├── kafka.ts
│   │   └── api-clients.ts
│   ├── controllers/
│   │   └── market-data.controller.ts
│   ├── services/
│   │   ├── market-data.service.ts
│   │   ├── finnhub.service.ts
│   │   └── alpha-vantage.service.ts
│   ├── kafka/
│   │   ├── producers/
│   │   └── consumers/
│   ├── types/
│   │   └── market-data.types.ts
│   ├── utils/
│   │   ├── logger.ts
│   │   └── rate-limiter.ts
│   ├── middleware/
│   │   ├── auth.middleware.ts
│   │   └── error.middleware.ts
│   └── app.ts
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── package.json
├── tsconfig.json
├── .env.example
├── Dockerfile
└── README.md
```

**Package.json:**
```json
{
  "name": "market-data-service",
  "version": "1.0.0",
  "main": "dist/app.js",
  "scripts": {
    "dev": "tsx watch src/app.ts",
    "build": "tsc",
    "start": "node dist/app.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:integration": "jest --config jest.integration.config.js",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "clean": "rm -rf dist"
  },
  "dependencies": {
    "express": "^4.18.2",
    "kafkajs": "^2.2.4",
    "axios": "^1.6.0",
    "dotenv": "^16.3.1",
    "cors": "^2.8.5",
    "helmet": "^7.1.0",
    "compression": "^1.7.4",
    "winston": "^3.11.0",
    "joi": "^17.11.0",
    "ioredis": "^5.3.2",
    "node-cron": "^3.0.3"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/cors": "^2.8.16",
    "@types/compression": "^1.7.5",
    "@types/node": "^20.8.0",
    "@types/node-cron": "^3.0.11",
    "@typescript-eslint/eslint-plugin": "^6.9.0",
    "@typescript-eslint/parser": "^6.9.0",
    "eslint": "^8.52.0",
    "jest": "^29.7.0",
    "@types/jest": "^29.5.6",
    "ts-jest": "^29.1.1",
    "tsx": "^3.14.0",
    "typescript": "^5.2.2",
    "supertest": "^6.3.3",
    "@types/supertest": "^2.0.15"
  }
}
```

**TypeScript Configuration:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "commonjs",
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "moduleResolution": "node",
    "baseUrl": "./",
    "paths": {
      "@/*": ["src/*"],
      "@/types": ["../shared/types/*"]
    },
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*", "../shared/types/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### 2. Frontend Next.js Setup

**Directory Structure:**
```
frontend/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   └── register/
│   ├── dashboard/
│   │   ├── page.tsx
│   │   └── loading.tsx
│   ├── portfolio/
│   │   ├── [id]/
│   │   └── new/
│   ├── analytics/
│   ├── settings/
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx
├── components/
│   ├── ui/
│   ├── charts/
│   ├── forms/
│   └── layout/
├── lib/
│   ├── supabase.ts
│   ├── websocket.ts
│   ├── api.ts
│   └── utils.ts
├── hooks/
│   ├── use-portfolio.ts
│   ├── use-market-data.ts
│   └── use-auth.ts
├── types/
├── stores/
│   ├── portfolio.store.ts
│   └── auth.store.ts
├── public/
├── package.json
├── next.config.js
├── tailwind.config.js
└── tsconfig.json
```

**Package.json:**
```json
{
  "name": "frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "test": "jest",
    "test:e2e": "playwright test"
  },
  "dependencies": {
    "next": "14.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@supabase/auth-helpers-nextjs": "^0.8.7",
    "@supabase/supabase-js": "^2.38.4",
    "zustand": "^4.4.6",
    "@tanstack/react-query": "^5.8.4",
    "recharts": "^2.8.0",
    "tailwindcss": "^3.3.5",
    "@headlessui/react": "^1.7.17",
    "@heroicons/react": "^2.0.18",
    "clsx": "^2.0.0",
    "date-fns": "^2.30.0",
    "react-hot-toast": "^2.4.1",
    "socket.io-client": "^4.7.4"
  },
  "devDependencies": {
    "@types/node": "^20.8.0",
    "@types/react": "^18.2.31",
    "@types/react-dom": "^18.2.14",
    "typescript": "^5.2.2",
    "eslint": "^8.52.0",
    "eslint-config-next": "14.0.0",
    "@typescript-eslint/eslint-plugin": "^6.9.0",
    "@typescript-eslint/parser": "^6.9.0",
    "autoprefixer": "^10.4.16",
    "postcss": "^8.4.31",
    "jest": "^29.7.0",
    "@testing-library/react": "^13.4.0",
    "@testing-library/jest-dom": "^6.1.4",
    "jest-environment-jsdom": "^29.7.0",
    "@playwright/test": "^1.40.0"
  }
}
```

## Development Workflow

### Daily Development Process

**1. Start Development Environment:**
```bash
# Terminal 1: Start infrastructure
npm run dev:infrastructure

# Wait for services to be ready (check Docker Desktop)
# Terminal 2: Start all services
npm run dev
```

**2. Service Development Workflow:**
```bash
# Work on specific service
cd services/market-data-service

# Install dependencies
npm install

# Start in watch mode
npm run dev

# Run tests
npm test

# Run integration tests
npm run test:integration
```

**3. Frontend Development:**
```bash
cd frontend

# Install dependencies
npm install

# Start development server
npm run dev

# Run type checking
npm run type-check

# Run tests
npm test
```

### Code Quality Standards

**ESLint Configuration (.eslintrc.js):**
```javascript
module.exports = {
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaVersion: 2022,
    sourceType: 'module',
    project: './tsconfig.json',
  },
  plugins: ['@typescript-eslint'],
  extends: [
    'eslint:recommended',
    '@typescript-eslint/recommended',
    '@typescript-eslint/recommended-requiring-type-checking',
  ],
  rules: {
    '@typescript-eslint/no-unused-vars': 'error',
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/explicit-function-return-type': 'warn',
    '@typescript-eslint/no-floating-promises': 'error',
    'prefer-const': 'error',
    'no-var': 'error',
  },
};
```

**Prettier Configuration (.prettierrc):**
```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false
}
```

**Husky Pre-commit Hooks:**
```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write",
      "git add"
    ],
    "*.{json,md}": [
      "prettier --write",
      "git add"
    ]
  }
}
```

### Testing Strategy

**Jest Configuration (jest.config.js):**
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: [
    '<rootDir>/tests/**/*.test.ts',
    '<rootDir>/src/**/*.test.ts'
  ],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts',
    '!src/types/**/*'
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts']
};
```

**Unit Test Example:**
```typescript
// tests/unit/market-data.service.test.ts
import { MarketDataService } from '../../src/services/market-data.service';
import { FinnhubService } from '../../src/services/finnhub.service';

jest.mock('../../src/services/finnhub.service');

describe('MarketDataService', () => {
  let marketDataService: MarketDataService;
  let mockFinnhubService: jest.Mocked<FinnhubService>;

  beforeEach(() => {
    mockFinnhubService = new FinnhubService() as jest.Mocked<FinnhubService>;
    marketDataService = new MarketDataService(mockFinnhubService);
  });

  describe('fetchStockPrice', () => {
    it('should return stock price for valid symbol', async () => {
      // Arrange
      const symbol = 'AAPL';
      const expectedPrice = 150.00;
      mockFinnhubService.getQuote.mockResolvedValue({
        c: expectedPrice,
        h: 152.00,
        l: 149.00,
        o: 151.00,
        pc: 149.50,
        t: Date.now()
      });

      // Act
      const result = await marketDataService.fetchStockPrice(symbol);

      // Assert
      expect(result).toBe(expectedPrice);
      expect(mockFinnhubService.getQuote).toHaveBeenCalledWith(symbol);
    });

    it('should throw error for invalid symbol', async () => {
      // Arrange
      const symbol = 'INVALID';
      mockFinnhubService.getQuote.mockRejectedValue(new Error('Invalid symbol'));

      // Act & Assert
      await expect(marketDataService.fetchStockPrice(symbol))
        .rejects.toThrow('Invalid symbol');
    });
  });
});
```

**Integration Test Example:**
```typescript
// tests/integration/market-data.integration.test.ts
import request from 'supertest';
import { app } from '../../src/app';
import { testContainer } from '../fixtures/test-container';

describe('Market Data API Integration', () => {
  beforeAll(async () => {
    await testContainer.start();
  });

  afterAll(async () => {
    await testContainer.stop();
  });

  describe('GET /api/market-data/:symbol', () => {
    it('should return market data for valid symbol', async () => {
      const response = await request(app)
        .get('/api/market-data/AAPL')
        .expect(200);

      expect(response.body).toHaveProperty('symbol', 'AAPL');
      expect(response.body).toHaveProperty('price');
      expect(response.body).toHaveProperty('timestamp');
    });

    it('should return 404 for invalid symbol', async () => {
      await request(app)
        .get('/api/market-data/INVALID')
        .expect(404);
    });
  });
});
```

### Environment Configuration

**Environment Variables (.env.example):**
```bash
# Application
NODE_ENV=development
PORT=3001
LOG_LEVEL=debug

# External APIs
FINNHUB_API_KEY=your_finnhub_api_key
ALPHA_VANTAGE_API_KEY=your_alpha_vantage_api_key
POLYGON_API_KEY=your_polygon_api_key

# Kafka
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=market-data-service

# Database
MONGODB_URL=mongodb://root:password@localhost:27017/portfolio_events?authSource=admin
REDIS_URL=redis://:password@localhost:6379

# Supabase
SUPABASE_URL=your_supabase_project_url
SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_supabase_service_role_key

# Monitoring
JAEGER_ENDPOINT=http://localhost:14268/api/traces
PROMETHEUS_GATEWAY=http://localhost:9091
```

**Environment Validation:**
```typescript
// src/config/env.ts
import joi from 'joi';

const envSchema = joi.object({
  NODE_ENV: joi.string().valid('development', 'staging', 'production').default('development'),
  PORT: joi.number().default(3001),
  LOG_LEVEL: joi.string().valid('error', 'warn', 'info', 'debug').default('info'),
  
  FINNHUB_API_KEY: joi.string().required(),
  ALPHA_VANTAGE_API_KEY: joi.string().required(),
  
  KAFKA_BROKERS: joi.string().default('localhost:9092'),
  KAFKA_CLIENT_ID: joi.string().default('market-data-service'),
  
  MONGODB_URL: joi.string().required(),
  REDIS_URL: joi.string().required(),
  
  SUPABASE_URL: joi.string().uri().required(),
  SUPABASE_ANON_KEY: joi.string().required(),
});

const { error, value: env } = envSchema.validate(process.env);

if (error) {
  throw new Error(`Config validation error: ${error.message}`);
}

export default env;
```

## Debugging and Monitoring

### Local Development Monitoring

**Monitoring URLs:**
- **Grafana**: http://localhost:3000 (admin/admin)
- **Prometheus**: http://localhost:9090
- **Jaeger**: http://localhost:16686
- **Kafka UI**: http://localhost:9000 (if installed)

**Useful Development Commands:**
```bash
# View Kafka topics
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list

# View Kafka messages
docker exec -it kafka kafka-console-consumer --bootstrap-server localhost:9092 --topic market-data-updates --from-beginning

# MongoDB shell
docker exec -it mongodb mongosh -u root -p password

# Redis CLI
docker exec -it redis redis-cli -a password

# View logs
docker-compose logs -f market-data-service
docker-compose logs -f kafka
```

### Performance Profiling

**Node.js Performance Monitoring:**
```typescript
// src/utils/performance.ts
import { performance, PerformanceObserver } from 'perf_hooks';

export class PerformanceMonitor {
  private observer: PerformanceObserver;

  constructor() {
    this.observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        console.log(`${entry.name}: ${entry.duration}ms`);
      }
    });
    
    this.observer.observe({ entryTypes: ['measure'] });
  }

  static measure(name: string, fn: () => Promise<any>): Promise<any> {
    return new Promise((resolve, reject) => {
      performance.mark(`${name}-start`);
      
      fn()
        .then((result) => {
          performance.mark(`${name}-end`);
          performance.measure(name, `${name}-start`, `${name}-end`);
          resolve(result);
        })
        .catch(reject);
    });
  }
}
```

This development guide provides a comprehensive foundation for building the Trading Portfolio Tracker with proper development practices, testing strategies, and monitoring capabilities.