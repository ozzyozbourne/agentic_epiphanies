# Radius MCP: tool catalog and web-search discoveries

This is a snapshot of the `mcp__radius` namespace discovered in this Pi session. The catalog contained **130 tools** at the time of inspection. Tool availability, names, permissions, behavior, and prices can change; catalog presence does not guarantee authorization. Each name below has the prefix `mcp__radius__` when called. These are Radius API operations, not all web-search tools.

## Web search versus page fetch

- `tools_webSearch_run` searches the public web from a keyword query or objective and returns links, titles, and excerpts. The search provider in this session is Parallel.
- `tools_webFetch_run` fetches content from one or more known public URLs. This is a separate operation, not another name for search.
- `organizations_webSearch_read` reads the organization's search policy; `organizations_webSearch_update` changes it. Changing policy can affect the whole organization, privacy, and billing and requires confirmation.
- The catalog also lists image generation and file description as separate utility operations. Shell `curl`, when network access permits it, is another way to request a known URL; it is not itself a search engine.

In the earlier research, keyword searches succeeded under the selected **Fast — free** policy. A separate request to fetch five URLs returned HTTP 402 for insufficient credit, with the response saying at least $0.05 more was required for that request. This does not establish that every fetch costs the same. Search worked; page fetch failed. A search result's excerpt is not equivalent to a full reading of the source.

## Search policies seen in the screenshot

| Option | Price shown per successful search | Provider zero-data-retention option |
|---|---:|---|
| Off | — | — |
| Fast — free | Free | No |
| Fast with ZDR | $0.001 | Yes |
| Advanced with ZDR | $0.005 | Yes |

ZDR means **zero data retention** under the provider's applicable terms; it does not mean the query is never sent to the provider or necessarily prevent retention in every other component. Consult current contractual and privacy terms for scope. Fast prioritizes low latency; Advanced trades additional cost and latency for potentially better search results, not a guarantee. These screenshot prices may change, apply to successful searches, and do not establish the price of page fetching or model usage. No policy was changed.

## How the catalog is used

Codemode runs JavaScript and can discover and call the Radius MCP tools. It is an orchestration interface, not the search engine itself. `find_operations` helps discover an operation from a task description. Catalog entries include read operations and potentially sensitive writes, including billing, keys, membership, and policy changes. Permissions still apply, and sensitive or destructive actions require appropriate user authorization. Search queries are sent externally to the provider.

## Complete tool-name inventory

Grouped by the first component of each tool name. Names indicate operations, but are not a substitute for their schema, permissions, or side-effect documentation.

### find (1)

- `find_operations`

### gateway (4)

- `gateway_health_read`
- `gateway_config_read`
- `gateway_credentialProviders_list`
- `gateway_modelProviders_list`

### tools (4)

- `tools_webSearch_run`
- `tools_webFetch_run`
- `tools_imageGeneration_run`
- `tools_describeFile_run`

### servers (1)

- `servers_list`

### routingRules (8)

- `routingRules_test`
- `routingRules_list`
- `routingRules_publish`
- `routingRules_read`
- `routingRules_revoke`
- `routingRules_versions_list`
- `routingRules_versions_read`
- `routingRules_builtins_list`

### rewriteRules (8)

- `rewriteRules_test`
- `rewriteRules_updatePriority`
- `rewriteRules_list`
- `rewriteRules_publish`
- `rewriteRules_read`
- `rewriteRules_revoke`
- `rewriteRules_versions_list`
- `rewriteRules_versions_read`

### context (1)

- `context_read`

### analytics (4)

- `analytics_schema_read`
- `analytics_snapshot_read`
- `analytics_directory_read`
- `analytics_query_run`

### apiKeys (5)

- `apiKeys_list`
- `apiKeys_create`
- `apiKeys_update`
- `apiKeys_attribution_update`
- `apiKeys_revoke`

### billing (9)

- `billing_summary_read`
- `billing_balance_read`
- `billing_checkoutSessions_create`
- `billing_portalSessions_create`
- `billing_creditCodes_redeem`
- `billing_autoTopUp_read`
- `billing_autoTopUp_update`
- `billing_autoTopUp_disable`
- `billing_transactions_list`

### artifacts (15)

- `artifacts_list`
- `artifacts_create`
- `artifacts_summary_read`
- `artifacts_read`
- `artifacts_visibility_update`
- `artifacts_ttl_update`
- `artifacts_delete`
- `artifacts_versions_publish`
- `artifacts_versions_read`
- `artifacts_files_download`
- `artifacts_scratch_open`
- `artifacts_scratch_read`
- `artifacts_scratch_files_put`
- `artifacts_scratch_files_delete`
- `artifacts_scratch_commit`

### packages (17)

- `packages_list`
- `packages_read`
- `packages_scoped_read`
- `packages_delete`
- `packages_scoped_delete`
- `packages_visibility_update`
- `packages_scoped_visibility_update`
- `packages_groups_add`
- `packages_scoped_groups_add`
- `packages_groups_replace`
- `packages_scoped_groups_replace`
- `packages_groups_remove`
- `packages_scoped_groups_remove`
- `packages_versions_read`
- `packages_scoped_versions_read`
- `packages_versions_yank`
- `packages_scoped_versions_yank`

### models (9)

- `models_list`
- `models_create`
- `models_upsert`
- `models_delete`
- `models_availability_list`
- `models_availability_update`
- `models_availability_batch`
- `models_pricing_list`
- `models_costEstimates_list`

### budgets (8)

- `budgets_list`
- `budgets_usage_list`
- `budgets_usage_read`
- `budgets_principals_list`
- `budgets_read`
- `budgets_create`
- `budgets_update`
- `budgets_delete`

### organizations (16)

- `organizations_privacy_read`
- `organizations_privacy_update`
- `organizations_modelCatalog_read`
- `organizations_webSearch_read`
- `organizations_webSearch_update`
- `organizations_list`
- `organizations_read`
- `organizations_create`
- `organizations_settings_read`
- `organizations_update`
- `organizations_artifactTtlPolicy_update`
- `organizations_artifactVisibilityPolicy_update`
- `organizations_twoFactor_update`
- `organizations_membershipDomain_update`
- `organizations_terms_read`
- `organizations_terms_accept`

### serviceAccounts (4)

- `serviceAccounts_list`
- `serviceAccounts_create`
- `serviceAccounts_update`
- `serviceAccounts_delete`

### groups (8)

- `groups_list`
- `groups_read`
- `groups_create`
- `groups_update`
- `groups_delete`
- `groups_members_replace`
- `groups_members_add`
- `groups_members_remove`

### members (5)

- `members_list`
- `members_read`
- `members_role_update`
- `members_groups_replace`
- `members_remove`

### invitations (3)

- `invitations_list`
- `invitations_create`
- `invitations_revoke`

## Sources and limitations

- [Parallel search modes](https://docs.parallel.ai/search/modes)
- [Parallel search best practices](https://docs.parallel.ai/search/best-practices)
- The exact tool inventory was obtained by inspecting the Radius MCP namespace in this session; the screenshot supplied by the user showed the listed policy prices and selected free option.
- The pricing and privacy explanations above are not a contractual interpretation. The full provider pages were not fetched during the earlier research because of insufficient fetch credits.
