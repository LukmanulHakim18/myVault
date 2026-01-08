# [Service Name]

**Status**: [Active | Deprecated | In Development]  
**Owner Team**: [MRG | UPG]  
**Primary Contact**: [Name/Email]  
**Last Updated**: YYYY-MM-DD

## Overview
Brief description of what this service does and its purpose in the system.

## Responsibilities
- Primary responsibility 1
- Primary responsibility 2
- Primary responsibility 3

## Technical Details

### Tech Stack
- **Language**: Go x.xx
- **Framework**: [Framework name and version]
- **Database**: [Database type and version]
- **Message Queue**: [If applicable]

### Architecture
Describe the high-level architecture of the service.

### Key Dependencies
- External service 1
- External service 2
- Internal service 1

## API Specification

### REST Endpoints
```http
GET /api/v1/endpoint
POST /api/v1/endpoint
```

### gRPC Services
```protobuf
service ServiceName {
  rpc MethodName(Request) returns (Response);
}
```

### Events Published
- `event.name.v1`: Description

### Events Consumed
- `event.name.v1`: Description

## Configuration

### Environment Variables
```bash
SERVICE_PORT=8080
DATABASE_URL=postgresql://...
```

### Feature Flags
- `feature_name`: Description

## Deployment

### Infrastructure
- **CPU**: X cores
- **Memory**: X GB
- **Instances**: X replicas

### Monitoring
- **Metrics**: Link to Grafana dashboard
- **Logs**: Link to log aggregation
- **Alerts**: Link to alert configuration

## Development

### Prerequisites
- Go 1.xx+
- Docker
- PostgreSQL 14+

### Local Setup
```bash
# Clone repository
git clone [repo-url]

# Install dependencies
go mod download

# Run tests
go test ./...

# Run locally
go run main.go
```

### Testing Strategy
