# Claude AI Assistant Configuration

This file configures how Claude should assist with this AWS-focused TypeScript/Node.js project.

## Project Context

### Technology Stack
- **Primary Language**: TypeScript
- **Runtime**: Node.js 22.x
- **Cloud Provider**: AWS
- **Infrastructure as Code**: AWS CDK v2
- **Frontend Framework**: React 18.x
- **Package Manager**: npm
- **Testing**: Jest, AWS SDK Mock, React Testing Library

### AWS Services Used
- Lambda (Node.js runtime)
- API Gateway (REST and HTTP APIs)
- DynamoDB
- S3
- CloudFormation (via CDK)
- CloudWatch
- IAM
- EventBridge
- SQS/SNS
- CloudFront (for React hosting)
- Cognito (for authentication)

## Development Guidelines

### Code Style
- Use TypeScript with strict mode enabled
- Follow AWS best practices for Lambda functions
- Implement proper error handling and logging
- Use async/await over callbacks
- Prefer functional programming patterns where appropriate

### AWS-Specific Practices
- Always use least-privilege IAM policies
- Implement proper error handling for AWS SDK calls
- Use environment variables for configuration
- Follow AWS Well-Architected Framework principles
- Implement proper logging with structured logs for CloudWatch

### Project Structure
```
project-root/
├── src/
│   ├── handlers/          # Lambda function handlers
│   ├── services/          # Business logic
│   ├── utils/             # Shared utilities
│   └── types/             # TypeScript type definitions
├── infrastructure/        # CDK infrastructure code
│   ├── stacks/
│   └── constructs/
├── tests/
│   ├── unit/
│   └── integration/
└── scripts/              # Deployment and utility scripts
```

## Available Agents

When working on this project, consider using these specialized agents from the awesome-claude-agents repository:

### Core Agents
- **Code Reviewer**: For reviewing TypeScript/Node.js code
- **Performance Optimizer**: For optimizing Lambda functions and API responses

### AWS Specialized Agents
- **AWS CDK Architect**: For infrastructure design and CDK patterns
- **Node.js Lambda Architect**: For serverless function design

### React Specialized Agents
- **React Component Architect**: For component design and architecture
- **React Next.js Expert**: For Next.js full-stack applications
- **React State Manager**: For state management solutions

### Orchestrators
- **AWS Solutions Architect**: For overall AWS architecture decisions
- **Tech Lead Orchestrator**: For coordinating complex features

## Task-Specific Instructions

### When Creating Lambda Functions
1. Always include proper error handling
2. Use AWS X-Ray tracing
3. Implement structured logging
4. Keep functions small and focused
5. Use Lambda Layers for shared dependencies

### When Working with CDK
1. Use L2 constructs when available
2. Implement proper tagging strategy
3. Use parameter store for configuration
4. Follow CDK best practices for testing

### When Implementing APIs
1. Use API Gateway request validation
2. Implement proper CORS configuration
3. Use Lambda proxy integration
4. Implement rate limiting and throttling
5. Use custom authorizers for authentication

## Testing Requirements

### Unit Tests
- Minimum 80% code coverage
- Mock all AWS service calls
- Test error scenarios
- Use Jest snapshots for response validation

### Integration Tests
- Test actual AWS service interactions
- Use test environments with proper isolation
- Clean up resources after tests

## Security Considerations

1. Never hardcode credentials
2. Use AWS Secrets Manager for sensitive data
3. Implement proper input validation
4. Use AWS WAF for API protection
5. Enable CloudTrail for audit logging
6. Encrypt data at rest and in transit

## Performance Goals

- Lambda cold start < 1 second
- API response time < 200ms (p99)
- DynamoDB queries optimized for single-digit millisecond latency
- Use caching where appropriate (CloudFront, ElastiCache)

## Documentation Standards

- Document all public APIs with OpenAPI/Swagger
- Include JSDoc comments for all functions
- Maintain README files for each service
- Document infrastructure decisions in ADRs

## Deployment Process

1. Run unit tests
2. Run integration tests in dev environment
3. Deploy to staging using CDK
4. Run smoke tests
5. Deploy to production with approval
6. Monitor CloudWatch metrics and alarms

## Helper Commands

```bash
# Local development
npm run dev

# Run tests
npm test
npm run test:integration

# Deploy infrastructure
npm run cdk:deploy -- --profile aws-profile

# Check Lambda logs
npm run logs:tail -- --function-name my-function
```

## Important Notes

- Always consider cost optimization in AWS resource usage
- Implement proper monitoring and alerting
- Follow the principle of least privilege for all IAM roles
- Use AWS tags for cost allocation and resource management
- Regularly review and update dependencies for security patches
