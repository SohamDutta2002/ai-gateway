# Elasticsearch Integration Guide

This guide provides detailed instructions on how to integrate the MagicAPI Gateway with Elasticsearch for telemetry and metrics collection.

## Overview

The Elasticsearch plugin allows the MagicAPI Gateway to export request metrics and telemetry data to an Elasticsearch cluster. This enables powerful analytics, monitoring, and visualization capabilities for your AI API gateway.

## Features

- Automatic collection and export of request metrics
- Detailed tracking of latency, token usage, and errors
- Ability to analyze provider-specific metrics
- Support for secure connections with authentication
- Compatible with Kibana for visualization

## Requirements

- Running Elasticsearch cluster (version 7.x or 8.x recommended)
- Elasticsearch credentials (if authentication is enabled)
- MagicAPI Gateway version 0.1.7 or higher

## Configuration

### 1. Environment Variables

To enable and configure the Elasticsearch plugin, set the following environment variables:

#### Required Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `ENABLE_ELASTICSEARCH` | Enable the Elasticsearch plugin | `false` |
| `ELASTICSEARCH_URL` | URL to your Elasticsearch cluster | `http://localhost:9200` |

#### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `ELASTICSEARCH_USERNAME` | Username for Elasticsearch authentication | None |
| `ELASTICSEARCH_PASSWORD` | Password for Elasticsearch authentication | None |
| `ELASTICSEARCH_INDEX` | Index name to store metrics | `ai-gateway-metrics` |

### 2. Environment File (.env)

Create or update your `.env` file with these variables:

```
# Gateway Configuration
PORT=3000
HOST=127.0.0.1
RUST_LOG=info

# Telemetry Configuration
ENABLE_ELASTICSEARCH=true

# Elasticsearch Configuration
ELASTICSEARCH_URL=http://localhost:9200
ELASTICSEARCH_USERNAME=elastic
ELASTICSEARCH_PASSWORD=your_secure_password
ELASTICSEARCH_INDEX=ai-gateway-metrics
```

## Deployment Options

### 1. Running Locally

To run the MagicAPI Gateway locally with Elasticsearch integration:

1. Create the `.env` file as shown above
2. Start the gateway:

```bash
cargo run --release
```

### 2. Docker Deployment

For Docker-based deployments, update your `docker-compose.yml` file:

```yaml
version: '3.8'
services:
  gateway:
    image: noveum/noveum-ai-gateway:latest
    environment:
      - RUST_LOG=info
      - ENABLE_ELASTICSEARCH=true
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=your_secure_password
      - ELASTICSEARCH_INDEX=ai-gateway-metrics
    ports:
      - "3000:3000"
    depends_on:
      - elasticsearch

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - "ELASTIC_PASSWORD=your_secure_password"
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

volumes:
  es_data:
```

### 3. Kubernetes Deployment

For Kubernetes deployments, add these environment variables to your deployment manifest:

```yaml
env:
  - name: ENABLE_ELASTICSEARCH
    value: "true"
  - name: ELASTICSEARCH_URL
    value: "http://elasticsearch.monitoring.svc.cluster.local:9200"
  - name: ELASTICSEARCH_USERNAME
    valueFrom:
      secretKeyRef:
        name: elasticsearch-credentials
        key: username
  - name: ELASTICSEARCH_PASSWORD
    valueFrom:
      secretKeyRef:
        name: elasticsearch-credentials
        key: password
  - name: ELASTICSEARCH_INDEX
    value: "ai-gateway-metrics"
```

## Collected Metrics

The Elasticsearch plugin collects and exports the following metrics for each request in an OpenTelemetry compatible format:

```json
{
  "timestamp": "2025-03-05T16:03:20.123Z",
  "resource": {
    "service.name": "noveum_ai_gateway",
    "service.version": "1.0.0",
    "deployment.environment": "production"
  },
  "name": "ai_gateway_request_log",
  "attributes": {
    "id": "msg_29",
    "thread_id": "thread_29",
    "org_id": "org_123",
    "user_id": "user_456",
    "project_id": "proj_design",
    "provider": "azure",
    "model": "gpt-4-turbo",
    "request": {
      "model": "gpt-4-turbo",
      "messages": [
        {
          "role": "user",
          "content": "Sample request content..."
        }
      ],
      "temperature": 0.7
    },
    "response": {
      "id": "chatcmpl-az-gpt4-001",
      "choices": [
        {
          "message": {
            "role": "assistant",
            "content": "Sample response content..."
          },
          "finish_reason": "stop"
        }
      ],
      "usage": {
        "prompt_tokens": 48,
        "completion_tokens": 865,
        "total_tokens": 913
      }
    },
    "metadata": {
      "project_id": "proj_design",
      "project_name": "UX Design",
      "latency": 6250,
      "tokens": { "input": 48, "output": 865, "total": 913 },
      "cost": 0.0456,
      "status": "success",
      "path": "/v1/chat/completions",
      "method": "POST",
      "request_size": 193,
      "response_size": 52280,
      "provider_latency": 255,
      "status_code": 200,
      "provider_status_code": 0,
      "error_count": 0,
      "error_type": null,
      "provider_error_count": 0,
      "provider_error_type": null
    }
  }
}
```

### Fields Explanation

#### Top Level
- `timestamp`: Time when the request was processed (ISO 8601 format)
- `resource`: Information about the service/deployment
- `name`: Log name identifier
- `attributes`: Detailed request information

#### Resource
- `service.name`: The name of the service
- `service.version`: The version of the gateway
- `deployment.environment`: The deployment environment (production, development, etc.)

#### Attributes
- **Basic identifying fields**:
  - `id`: Unique message ID
  - `thread_id`: Unique thread ID for conversation
  - `org_id`: Organization identifier (from header)
  - `user_id`: User identifier (from header)
  - `project_id`: Project identifier (from header)
- **Provider/model details**:
  - `provider`: AI provider name
  - `model`: Model name
- **Request/Response objects**:
  - `request`: Complete request payload
  - `response`: Complete response payload
- **Metadata**:
  - `project_id`: Project identifier
  - `project_name`: Project name (if available)
  - `latency`: Total request processing time in milliseconds
  - `tokens`: Token usage information (input, output, total)
  - `cost`: Estimated cost of the request
  - `status`: Request status (success or error)
  - `path`: API endpoint path
  - `method`: HTTP method
  - `request_size`: Size of the request in bytes
  - `response_size`: Size of the response in bytes
  - `provider_latency`: Time spent waiting for the provider response
  - `status_code`: HTTP status code of the response
  - `provider_status_code`: Status code from the provider
  - `error_count`: Number of errors encountered
  - `error_type`: Type of error (if any)
  - `provider_error_count`: Number of provider errors
  - `provider_error_type`: Type of provider error (if any)

## Kibana Integration (Optional)

For enhanced visualization capabilities, you can integrate with Kibana:

1. Add Kibana to your `docker-compose.yml`:

```yaml
kibana:
  image: docker.elastic.co/kibana/kibana:8.12.0
  environment:
    - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    - ELASTICSEARCH_USERNAME=elastic
    - ELASTICSEARCH_PASSWORD=your_secure_password
  ports:
    - "5601:5601"
  depends_on:
    - elasticsearch
```

2. Create visualizations and dashboards in Kibana to monitor:
   - Request volume over time
   - Average latency by provider and model
   - Token usage patterns
   - Error rates
   - Cost analysis

## Data Retention Policy (Optional)

For production environments, it's recommended to set up an index lifecycle policy to manage data retention:

```bash
curl -X PUT "localhost:9200/_ilm/policy/ai-gateway-metrics-policy" \
  -u elastic:your_secure_password \
  -H 'Content-Type: application/json' \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "actions": {}
        },
        "delete": {
          "min_age": "30d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'
```

Then apply the policy to your index template:

```bash
curl -X PUT "localhost:9200/_index_template/ai-gateway-metrics-template" \
  -u elastic:your_secure_password \
  -H 'Content-Type: application/json' \
  -d '{
    "index_patterns": ["ai-gateway-metrics*"],
    "template": {
      "settings": {
        "index.lifecycle.name": "ai-gateway-metrics-policy"
      }
    }
  }'
```

## Verifying the Setup

To verify that the Elasticsearch plugin is working correctly:

1. Start the gateway with Elasticsearch enabled
2. Make a few test requests to the gateway
3. Check the logs for successful Elasticsearch exports
4. Query Elasticsearch to confirm metrics are being stored:

```bash
curl -X GET "localhost:9200/ai-gateway-metrics/_search?pretty" \
  -u elastic:your_secure_password \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "match_all": {}
    },
    "size": 5,
    "sort": [
      {
        "timestamp": {
          "order": "desc"
        }
      }
    ]
  }'
```

## Custom Index Mapping (Optional)

For better performance and analysis, you can create a custom index mapping:

```bash
curl -X PUT "localhost:9200/ai-gateway-metrics" \
  -u elastic:your_secure_password \
  -H 'Content-Type: application/json' \
  -d '{
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "provider": { "type": "keyword" },
        "model": { "type": "keyword" },
        "path": { "type": "keyword" },
        "method": { "type": "keyword" },
        "total_latency_ms": { "type": "long" },
        "provider_latency_ms": { "type": "long" },
        "request_size": { "type": "long" },
        "response_size": { "type": "long" },
        "input_tokens": { "type": "integer" },
        "output_tokens": { "type": "integer" },
        "total_tokens": { "type": "integer" },
        "status_code": { "type": "short" },
        "provider_status_code": { "type": "short" },
        "error_count": { "type": "short" },
        "error_type": { "type": "keyword" },
        "provider_error_count": { "type": "short" },
        "provider_error_type": { "type": "keyword" },
        "cost": { "type": "float" }
      }
    }
  }'
```

## Troubleshooting

### Connection Issues

If you're having trouble connecting to Elasticsearch:

1. Verify Elasticsearch is running:
   ```bash
   curl -X GET "localhost:9200/_cluster/health?pretty"
   ```

2. Check network connectivity between the Gateway and Elasticsearch
3. Ensure correct URL and port are configured
4. Check for any firewall rules blocking connections

### Authentication Issues

If authentication fails:

1. Verify that your username and password are correct
2. Check if Elasticsearch has authentication enabled:
   ```bash
   curl -X GET "localhost:9200/_security/enabled" -u elastic:password
   ```
3. Ensure any necessary roles and permissions are configured

### Index Issues

If index creation or writing fails:

1. Check if the user has permissions to create/write to indices
2. Verify available disk space on the Elasticsearch nodes
3. Check Elasticsearch logs for any index-related errors

### Performance Considerations

For high-volume deployments:

1. Consider using index aliases and date-based indices (e.g., ai-gateway-metrics-YYYY-MM-DD)
2. Adjust refresh interval for better indexing performance
3. Configure appropriate shard and replica settings

## Further Resources

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- [MagicAPI Gateway Documentation](https://github.com/MagicAPI/ai-gateway/docs) 