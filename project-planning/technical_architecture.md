# Trading Portfolio Tracker - Technical Architecture

## System Architecture Overview

### Architectural Principles

The Trading Portfolio Tracker follows modern distributed systems principles with a focus on:

- **Event-Driven Architecture**: Loose coupling through asynchronous event communication
- **Microservices Design**: Domain-driven service boundaries with independent deployability
- **CQRS Pattern**: Separation of command and query responsibilities for optimal performance
- **Event Sourcing**: Complete audit trail with event replay capabilities
- **Reactive Programming**: Real-time data processing and user experience

### High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Layer                             │
├─────────────────────────────────────────────────────────────┤
│  Next.js Frontend (TypeScript)                            │
│  - React Components                                        │
│  - Real-time WebSocket Client                             │
│  - Supabase Client Integration                            │
│  - State Management (Zustand/Redux)                       │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                 API Gateway Layer                           │
├─────────────────────────────────────────────────────────────┤
│  Nginx / Spring Cloud Gateway                             │
│  - Load Balancing                                         │
│  - Rate Limiting                                          │
│  - Authentication Validation                              │
│  - Request Routing                                        │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│               Service Mesh Layer                            │
├─────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐ ┌─────────────┐ ┌──────────────────┐    │
│  │  Market Data │ │ Portfolio   │ │   Analytics      │    │
│  │   Service    │ │  Service    │ │    Service       │    │
│  │              │ │             │ │                  │    │
│  │ - Finnhub    │ │ - CRUD Ops  │ │ - Calculations   │    │
│  │ - WebSocket  │ │ - P&L Calc  │ │ - Risk Analysis  │    │
│  │ - Kafka Prod │ │ - Supabase  │ │ - Stream Process │    │
│  └──────────────┘ └─────────────┘ └──────────────────┘    │
│                                                            │
│  ┌──────────────┐ ┌─────────────┐                        │
│  │ Notification │ │    User     │                        │
│  │   Service    │ │  Service    │                        │
│  │              │ │             │                        │
│  │ - Alerts     │ │ - Auth      │                        │
│  │ - Email/SMS  │ │ - Profiles  │                        │
│  │ - WebSocket  │ │ - Supabase  │                        │
│  └──────────────┘ └─────────────┘                        │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│              Event Streaming Layer                          │
├─────────────────────────────────────────────────────────────┤
│  Apache Kafka Cluster                                     │
│                                                            │
│  Topics:                                                   │
│  - market-data-updates                                     │
│  - portfolio-events                                        │
│  - user-events                                            │
│  - notification-events                                     │
│  - analytics-events                                        │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                Data Layer                                   │
├─────────────────────────────────────────────────────────────┤
│  ┌───────────┐ ┌──────────┐ ┌─────────┐ ┌──────────────┐   │
│  │ Supabase  │ │ MongoDB  │ │  Redis  │ │   InfluxDB   │   │
│  │PostgreSQL │ │          │ │         │ │              │   │
│  │           │ │ Event    │ │ Cache   │ │  Time-Series │   │
│  │- Users    │ │ Store    │ │ Session │ │  Market Data │   │
│  │- Portfolios│ │         │ │ Real-time│ │              │   │
│  │- Positions│ │         │ │         │ │              │   │
│  └───────────┘ └──────────┘ └─────────┘ └──────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Microservices Detailed Design

### 1. Market Data Service

**Responsibilities:**
- External API integration (Finnhub, Alpha Vantage, Polygon)
- Real-time price feed normalization
- Rate limiting and API management
- Market data event publishing

**Technical Stack:**
- **Runtime**: Node.js with TypeScript
- **Framework**: Express.js
- **Kafka**: KafkaJS for event publishing
- **Scheduling**: node-cron for periodic data fetching
- **HTTP Client**: Axios with retry logic

**Key Components:**
```typescript
interface MarketDataEvent {
  symbol: string;
  price: number;
  volume: number;
  timestamp: Date;
  exchange: string;
  source: 'finnhub' | 'alpha-vantage' | 'polygon';
}

class MarketDataService {
  private kafka: Kafka;
  private apiClients: Map<string, ApiClient>;
  
  async fetchAndPublishMarketData(): Promise<void>;
  async publishPriceUpdate(event: MarketDataEvent): Promise<void>;
  async handleRateLimit(source: string): Promise<void>;
}
```

**Kafka Topics Produced:**
- `market-data-updates`: Real-time price updates
- `market-data-errors`: API failures and rate limiting events

**API Integrations:**
- **Finnhub WebSocket**: Real-time price feeds
- **Alpha Vantage REST**: Historical data and technical indicators
- **Polygon REST**: Company fundamentals and news

### 2. Portfolio Service

**Responsibilities:**
- Portfolio and position management
- Transaction processing and P&L calculations
- Event sourcing implementation
- Integration with Supabase for persistent storage

**Technical Stack:**
- **Runtime**: Node.js with TypeScript
- **Framework**: Express.js
- **Database**: Supabase PostgreSQL + MongoDB (Event Store)
- **Kafka**: Event consumption and production
- **Validation**: Joi or Yup for request validation

**Key Components:**
```typescript
interface PortfolioAggregate {
  id: string;
  userId: string;
  name: string;
  positions: Position[];
  totalValue: number;
  version: number;
}

interface PortfolioEvent {
  aggregateId: string;
  eventType: string;
  eventData: any;
  timestamp: Date;
  version: number;
}

class PortfolioService {
  async createPortfolio(command: CreatePortfolioCommand): Promise<void>;
  async addPosition(command: AddPositionCommand): Promise<void>;
  async calculatePortfolioValue(portfolioId: string): Promise<number>;
  async handleMarketDataUpdate(event: MarketDataEvent): Promise<void>;
}
```

**Event Sourcing Schema:**
- **Event Store**: MongoDB collection storing all portfolio events
- **Snapshots**: Periodic portfolio state snapshots for performance
- **Projections**: Read models in Supabase for queries

**Kafka Topics:**
- **Consumed**: `market-data-updates`
- **Produced**: `portfolio-events`, `portfolio-valuations`

### 3. Analytics Service

**Responsibilities:**
- Real-time portfolio analytics calculation
- Risk assessment and diversification analysis
- Performance benchmarking
- Historical trend analysis

**Technical Stack:**
- **Runtime**: Node.js with TypeScript
- **Stream Processing**: Kafka Streams equivalent (KafkaJS with manual state management)
- **Calculations**: Math.js for financial computations
- **Time-Series**: InfluxDB for historical data storage

**Key Components:**
```typescript
interface AnalyticsMetrics {
  portfolioId: string;
  totalReturn: number;
  sharpeRatio: number;
  beta: number;
  volatility: number;
  diversificationScore: number;
  riskMetrics: RiskMetrics;
}

class AnalyticsService {
  async calculatePerformanceMetrics(portfolioId: string): Promise<AnalyticsMetrics>;
  async assessRisk(portfolio: Portfolio): Promise<RiskAssessment>;
  async generateRebalancingRecommendations(portfolioId: string): Promise<RebalancingPlan>;
}
```

**Stream Processing Windows:**
- **1-minute windows**: Real-time portfolio valuation
- **Daily windows**: Performance calculations
- **Weekly windows**: Risk assessment updates

### 4. Notification Service

**Responsibilities:**
- Alert rule engine processing
- Multi-channel notification delivery (email, SMS, WebSocket)
- User preference management
- Notification history and tracking

**Technical Stack:**
- **Runtime**: Node.js with TypeScript
- **Email**: SendGrid or SMTP
- **WebSocket**: Socket.io for real-time browser notifications
- **Queue**: Redis for notification queuing
- **Templates**: Handlebars for notification templates

**Key Components:**
```typescript
interface NotificationRule {
  id: string;
  userId: string;
  type: 'price_alert' | 'risk_alert' | 'rebalance_alert';
  conditions: AlertCondition[];
  channels: NotificationChannel[];
  active: boolean;
}

class NotificationService {
  async evaluateAlertRules(event: PortfolioEvent): Promise<void>;
  async sendNotification(notification: Notification): Promise<void>;
  async manageUserPreferences(userId: string, preferences: NotificationPreferences): Promise<void>;
}
```

## Data Architecture

### Database Design Decisions

**Supabase PostgreSQL (Primary Database):**
```sql
-- Users table (managed by Supabase Auth)
CREATE TABLE profiles (
  id UUID REFERENCES auth.users(id) PRIMARY KEY,
  username TEXT UNIQUE,
  full_name TEXT,
  avatar_url TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Portfolios table
CREATE TABLE portfolios (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES profiles(id) NOT NULL,
  name TEXT NOT NULL,
  description TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Positions table (CQRS Read Model)
CREATE TABLE positions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  portfolio_id UUID REFERENCES portfolios(id) NOT NULL,
  symbol TEXT NOT NULL,
  quantity DECIMAL NOT NULL,
  average_price DECIMAL NOT NULL,
  current_price DECIMAL,
  market_value DECIMAL,
  unrealized_pnl DECIMAL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Transactions table
CREATE TABLE transactions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  portfolio_id UUID REFERENCES portfolios(id) NOT NULL,
  symbol TEXT NOT NULL,
  transaction_type TEXT NOT NULL CHECK (transaction_type IN ('BUY', 'SELL')),
  quantity DECIMAL NOT NULL,
  price DECIMAL NOT NULL,
  fees DECIMAL DEFAULT 0,
  executed_at TIMESTAMP WITH TIME ZONE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**MongoDB (Event Store):**
```javascript
// Portfolio Events Collection
{
  _id: ObjectId,
  aggregateId: "portfolio-uuid",
  aggregateType: "Portfolio",
  eventType: "PositionAdded",
  eventData: {
    symbol: "AAPL",
    quantity: 100,
    price: 150.00
  },
  eventVersion: 1,
  timestamp: ISODate,
  userId: "user-uuid"
}

// Event Stream Indexes
db.portfolio_events.createIndex({ "aggregateId": 1, "eventVersion": 1 });
db.portfolio_events.createIndex({ "timestamp": 1 });
db.portfolio_events.createIndex({ "userId": 1, "timestamp": 1 });
```

**Redis (Cache & Sessions):**
```
# Portfolio value cache
portfolio:value:${portfolioId} -> JSON serialized portfolio value
TTL: 300 seconds (5 minutes)

# Market data cache
market:price:${symbol} -> price value
TTL: 60 seconds

# User sessions
session:${sessionId} -> JSON serialized session data
TTL: 24 hours

# Rate limiting
rate_limit:${apiSource}:${minute} -> request count
TTL: 60 seconds
```

## Event-Driven Communication Patterns

### Kafka Topic Design

**Topic Naming Convention:**
`{domain}.{entity}.{event-type}`

**Topic Configuration:**
```yaml
market-data.prices.updated:
  partitions: 12  # Partitioned by symbol hash
  replication-factor: 3
  retention.ms: 86400000  # 24 hours

portfolio.aggregates.events:
  partitions: 6   # Partitioned by portfolio ID
  replication-factor: 3
  retention.ms: 2592000000  # 30 days

analytics.calculations.completed:
  partitions: 3
  replication-factor: 3
  retention.ms: 604800000  # 7 days

notifications.alerts.triggered:
  partitions: 3
  replication-factor: 3
  retention.ms: 604800000  # 7 days
```

### Event Schema Design

**Base Event Schema:**
```typescript
interface BaseEvent {
  eventId: string;
  eventType: string;
  aggregateId: string;
  aggregateType: string;
  eventVersion: number;
  timestamp: Date;
  userId?: string;
  correlationId?: string;
}

interface MarketDataUpdatedEvent extends BaseEvent {
  eventType: 'MarketDataUpdated';
  eventData: {
    symbol: string;
    price: number;
    volume: number;
    timestamp: Date;
    source: string;
  };
}

interface PortfolioValueCalculatedEvent extends BaseEvent {
  eventType: 'PortfolioValueCalculated';
  eventData: {
    portfolioId: string;
    totalValue: number;
    positions: PositionValue[];
    calculatedAt: Date;
  };
}
```

## Security Architecture

### Authentication & Authorization

**Supabase Auth Integration:**
- JWT token-based authentication
- Row Level Security (RLS) for data access
- Social login integration (Google, GitHub)
- Multi-factor authentication support

**API Security:**
```typescript
// JWT Middleware
async function authenticateToken(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Access token required' });
  
  try {
    const { data: user } = await supabase.auth.getUser(token);
    req.user = user;
    next();
  } catch (error) {
    return res.status(403).json({ error: 'Invalid token' });
  }
}
```

**Database Security:**
```sql
-- Row Level Security Policies
ALTER TABLE portfolios ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can only access their own portfolios"
ON portfolios FOR ALL
USING (user_id = auth.uid());

CREATE POLICY "Users can only access their own positions"
ON positions FOR ALL
USING (
  portfolio_id IN (
    SELECT id FROM portfolios WHERE user_id = auth.uid()
  )
);
```

## Performance Architecture

### Caching Strategy

**Multi-Level Caching:**
1. **Browser Cache**: Static assets and API responses
2. **Redis Cache**: Frequently accessed data (portfolio values, market prices)
3. **Database Query Cache**: Supabase automatic query caching
4. **CDN Cache**: Static assets and API responses

**Cache Invalidation Patterns:**
```typescript
class CacheManager {
  async invalidatePortfolioCache(portfolioId: string): Promise<void> {
    await Promise.all([
      this.redis.del(`portfolio:value:${portfolioId}`),
      this.redis.del(`portfolio:analytics:${portfolioId}`),
      this.redis.del(`portfolio:positions:${portfolioId}`)
    ]);
  }
  
  async updateMarketDataCache(symbol: string, price: number): Promise<void> {
    await this.redis.setex(`market:price:${symbol}`, 60, price.toString());
  }
}
```

### Real-Time Performance Optimization

**WebSocket Connection Management:**
```typescript
class WebSocketManager {
  private connections: Map<string, WebSocket> = new Map();
  private subscriptions: Map<string, Set<string>> = new Map();
  
  subscribeToSymbol(userId: string, symbol: string): void {
    // Efficient subscription management to minimize external API calls
  }
  
  broadcastPriceUpdate(symbol: string, price: number): void {
    // Broadcast only to subscribed users
  }
}
```

## Monitoring & Observability

### Metrics Collection

**Business Metrics:**
- Portfolio value changes
- Transaction volumes
- User engagement rates
- API usage patterns

**Technical Metrics:**
- Response times by endpoint
- Kafka consumer lag
- Database connection pool utilization
- Cache hit/miss rates

**Custom Metrics Implementation:**
```typescript
import prometheus from 'prom-client';

const portfolioValueGauge = new prometheus.Gauge({
  name: 'portfolio_total_value',
  help: 'Total portfolio value by user',
  labelNames: ['userId', 'portfolioId']
});

const apiResponseTime = new prometheus.Histogram({
  name: 'api_response_duration_seconds',
  help: 'API response time',
  labelNames: ['method', 'endpoint', 'status']
});
```

### Distributed Tracing

**Jaeger Integration:**
```typescript
import { initTracer } from 'jaeger-client';

const tracer = initTracer({
  serviceName: 'portfolio-service',
  sampler: { type: 'const', param: 1 }
});

// Trace portfolio operations across services
function tracePortfolioOperation(operationName: string) {
  const span = tracer.startSpan(operationName);
  return {
    span,
    finish: () => span.finish()
  };
}
```

## Deployment Architecture

### Containerization Strategy

**Multi-Stage Docker Builds:**
```dockerfile
# Market Data Service Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY dist ./dist
EXPOSE 3001
CMD ["node", "dist/index.js"]
```

**Docker Compose Development:**
```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on: [zookeeper]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: portfolio_tracker
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password

  mongodb:
    image: mongo:6
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: password

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass password

  market-data-service:
    build: ./services/market-data-service
    environment:
      KAFKA_BROKERS: kafka:9092
      DATABASE_URL: mongodb://root:password@mongodb:27017
    depends_on: [kafka, mongodb]

  portfolio-service:
    build: ./services/portfolio-service
    environment:
      SUPABASE_URL: ${SUPABASE_URL}
      SUPABASE_ANON_KEY: ${SUPABASE_ANON_KEY}
      KAFKA_BROKERS: kafka:9092
    depends_on: [kafka]
```

## Technology Integration Points

### Next.js Frontend Integration

**Real-Time Data Flow:**
```typescript
// Real-time portfolio updates
import { createClientComponentClient } from '@supabase/auth-helpers-nextjs';
import { useWebSocket } from '@/lib/websocket';

export function useRealTimePortfolio(portfolioId: string) {
  const supabase = createClientComponentClient();
  const [portfolio, setPortfolio] = useState<Portfolio | null>(null);
  
  // Supabase real-time for database changes
  useEffect(() => {
    const channel = supabase
      .channel(`portfolio-${portfolioId}`)
      .on('postgres_changes', 
        { event: '*', schema: 'public', table: 'positions' },
        (payload) => handlePositionUpdate(payload)
      )
      .subscribe();
      
    return () => supabase.removeChannel(channel);
  }, [portfolioId]);
  
  // WebSocket for market data updates
  const marketData = useWebSocket('ws://localhost:8080/market-data', {
    onMessage: handleMarketDataUpdate
  });
  
  return { portfolio, marketData };
}
```

This technical architecture provides the foundation for a robust, scalable, and maintainable trading portfolio tracker that demonstrates senior-level engineering practices and modern distributed systems design patterns.