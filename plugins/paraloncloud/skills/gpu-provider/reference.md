# Provider API over HTTP

For when the ParalonCloud MCP server is not connected. Base URL `https://paraloncloud.com/api/v1`, header `Authorization: Bearer $PARALON_API_KEY` (a `prlc_` key with the My nodes permission).

```bash
# Your nodes (state, price, rental, live GPU metrics)
curl https://paraloncloud.com/api/v1/nodes -H "Authorization: Bearer $PARALON_API_KEY"

# One node
curl https://paraloncloud.com/api/v1/nodes/<node_id> -H "Authorization: Bearer $PARALON_API_KEY"

# Add a node; the answer has docker_command (holds the node's secret token)
curl -X POST https://paraloncloud.com/api/v1/nodes \
  -H "Authorization: Bearer $PARALON_API_KEY" -H "Content-Type: application/json" \
  -d '{"name": "rig-2", "type": "gpu", "os": "linux"}'

# The install command again
curl https://paraloncloud.com/api/v1/nodes/<node_id>/command -H "Authorization: Bearer $PARALON_API_KEY"

# Change: any of available, price_per_hour (USD/hour or null), spot, name
curl -X PATCH https://paraloncloud.com/api/v1/nodes/<node_id> \
  -H "Authorization: Bearer $PARALON_API_KEY" -H "Content-Type: application/json" \
  -d '{"available": false}'

# Spot node: take it back (a spot rental stops two minutes later)
curl -X POST https://paraloncloud.com/api/v1/nodes/<node_id>/reclaim -H "Authorization: Bearer $PARALON_API_KEY"

# Earnings by UTC day (days 1 to 366, default 30)
curl "https://paraloncloud.com/api/v1/earnings?days=30" -H "Authorization: Bearer $PARALON_API_KEY"
```

Node fields: `id`, `name`, `type`, `online`, `state` (`available`, `rented`, `paused`, `not_verified`, `not_ready`, `offline`), `available`, `spot`, `gpus[]` (`model`, `vram_mb`), `price` (`default`, `min`, `max`, `custom`, `renter_pays`), `verification` (`status`, `verified_at`, `error`), `rent_ready`, `rent_ready_reason`, `reputation` (`tier`, `score`), `last_seen`, `metrics` (`at`, `gpus[]` with `utilization_pct`, `temperature_c`, `power_w`, `vram_used_mb`, `vram_free_mb`; `cpu_pct`, `memory_used_mb`, `memory_free_mb`), `rental` (`id`, `status`, `started_at`, `ends_at`, `price_per_hour`, `reclaim_at`).

Earnings: `days[]` (`day`, `rentals`, `inference`, `total`, `nodes[]`), `total`, `earnings_balance`, `running_rentals_billed_before_fee`.

Errors: `400 invalid_request` (bad field, price outside min to max), `401 invalid_api_key`, `403 insufficient_scope`, `404 node_not_found`, `409 conflict` (spot change while rented, reclaim on a node that is not spot), `429 rate_limit_exceeded` (also: at most 20 new nodes an hour per key).

Full docs: https://paraloncloud.com/docs/provider-api
