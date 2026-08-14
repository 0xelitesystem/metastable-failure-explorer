# Metastable Failure Explorer

Solve the retry-amplification fixed point for a service and its dependency, enumerate every equilibrium, and trace the hysteresis loop between the tipping point and the recovery point.

**Live demo:** https://0xelitesystem.github.io/metastable-failure-explorer/

The headline, computed by the page from the example it loads on first paint: a service at 60 percent utilization, three attempts per request, no retry budget. Degrade the dependency's service rate and the healthy equilibrium stops existing at 62.78 percent of baseline. Restore the service rate to exactly 100 percent and the system stays collapsed, because the collapsed branch does not stop existing until 180 percent of baseline. Turn on the Google SRE client retry budget, 10 percent of requests, and run the identical shock: the tipping point barely moves, still 62.78 percent, and the recovery point drops from 180.00 percent to 67.14 percent. The budget does not stop you falling in. It stops the fall from being permanent.

## Use

Enter the dependency (servers, mean service time), the client (arrival rate, timeout, max attempts, optional retry budget), pick a control to sweep, press Solve. The page returns:

- **Every equilibrium**, marked stable or unstable, with the attempt rate, utilization, timeout probability, goodput and local slope at each one. A model with three roots is in Bronson's vulnerable state: healthy right now, holding a second stable state it will not leave on its own.
- **The two saddle-nodes.** The control value where the healthy branch stops existing, and the very different control value where the collapsed branch stops existing. The gap between them is the hysteresis loop, and it is the honest answer to "traffic went back to normal an hour ago and we are still serving 500s".
- **A transient view** for the controls that cannot move a fixed point. Backoff delay does not appear anywhere in the amplification equation, so a backoff toggle provably cannot move an equilibrium. It is simulated separately.

Four examples ship with the page. `Load example: capacity shock` is the headline above. `Same shock, retry budget on` is the comparison. `HotOS 2021 case study` is the worked example from section 2.1 of the Bronson paper. `400 servers` exists to break the textbook Erlang C formula on purpose.

Everything the page prints is a property of the numbers in the model box. Nothing on the page is fitted to, or predictive of, a real service.

## Why this exists

Retries are the most common sustaining effect in distributed systems, and the reason an outage outlives its trigger. Bronson et al. (HotOS 2021) define a metastable failure as a bad state that persists after the trigger is removed, held there by a feedback effect that usually involves work amplification. The arithmetic behind that is a scalar fixed point:

```
lambda_eff = lambda * sum over i of p(lambda_eff) to the i
```

The dependency sees `lambda_eff`, the total attempt rate. Each attempt fails with probability `p`, which depends on `lambda_eff`. Each failure adds another attempt. Solve for the roots and the bistability falls out: a healthy root, an unstable root that is the tipping threshold, and a collapsed root pinned at `lambda * max_attempts`.

Four decisions in this build are worth stating, because the obvious version of each one is wrong.

**The primary swept control is the dependency's service rate, not offered load.** Sweeping offered load cannot produce the story people actually live through, because the healthy branch survives until utilization is nearly saturated. In the shipped example the offered-load saddle-node sits at 96.9 percent utilization. That is a real number the page computes, and it is useless as an explanation for an incident that began at 60 percent. What actually starts these incidents is a capacity or latency shock, so the primary control here holds arrival rate fixed at 60 percent utilization and sweeps the dependency's service rate instead. The offered-load axis is kept as a secondary control with its saddle-node labelled honestly. The two are not equivalent: degrading the service rate also shrinks the timeout measured in service times, so a capacity shock tips at a **lower** utilization than an equal increase in load. In the shipped example, 95.6 percent against 96.9 percent.

**The failure probability comes from the Erlang C waiting-time tail, not from Kingman.** Kingman's approximation gives a mean wait. The event that generates a retry is a timeout, which is a tail event, so this uses the exact M/M/c waiting-time tail for the stated model:

```
P(W > T) = C(c,a) * exp(-(c*mu - lambda) * T)
```

That matters for more than tidiness. If the failure curve is a shape the user picks, the tool is confirming a curve the user drew, and the bistability is an artifact of the drawing. Deriving `p` from the queueing model the user already specified closes that circularity.

**Erlang C is computed in log space.** The textbook form needs `a^c / c!`. In float64 `171!` is not finite, and `a^c` stops being finite earlier than that: at 90 percent utilization the textbook expression returns a non-finite value from 150 servers up. The page computes the same ratio through lgamma and a log-sum-exp, and checks itself in the browser against an independent Erlang B recursion. That check is a table on the page, not a claim in this file.

**Backoff needed a second engine.** A backoff control cannot move a steady-state fixed point, because delay does not appear in the amplification equation at all. Shipping a backoff toggle against the steady-state solver would produce a control that does nothing and looks broken. So backoff gets its own fluid transient simulation, where retries are scheduled into future time bins by policy. In that engine, on the shipped example, the same 0.5 second capacity dip produces a permanent 2-second-period retry pulse train under a fixed schedule (peak 1800 req/s, 25 percent of wall time at zero goodput, indefinitely) and full recovery under full jitter (peak 759 req/s). The steady-state equilibria are byte-identical in both runs. That is the whole point: backoff moves the path, not the attractor.

On the retry budget, this repo does not frame it as budget instead of backoff. Google's SRE book recommends both, in different chapters and for different reasons: chapter 22 says retries should always use randomized exponential backoff, and chapter 21 describes a per-request budget of up to three attempts plus a per-client budget that stops retrying once retries pass a 10 percent share of requests, which holds request growth to about 1.1x instead of the roughly 3x a bare three-attempt cap allows. This page treats them as two separate controls because they act on two separate things, and shows which one moves which number.

## Privacy

Everything runs in the browser tab. One HTML file, no network requests of any kind, no analytics, no fonts, no storage beyond a single localStorage key for the light or dark preference. Save the file and it works offline, on a plane, inside an air-gapped network. Nothing you type leaves the machine.

## Run locally

```
git clone https://github.com/0xelitesystem/metastable-failure-explorer.git
cd metastable-failure-explorer
open index.html
```

Or download `index.html` on its own and open it. There is nothing else to fetch.

## Build

There is no build. One file, inline CSS, inline JavaScript, no dependencies and no toolchain. Edit `index.html` and reload.

The maths lives in a single `MFE` object at the top of the script block: `erlangC` and `lgamma` for the queueing tail, `ampF` and `solve` for the fixed point and its roots, `hysteresis` for the branch-following sweep, `findTransition` for the saddle-node bisection, `transient` for the second engine. The DOM wiring below it is guarded by a `typeof document` check, so the same script block loads in a plain JavaScript runtime if you want to drive the solver from a test harness.

## Related

- [retry-backoff-calculator](https://0xelitesystem.github.io/retry-backoff-calculator/) - the per-attempt delay table for a single client: attempts, base delay, cap, jitter strategy, worst-case total wait. It answers how long one request waits and models no amplification at all. This repo answers what the whole client population does to the dependency. Different question, adjacent name, so start there if you want the delay schedule and here if you want the failure mode.
- [llm-timeout-budget](https://0xelitesystem.github.io/llm-timeout-budget/) - the same timeout-versus-tail question for one request path across every layer that can time it out.
- [idempotency-and-safe-retries-reference](https://github.com/0xelitesystem/idempotency-and-safe-retries-reference) - whether a request is safe to retry at all, which is the question that comes before how often.
- [background-jobs-and-queues-basics-reference](https://github.com/0xelitesystem/background-jobs-and-queues-basics-reference) - queues, workers and dead-letter handling in plain language.
- [incident-response-runbooks](https://github.com/0xelitesystem/incident-response-runbooks) - what to do once you are already in the bad state.

Sources for the claims above: Bronson, Aghayev, Charapko and Zhu, Metastable Failures in Distributed Systems, HotOS 2021, [doi:10.1145/3458336.3465286](https://doi.org/10.1145/3458336.3465286); Google, Site Reliability Engineering, [chapter 21 Handling Overload](https://sre.google/sre-book/handling-overload/) and [chapter 22 Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/). The Erlang C formula and the M/M/c waiting-time tail are standard queueing theory and are checked in the page against an independent Erlang B recursion.

## License

MIT. See [LICENSE](LICENSE).

## More

Part of a catalog of single-file browser tools and plain-language references,
all MIT licensed and dependency-free: [0xelitesystem.github.io](https://0xelitesystem.github.io/).
Built by [elitesystem.ai](https://elitesystem.ai).
