# Strategies, routing groups and sub-routers

Once the filters have left a list of deployments that may serve a request, a strategy picks one. Routing groups let different model groups use different strategies, and sub-routers decide which model group a request should go to before any of that happens. This page covers all three, plus custom strategies and routing plugins

![A request is routed by a sub-router to a model group, resolved and filtered, then the chosen strategy picks one deployment](../img/strategies-1.svg)

## Strategies

| Strategy | Picks | Reads | Runs |
| --- | --- | --- | --- |
| `simple-shuffle` (default) | a weighted random deployment | per-request weights, then `weight`, then `rpm`, then `tpm`, else uniform | in the core |
| `least-busy` | the deployment with the fewest requests in flight | in-flight counters | host |
| `usage-based-routing-v2` | the deployment with the lowest tpm this minute that stays under its tpm and rpm | per-minute usage counters | host |
| `usage-based-routing` | the v1 version of the above, kept for old configs | per-minute usage | host |
| `latency-based-routing` | the lowest recent latency, or time to first token for streams, within a buffer | latency samples | host |
| `cost-based-routing` | the cheapest deployment still under its limits | model cost map, usage | host |
| `lar1` | a deployment by the caller's confidence level, with thresholds from `routing_strategy_args` | `metadata.lar1` on the request | host, as a custom strategy |

`simple-shuffle` is the only strategy the core implements itself (`select.rs`). The others are stateful Python objects in `litellm/router_strategy/` that keep their own counters, fed by litellm's logging callbacks. For those the core asks the host for the pick with a `select` step and passes the filtered candidates, and the host runs the strategy object over them (`litellm/router_rust/steps/selection.py`). The filters always run in the core first, so a host strategy only ever sees deployments the core already approved

### Weights in `simple-shuffle`

![How simple-shuffle picks its weights: request weights, else weight, else rpm, else tpm, else uniform random](../img/strategies-2.svg)

The first source with a positive value on any candidate is used for all of them, and a candidate that does not set it gets weight 0. Per-request weights come from key or team level `router_settings`, and the proxy puts them on the request as `{model_group: {deployment_id: weight}}` (`weights.rs` validates them). One team can then send most of a group's traffic to one deployment while everyone else gets the global split, without a separate router per team. The draw matches Python's `random.choices`, so a seeded test picks the same deployment on every run

## Which strategy applies

![Which strategy applies: the request's routing_strategy, else the routing group's, else the router's](../img/strategies-3.svg)

A per-request `routing_strategy`, which the proxy forwards from key or team `router_settings`, wins. Then the routing group the resolved name belongs to. Then the router's own `routing_strategy`. Stateful strategies keep separate state per scope, so the same strategy used by two routing groups does not mix their counters (`strategy.rs`)

## Routing groups

A routing group is a named set of model groups with its own strategy

```yaml
routing_groups:
  - group_name: fast
    models: [gpt-4o-mini, claude-haiku]
    routing_strategy: latency-based-routing
  - group_name: tiers
    models: [gpt-4o-ptu, gpt-4o-paygo]
    routing_strategy: priority
    model_priorities: {gpt-4o-ptu: 1, gpt-4o-paygo: 2}
```

The group name is callable. A request for `fast` resolves to the deployments of both members, and latency-based routing picks among them. A request for `gpt-4o-mini` directly also uses latency-based routing, since its group belongs to `fast`. Every model group that is not a member of a configured routing group belongs to the implicit default group, which uses the router's `routing_strategy`

A priority group is the exception. It does not pick by a strategy of its own. It serves each member's priority as its deployments' `order`, so the order filter sends all traffic to priority 1 while any of it is healthy, and order fallbacks walk down the priorities when it fails (`routing_groups.rs`). See [How a deployment is picked](selection.md) for the order filter and [Retries, fallbacks and cooldowns](reliability.md) for order fallbacks

## Sub-routers

The auto router, complexity router, quality router and adaptive router are not strategies. They choose which model group should serve a request, before the name is resolved. Each is configured as a deployment whose `litellm_params.model` starts with `auto_router/`. `auto_router/complexity_router...`, `auto_router/quality_router...` and `auto_router/adaptive_router...` select those routers, and any other name under `auto_router/` is the auto router. Calling that deployment's `model_name` runs it

| Sub-router | Chooses by |
| --- | --- |
| auto router | matching the prompt against configured example utterances per route |
| complexity router | how hard the prompt looks, sending simple prompts to a cheaper group |
| quality router | the prompt's complexity mapped to a quality tier, then the model whose routing preferences match that tier |
| adaptive router | the kind of request (code, writing, reasoning and so on) and a bandit that learns from past responses which model serves each kind best, weighed against cost |

![Sequence in which the core asks the host to pre-route, an auto router classifies the prompt, and the core picks within the returned group](../img/strategies-4.svg)

The core asks once per pick with a `pre_route` step and resolves whatever group the host answers (`prerouting.rs`). The attempt still reports the requested name, `smart`, as its model group, so headers and spend logs show what the caller asked for. Marker deployments are dropped from every candidate list, so they are never called directly. Whether a caller may use a sub-router that belongs to a team needs the proxy's team checks, which the router asks the proxy for through `RouterHost.authorize_strategy_inference`

The same step carries session binding: a request that belongs to a coding session can be bound to the group that served the session so far

## Routing plugins

`Router(plugins=[...])` takes routing plugins, objects that look at a request and narrow which model groups or deployments may serve it, or attach signals that later stages read. The host runs the plugins during pre-routing and reports the candidate models they left, and the core keeps only the deployments of those models (`routing_plugins.rs`)

## Custom strategies

`router.register_plugin(strategy)` with a `CustomRoutingStrategyBase` does not slot into the strategy step. It replaces the whole pick. The host answers the `pre_route` step with the deployment the custom strategy returned, and the core serves that deployment as is, without resolution, filters or a strategy of its own. It may even return a deployment the registry does not hold. `lar1` works this way. See [Add a custom routing strategy](../how-to/add-a-routing-strategy.md)
