# Microsoft - Detecting Late Arriving Events in Sentinel

## The Core Problem

A SIEM can collect a record yet miss detecting it: the record arrives after its event-time window has passed. Widening that window creates overlapping evaluations.

This case studies scheduled rules; Microsoft's [ingestion-delay guide](https://learn.microsoft.com/en-us/azure/sentinel/ingestion-delay) now recommends unified custom detections for new rules.

## Architecture & Component Design

**Separate frequency from coverage.** The [scheduled-rules overview](https://learn.microsoft.com/en-us/azure/sentinel/scheduled-rules-overview#query-scheduling) distinguishes how often a Kusto query executes from how much historical data it examines. Sentinel also executes scheduled analytics rules with a built-in five-minute delay. That scheduler allowance is distinct from variable source-to-workspace latency; it is not evidence that every connector delivers within five minutes.

**Measure arrival lag.** Compare `ingestion_time()` with `TimeGenerated` per data type; joined sources can have different delays.

**Use two time filters.** Microsoft's example combines a five-minute rule interval with a two-minute delay allowance. It expands the configured lookback to seven minutes and filters event time with `TimeGenerated >= ago(7m)`. It also requires `ingestion_time() > ago(5m)`, selecting recently ingested records within that broader event-time range. The wider range recovers late arrivals; the ingestion filter limits repeated selection caused by overlapping event-time windows. These numbers illustrate the mechanism, not a universal production setting.

**Keep detection and packaging separate.** Query-result thresholds determine whether a run produces alerts. Event grouping controls whether results become one alert or individual alerts; alert grouping combines alerts into incidents. Those settings, documented in the overview, do not replace selecting the correct input records.

## Trade-offs & Bottlenecks

The [Kusto function reference](https://learn.microsoft.com/en-us/kusto/query/ingestion-time-function) defines ingestion time as approximate query-readiness time measured by the service. Concurrent ingestion operations can overlap, so these timestamps do not establish a strict ordering. If the ingestion-time policy was not enabled when records arrived, the function returns null. Design implication: validate that the relevant tables provide usable values before relying on this filter.

This windowing pattern is not end-to-end [[Idempotency]]. Kusto explicitly distinguishes approximate ingestion timestamps from exactly-once table processing, for which its documentation points to database cursors. Design implication: repeated remediation requests still need their own protection against duplicate side effects; selecting a record once during normal scheduling does not prove an external action executes once.

Design implication: a finite lookback still leaves a lateness boundary. Monitor delayed sources and test events near that boundary rather than treating a wider window as unlimited recovery.

## Key Takeaway

Separate when an event happened, when it became queryable, and how a detection becomes an alert. Each boundary solves a different SIEM reliability problem.

Sources linked inline; reviewed November 2, 2026.
