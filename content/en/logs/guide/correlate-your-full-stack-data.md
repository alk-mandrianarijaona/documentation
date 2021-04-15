---
title: Correlate your full stack data
kind: guide
aliases:
  - /logs/guide/xxx
further_reading:
  - link: /logs/guide/logs-rbac/
    tag: Documentation
    text: "Set up Roles Based Access Controls (RBAC) for Log Management"
  - link: /agent/logs/advanced_log_collection
    tag: Documentation
    text: "Filter and Redact logs with Advanced Log Collection"
  - link: /agent/guide/autodiscovery-management/
    tag: Documentation
    text: "Exclude containers from Log Collection with Autodiscovery"

---

## Overview

Logs and traces contain valuable information that truly shine when seen together. Correlating your data enhance your analytics (e.g with [unified service tagging](https://docs.datadoghq.com/getting_started/tagging/unified_service_tagging)) but also your searches. How often do we find ourselves slicing and dicing through logs to find one critical information? On the other hand, how can we evaluate one log criticity without its context?

For example, production slow queries are hard to reproduce and analyze without investing a lot of time and resources. Let's see how we can easily correlate slow query analysis with traces.

{{< img src="logs/guide/correlate-your-full-stack-data/database-slow-query-correlation.png" alt="Slow query logs correlation" style="width:80%;" >}}

Correlating your logs also allows for [consistent sampling based on Trace ID](https://docs.datadoghq.com/logs/indexes/#sampling-consistently-with-higher-level-entities) without losing search capabilities.

This guide walks you through the steps you should take to correlate your full stack for search purposes:

1. [Correlate your application logs](#correlate-your-application-logs)
2. [Correlate your proxy logs](#correlate-your-proxy-logs)
3. [Correlate your database logs](#correlate-your-database-logs)
4. [Correlate your queuing logs](#correlate-your-queuing-logs)
5. [Test the whole correlation from a Synthetic test](#test-the-whole-correlation-from-a-synthetic-test)
6. [Correlate your browser logs](#correlate-your-browser-logs)
7. [Correlate RUM views](#correlate-rum-views)

Note: Depending on your use case, you may skip steps. Steps dependant from others are explicit.

## Correlate your application logs

Application logs are the backbone of your context that give most of the code and business logic issues. They can even help you solve other services issues, e.g. most ORMs logs database errors.

Use one of the [various OOTB correlations](https://docs.datadoghq.com/tracing/connect_logs_and_traces/). If you use a custom tracer or if you have any issues, you can go on the [correlation FAQ](https://docs.datadoghq.com/tracing/faq/why-cant-i-see-my-correlated-logs-in-the-trace-id-panel/).

## Correlate your proxy logs

Proxy logs provide higher-tier information than application logs. They also cover wider entrypoints than application logs, like static content and redirections.

The application tracer generates Trace IDs by default. This can be changed by injecting `x-datadog-trace-id` in HTTP Request headers.

{{< tabs >}}
{{% tab "NGINX" %}}

### Setup opentracing

Follow [NGINX tracing integration](https://docs.datadoghq.com/tracing/setup_overview/proxy_setup/?tab=nginx).

### Inject Trace ID in logs

Trace ID is stored as `opentracing_context_x_datadog_trace_id` variable. Update the NGINX log format by adding the following configuration block in the http section of your NGINX configuration file `/etc/nginx/nginx.conf`:

```conf
http {
  log_format main '$remote_addr - $opentracing_context_x_datadog_trace_id $http_x_forwarded_user [$time_local] "$request" '
          '$status $body_bytes_sent "$http_referer" '
          '"$http_user_agent" "$http_x_forwarded_for" ';

  access_log /var/log/nginx/access.log;
}
```

### Parse Trace ID in pipelines

1. Clone NGINX pipeline.

2. Customize the first [grok parser](https://docs.datadoghq.com/logs/processing/processors/?tab=ui#grok-parser):
   - In *Parsing rules*, replace the first parsing rule with:
   ```text
   access.common %{_client_ip} %{_ident} %{_trace_id} %{_auth} \[%{_date_access}\] "(?>%{_method} |)%{_url}(?> %{_version}|)" %{_status_code} (?>%{_bytes_written}|-)
   ```
   - In `Advanced settings -> Helper Rules`, add the line:
   ```text
   _trace_id %{notSpace:dd.trace_id:nullIf("-")}
   ```

3. Add a Trace Id remapper on `dd.trace_id` attribute.

{{% /tab %}}
{{< /tabs >}}

## Correlate your database logs

As said in the introduction, database logs are hard to contextualize. Let's simplify it on PostgreSQL.

### Enrich your database logs

PostgreSQL default logs are not detailed. Follow [this integration guide](https://docs.datadoghq.com/integrations/postgres/?tab=host#log-collection) to enrich them.

Following our redline, we also want to have rich plan explanation on our slow queries. For having execution plan results, update `/etc/postgresql/<VERSION>/main/postgresql.conf` with:

```conf
session_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '500s'
```

Your query longer than 500ms will now log their execution plan.

Note: `auto_explain.log_analyze = 'true'` provide even more information but greatly impact performance. You can learn more on the [official documentation](https://www.postgresql.org/docs/9.2/auto-explain.html).

### Inject trace_id into your database logs

SQL logs are now enriched, but still not correlated. Using SQL comments, we can inject `trace_id` into most of our database logs. Here is an example with Flask and SQLAlchemy:

```python
if os.environ.get('DD_LOGS_INJECTION') == 'true':
    from ddtrace.helpers import get_correlation_ids
    from sqlalchemy.engine import Engine
    from sqlalchemy import event

    @event.listens_for(Engine, "before_cursor_execute", retval=True)
    def comment_sql_calls(conn, cursor, statement, parameters, context, executemany):
        trace_id, span_id = get_correlation_ids()
        statement = f"{statement} -- dd.trace_id=<{trace_id or 0}>"
        return statement, parameters
```

You can now customize PostgreSQL pipeline by adding a new grok parser:

```text
extract_trace %{data}\s+--\s+dd.trace_id=<%{notSpace:dd.trace_id}>\s+%{data}
```

Add a Trace Id remapper on `dd.trace_id` attribute.

Note: this will only correlate logs that include your statement. Error logs like `ERROR:  duplicate key value violates unique constraint "user_username_key"` will stay out of context. Most of the time you can still get error information through your application logs.

## Correlate your queuing logs

TODO

## Test the whole correlation from a Synthetic test

The APM integration with Synthetic Monitoring allows you to go from a test run that potentially failed to the root cause of the issue by looking at the trace generated by that very test run.

Having network-related specifics (thanks to your test) as well as backend, infrastructure, and log information (thanks to your trace) allows you to access a new level of details about the way your application is behaving, as experienced by your user.

For that, simply [enable APM integration on Synthetic settings](https://docs.datadoghq.com/synthetics/apm).

## Correlate your browser logs

TODO

## Correlate RUM views

TODO
