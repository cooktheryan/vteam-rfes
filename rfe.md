# Restore Jira Integration Using Atlassian MCP Server

**Feature Overview:**
*Replace the custom Python-based Jira integration that was previously implemented in vTeam with the official Atlassian MCP (Model Context Protocol) Server. This migration will provide a more robust, secure, and maintainable solution for publishing RFE workflow artifacts to Jira, while leveraging Anthropic's open MCP standard and Atlassian's supported infrastructure.*

The vTeam application previously had working Jira integration (commits c149a3f, 6497b4c, bae8bfc) that allowed users to publish workspace files from RFE workflows directly to Jira issues. This functionality was implemented using custom Go code in the backend that made direct REST API calls to Jira. This RFE proposes restoring this capability using the Atlassian MCP Server instead of custom API integration code.

**Goals:**

*Restore the ability for vTeam users to seamlessly publish RFE workflow artifacts (specifications, architectures, epics, user stories, etc.) to Jira from within the vTeam UI. The new implementation will leverage the Atlassian MCP Server to provide:*

* **Enhanced Security**: OAuth 2.0 authentication with TLS 1.2+ encryption, respecting all existing Jira permissions
* **Reduced Maintenance**: Shift integration maintenance to Atlassian's supported infrastructure rather than custom code
* **Better Reliability**: Leverage Atlassian's rate-limited, production-grade MCP server with built-in health monitoring
* **Future-Proof Architecture**: Build on the open Model Context Protocol standard for potential expansion to other integrations
* **Preserved User Experience**: Maintain the same or better UX as the previous custom integration

**Target Users**: Development teams, product managers, and architects using vTeam for RFE workflow management who need to create and track Jira issues for their feature requirements.

**Expected Outcome**: Users can configure Jira settings in their project settings page and publish individual workspace files to Jira with one click, creating linked issues that are tracked within the RFE workflow CR (Custom Resource).

**Out of Scope:**

*Items explicitly excluded from this Feature:*

* Bi-directional sync between Jira and vTeam workspaces
* Automatic Jira issue updates when workspace files change
* Confluence integration (may be considered in future RFE)
* Support for on-premise Jira instances (Atlassian MCP Server requires Atlassian Cloud)
* Migration of existing Jira links from the old implementation
* Custom Jira field mapping beyond summary, description, and project
* Jira webhook handling for status updates

**Requirements:**

*Specific needs and capabilities that must be delivered:*

* **[MVP] MCP Client Integration**: Integrate the existing MCP client library (`/archive/mcp_client_integration`) into the vTeam backend, enabling connection to Atlassian MCP Server
* **[MVP] OAuth Authentication Flow**: Implement OAuth 2.0 authentication flow for Atlassian Cloud, replacing the current basic auth (email + API token) approach
* **[MVP] Issue Creation**: Restore ability to create Jira issues from RFE workflow workspace files via MCP instead of direct REST API calls
* **[MVP] Linkage Tracking**: Maintain the `jiraLinks` field in RFEWorkflow CRD to track relationships between workspace file paths and Jira issue keys
* **[MVP] Frontend UI Preservation**: Preserve existing UI elements for Jira configuration in project settings and "Publish to Jira" functionality in RFE workflow pages
* **Backend Refactoring**: Refactor Go backend handlers to use MCP client instead of direct HTTP calls to Jira REST API
* **Configuration Management**: Update environment configuration and Kubernetes manifests to support Atlassian MCP Server connection
* **Error Handling**: Implement comprehensive error handling for MCP connection failures, authentication issues, and Jira operation errors
* **Rate Limit Handling**: Implement graceful handling of Atlassian MCP Server rate limits (1,000 requests/hour for Premium/Enterprise)
* **Health Monitoring**: Add health check endpoints for MCP server connectivity status

**Done - Acceptance Criteria:**

*The feature is complete when:*

* Users can configure Atlassian Cloud OAuth credentials in vTeam project settings
* Users can successfully publish workspace files from RFE workflows to Jira, creating new issues via Atlassian MCP Server
* Created Jira issues contain proper summary, description, and are assigned to the configured project
* RFEWorkflow CRs properly track `jiraLinks` entries mapping file paths to Jira keys
* The frontend displays Jira issue links and allows users to open them in a new tab
* Error messages are clear and actionable when Jira operations fail
* MCP server health status is visible to administrators
* All existing Jira integration UI elements work as they did in the previous implementation
* Documentation is updated to reflect OAuth setup requirements and Atlassian Cloud prerequisites
* Integration tests validate end-to-end Jira issue creation workflow

**Use Cases - i.e. User Experience & Workflow:**

**Main Success Scenario: Publishing an RFE Specification to Jira**

1. Product Manager creates a new RFE workflow in vTeam with title "Dark Mode Support"
2. An AI agent generates a specification document in the workspace at `spec/requirements.md`
3. Product Manager navigates to Project Settings and configures Atlassian OAuth credentials
4. Product Manager opens the RFE workflow detail page
5. Product Manager clicks "Publish to Jira" button next to the `spec/requirements.md` file
6. Backend uses MCP client to create a Jira issue with summary "[RFE] Dark Mode Support - specify"
7. Jira issue is created successfully with issue key "PROJ-123"
8. RFEWorkflow CR is updated with jiraLinks entry: `{path: "spec/requirements.md", jiraKey: "PROJ-123"}`
9. UI displays the Jira link next to the file with clickable issue key
10. Product Manager clicks the issue key and Jira opens in a new tab

**Alternative Flow 1: Jira Configuration Missing**

1-4. Same as main scenario
5. Product Manager clicks "Publish to Jira" button
6. Backend checks for Jira configuration in runner secret
7. Configuration is missing or incomplete
8. User receives error message: "Jira integration not configured. Please configure in Project Settings."
9. User navigates to settings and completes configuration
10. Returns to main success scenario

**Alternative Flow 2: MCP Server Connection Failure**

1-5. Same as main scenario
6. Backend attempts to connect to Atlassian MCP Server
7. Connection fails (network issue, MCP server down, etc.)
8. Error is caught and logged with context
9. User receives error message: "Unable to connect to Jira integration service. Please try again later."
10. Administrator checks MCP server health monitoring dashboard

**Alternative Flow 3: OAuth Token Expired**

1-5. Same as main scenario
6. Backend uses MCP client with OAuth token
7. Atlassian returns 401 Unauthorized (token expired)
8. System detects OAuth error
9. User receives error message: "Jira authentication expired. Please re-authenticate in Project Settings."
10. User re-authenticates and returns to main scenario

**Documentation Considerations:**

*Documentation updates required:*

* **Admin Guide**: Add section on deploying and configuring Atlassian MCP Server connection, including environment variables and Kubernetes secret configuration
* **User Guide**: Update Jira integration setup instructions to reflect OAuth authentication flow instead of API token
* **Migration Guide**: Create guide for administrators migrating from old Jira integration to MCP-based integration
* **Prerequisites**: Document requirement for Atlassian Cloud (on-premise Jira not supported by MCP Server)
* **Rate Limits**: Document Atlassian MCP Server rate limits and how they affect usage
* **Troubleshooting**: Add troubleshooting section for common MCP connection and authentication issues
* **API Reference**: Update backend API documentation to reflect changes to Jira-related endpoints
* **Architecture Docs**: Update architecture diagrams to show MCP Server integration layer

**Questions to answer:**

*Refinement and architectural questions:*

1. **MCP Client Language**: The existing MCP client library is in Python. Should we create a Go MCP client library, use the Python client via subprocess/service, or evaluate a Go-based MCP SDK?
2. **OAuth Token Storage**: Where should OAuth refresh tokens be stored? In the existing runner Secret? In a separate Secret? How do we handle token refresh?
3. **MCP Server Deployment**: Should the Atlassian MCP Server connection be configured per-project or globally for the vTeam instance?
4. **Rate Limit Strategy**: How should we handle Atlassian MCP Server rate limits? Should we implement client-side rate limiting, queuing, or just fail fast with clear errors?
5. **Backward Compatibility**: Should we maintain the existing REST API-based integration as a fallback option, or fully commit to MCP?
6. **MCP Server Hosting**: Will we rely on Atlassian's hosted MCP Server or deploy our own using the open-source implementation?
7. **Multi-Workspace Support**: Previous implementation created one Jira issue per workspace file. Should we support batch operations or multiple file selection?
8. **Custom Fields**: The previous implementation only set summary, description, and project. Should we support custom Jira fields through MCP?
9. **Go-Python Bridge**: If we use the Python MCP client from Go, what's the best approach? gRPC service? REST wrapper? Subprocess execution?
10. **Issue Type Selection**: Previous implementation always created generic issues. Should we allow users to specify issue types (Story, Task, Epic, Bug)?

**Background & Strategic Fit:**

*Additional context:*

vTeam is a platform for managing RFE (Request for Enhancement) workflows using AI agents to generate specifications, architectures, and implementation plans. A key part of the workflow is transitioning from planning to execution by creating tracked work items in Jira.

The previous custom Jira integration (implemented in September 2025, commits c149a3f through bae8bfc) worked well for basic use cases but had limitations:
- Required manual API token management
- Direct REST API calls required ongoing maintenance as Jira API evolves
- No OAuth support meant less secure credential storage
- Custom error handling for Jira-specific API quirks
- No standardized way to extend to other Atlassian products (Confluence, Compass)

Atlassian announced their Remote MCP Server in early 2025 as part of their AI integration strategy. The MCP Server provides:
- Official Atlassian support and maintenance
- OAuth 2.0 security model
- Built-in rate limiting and reliability features
- Standardized protocol for accessing Jira, Confluence, and other Atlassian products
- Alignment with Anthropic's MCP standard, which is gaining industry adoption

By migrating to the Atlassian MCP Server, vTeam positions itself to leverage a supported, enterprise-grade integration that can expand to other Atlassian products and potentially other MCP-enabled services in the future.

The vTeam codebase already contains an MCP client integration library (archived in `/archive/mcp_client_integration`) that was developed for earlier Llama Stack integration work. This library provides connection pooling, health monitoring, and error handling patterns that can be leveraged for this integration.

**Customer Considerations**

*Customer-specific considerations:*

* **Atlassian Cloud Requirement**: Customers using on-premise Jira Server or Data Center will not be able to use this integration. The Atlassian MCP Server only supports Atlassian Cloud. This should be clearly documented and communicated.
* **OAuth Setup Overhead**: OAuth authentication requires more setup than simple API tokens (creating OAuth apps in Atlassian). This may be a barrier for small teams. Documentation must be comprehensive and include screenshots.
* **Rate Limits**: Customers on Atlassian Standard plans have lower rate limits (moderate usage thresholds) compared to Premium/Enterprise. High-volume users may hit limits. Consider adding rate limit monitoring and warnings.
* **Red Hat Internal Use**: For Red Hat internal use cases with issues.redhat.com (Jira), ensure the OAuth app registration and MCP Server work with Red Hat's Atlassian instance.
* **Data Residency**: Atlassian MCP Server is hosted by Atlassian. Customers with data residency requirements should understand that requests flow through Atlassian's infrastructure, though data is encrypted in transit.
* **Migration Path**: Existing vTeam users who configured the old Jira integration will need to reconfigure with OAuth. Provide a smooth migration experience and communication plan.
* **Feature Parity**: Ensure the MCP-based integration provides equal or better functionality than the custom integration to avoid customer regression.
