# AWS Bedrock AgentCore Gateway - Lambda to MCP Tools

A comprehensive guide and implementation for creating AWS Bedrock AgentCore Gateways that expose AWS Lambda functions as Model Context Protocol (MCP) tools, enabling seamless integration with AI agents using Strands Agents.

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## 🎯 Overview

This project demonstrates how to:

1. **Create an AgentCore Gateway** with Cognito-based authentication (EZAuth flow)
2. **Register Lambda functions** as MCP tools that can be discovered and invoked by AI agents
3. **Connect Strands Agents** to the gateway and use the Lambda-backed tools in agent workflows
4. **Manage multiple Lambda targets** with custom tool schemas

The gateway acts as a bridge between your AWS Lambda functions and AI agents, allowing them to discover and invoke your functions as tools through the Model Context Protocol.

## ✨ Features

- 🔐 **Secure Authentication**: Cognito-based OAuth 2.0 authentication with automatic setup
- 🚀 **Easy Lambda Integration**: Simple configuration to expose Lambda functions as MCP tools
- 🔧 **Flexible Tool Schemas**: Support for inline payload schemas and S3-based schemas
- 🤖 **Agent Integration**: Seamless integration with Strands Agents framework
- 📊 **Observability**: Built-in logging and tracing capabilities
- 🎯 **Multiple Targets**: Support for registering multiple Lambda functions as separate targets

## 🏗️ Architecture

```
┌─────────────────┐
│  Strands Agent  │
│   (Bedrock AI)  │
└────────┬────────┘
         │ MCP Protocol
         │ (HTTP/SSE)
         ▼
┌─────────────────────────────┐
│   AgentCore Gateway         │
│   (Cognito Auth)            │
└────────┬────────────────────┘
         │
         ├──► Lambda Target 1 (get_weather, get_time)
         │
         └──► Lambda Target 2 (get_random_number)
```

### Components

- **AgentCore Gateway**: Managed AWS service that exposes Lambda functions via MCP protocol
- **Cognito User Pool**: Handles OAuth 2.0 authentication and authorization
- **Lambda Functions**: Your business logic exposed as MCP tools
- **Strands Agents**: AI agent framework that consumes MCP tools

## 📦 Prerequisites

Before you begin, ensure you have:

- **Python 3.12+** installed
- **AWS CLI** configured with appropriate credentials
- **AWS Account** with permissions for:
  - Bedrock AgentCore Gateway
  - AWS Lambda
  - Amazon Cognito
  - IAM (for role creation)
- **AWS Profile** configured (or use default credentials)

## 🔧 Installation

1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd mcpfying-lambda-into-mcp-tools
   ```

2. **Install dependencies**:
   ```bash
   pip install bedrock_agentcore_starter_toolkit strands-agents strands-agents-tools
   ```

   Or install from the notebook:
   ```python
   !pip install bedrock_agentcore_starter_toolkit strands-agents strands-agents-tools
   ```

## ⚙️ Configuration

### Environment Variables

Set up your AWS configuration:

```python
import os

os.environ['AWS_DEFAULT_REGION'] = 'eu-central-1'  # Your AWS region
os.environ['AWS_PROFILE'] = 'your-aws-profile'     # Optional: AWS profile name
```

### Required AWS Permissions

Your AWS credentials need permissions for:

- `bedrock-agentcore:CreateGateway`
- `bedrock-agentcore:CreateTarget`
- `bedrock-agentcore:DeleteGateway`
- `cognito-idp:*` (for Cognito setup)
- `lambda:*` (for Lambda function access)
- `iam:CreateRole`, `iam:AttachRolePolicy` (for execution role)

## 🚀 Usage

### Step 1: Create the Gateway

```python
from bedrock_agentcore_starter_toolkit.operations.gateway.client import GatewayClient
import logging

# Setup the client
client = GatewayClient(region_name=os.environ['AWS_DEFAULT_REGION'])
client.logger.setLevel(logging.DEBUG)

# Create Cognito Authorizer (EZAuth flow)
cognito_authorizer = client.create_oauth_authorizer_with_cognito("agentcore-gateway-test")

# Create the Gateway
gateway = client.create_mcp_gateway(authorizer_config=cognito_authorizer["authorizer_config"])

gatewayID = gateway["gatewayId"]
gatewayUrl = gateway["gatewayUrl"]

print(f"Gateway ID = {gatewayID}")
print(f"Gateway URL = {gatewayUrl}")
```

### Step 2: Create Lambda Targets

#### Option A: Default Lambda Target

```python
# Creates a default Lambda with sample tools
lambda_target = client.create_mcp_gateway_target(
    gateway=gateway, 
    target_type="lambda"
)
```

#### Option B: Custom Lambda Target

```python
lambda_target_configuration = {
    'lambdaArn': 'arn:aws:lambda:eu-central-1:ACCOUNT:function:YourFunction',
    'toolSchema': {
        'inlinePayload': [
            {
                'name': 'your_tool_name',
                'description': 'Tool description',
                'inputSchema': {
                    'type': 'object',
                    'properties': {
                        'param1': {'type': 'string'}
                    },
                    'required': ['param1']
                },
                'outputSchema': {
                    'type': 'string'
                }
            }
        ]
    }
}

response = client.create_mcp_gateway_target(
    gateway=gateway,
    target_type="lambda",
    target_payload=lambda_target_configuration
)
```

### Step 3: Connect with Strands Agents

```python
from strands import Agent
from strands.models import BedrockModel
from strands.tools.mcp.mcp_client import MCPClient
from mcp.client.streamable_http import streamablehttp_client

# Get access token
access_token = client.get_access_token_for_cognito(cognito_authorizer["client_info"])

# Ensure gateway URL ends with /mcp
mcp_url = gatewayUrl if gatewayUrl.endswith('/mcp') else f"{gatewayUrl}/mcp"

# Create MCP client
streamable_http_mcp_client = MCPClient(
    lambda: streamablehttp_client(
        url=mcp_url,
        headers={"Authorization": f"Bearer {access_token}"}
    )    
)

# Start the MCP client
streamable_http_mcp_client.start()

# Get available tools
tools = streamable_http_mcp_client.list_tools_sync()
print(f"Found {len(tools)} tools")

# Create agent with tools
model = BedrockModel(model_id="eu.amazon.nova-pro-v1:0")
agent = Agent(model=model, tools=tools)

# Use the agent
response = agent("Get the time for PST")
print(response)
```

**⚠️ Important**: Keep the MCP client running while using the agent. Stop it when done:

```python
streamable_http_mcp_client.stop()
```

## 📁 Project Structure

```
mcpfying-lambda-into-mcp-tools/
├── gateway-target-lambda-oauth.ipynb    # Main Jupyter notebook with examples
└── README.md                    # This file
```

## 🧹 Cleanup

To clean up AWS resources:

```python
# Stop MCP client if running
try:
    if streamable_http_mcp_client.is_running():
        streamable_http_mcp_client.stop()
except:
    pass

# Delete all targets first
targets = client.list_gateway_targets(gatewayID)
for target in targets:
    client.delete_gateway_target(gatewayID, target["targetId"])

# Delete the gateway
client.delete_gateway(gatewayID)
```

**Note**: The Cognito User Pool and Lambda functions are not automatically deleted. You may need to clean them up manually if they were created by this script.

## 🔍 Troubleshooting

### Common Issues

1. **Authentication Errors**
   - Verify your AWS credentials are configured correctly
   - Check that your AWS profile has the necessary permissions
   - Ensure the Cognito domain has propagated (may take a few minutes)

2. **Gateway Creation Fails**
   - Verify Bedrock AgentCore is available in your region
   - Check IAM permissions for gateway creation
   - Ensure you have sufficient AWS service quotas

3. **MCP Client Connection Issues**
   - Verify the gateway URL is correct and ends with `/mcp`
   - Check that the access token is valid and not expired
   - Ensure the gateway is in an active state

4. **Lambda Invocation Errors**
   - Verify Lambda function ARNs are correct
   - Check Lambda execution role permissions
   - Ensure tool schemas match Lambda function signatures

### Debug Logging

Enable debug logging to troubleshoot issues:

```python
import logging
client.logger.setLevel(logging.DEBUG)
```

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request. For major changes, please open an issue first to discuss what you would like to change.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📄 License

This repository is a personal, educational reference implementation inspired by public AWS documentation and publicly available AWS sample projects.

## 📚 References

- [AWS Bedrock AgentCore Documentation](hhttps://docs.aws.amazon.com/bedrock-agentcore/)
- [Model Context Protocol (MCP) Specification](https://modelcontextprotocol.io/)
- [Strands Agents Documentation](https://github.com/strands-ai/strands-agents)
- [Bedrock AgentCore Starter Toolkit](https://github.com/aws-samples/bedrock-agentcore-starter-toolkit)

## 🙏 Acknowledgments

- AWS Bedrock Team for AgentCore Gateway
- Strands AI for the agents framework
- The MCP protocol community

---

**Note**: This is a sample implementation. For production use, ensure proper error handling, security best practices, and resource management.
