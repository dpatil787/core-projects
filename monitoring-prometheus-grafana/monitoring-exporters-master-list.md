# Prometheus Exporters - Complete Reference Guide

This document lists all major Prometheus exporters and what they monitor.

---

## Operating System & Hardware

| Exporter | What It Monitors |
|----------|------------------|
| Node Exporter | Linux system metrics (CPU, memory, disk, network, load average, hardware temperature) |
| Windows Exporter | Windows system metrics (CPU, disk, memory, IIS, Active Directory, Windows Services) |

---

## Databases

| Exporter | What It Monitors |
|----------|------------------|
| MySQL Exporter | MySQL metrics (query rate, connections, lock waits, InnoDB buffer pool, slow queries, replication health) |
| PostgreSQL Exporter | PostgreSQL metrics (active connections, query execution times, cache hit ratios, deadlocks, locks) |
| Redis Exporter | Redis metrics (keyspace hits/misses, memory fragmentation, connected clients, evicted keys) |

---

## Message Queues

| Exporter | What It Monitors |
|----------|------------------|
| Kafka Exporter | Kafka metrics (topic partition lag, consumer group offsets, broker health, message throughput) |
| RabbitMQ Exporter | RabbitMQ metrics (queue stats, consumers, connections, message rates) |

---

## Web Servers & Proxies

| Exporter | What It Monitors |
|----------|------------------|
| Nginx Exporter | Nginx metrics (connections, requests, active connections) |
| Apache Exporter | Apache web server metrics |
| HAProxy Exporter | HAProxy metrics (frontend/backend stats, request rates) |

---

## Network Monitoring

| Exporter | What It Monitors |
|----------|------------------|
| Blackbox Exporter | Endpoint probing (HTTP, HTTPS, DNS, TCP, ICMP for uptime/latency from outside) |
| SNMP Exporter | Network devices (routers, switches via SNMP protocol) |

---

## Container & Orchestration

| Exporter | What It Monitors |
|----------|------------------|
| cAdvisor | Container resource usage (Docker/Kubernetes CPU, memory, network) |
| kube-state-metrics | Kubernetes object states (deployments, pods, nodes, cronjobs) |

---

## JVM & Applications

| Exporter | What It Monitors |
|----------|------------------|
| JMX Exporter | JVM applications (heap memory, threads, GC pauses, custom MBeans for Kafka/Cassandra/Spark) |

---

## Logging

| Exporter | What It Monitors |
|----------|------------------|
| Rsyslog Exporter | Converts Rsyslog log metrics to Prometheus format (log rates, error counts) |

---

## Search

| Exporter | What It Monitors |
|----------|------------------|
| Elasticsearch Exporter | Cluster health, shard stats, index metrics |

---

## Use Case Priority

| Priority | Exporter | When to Use |
|----------|----------|-------------|
| 1 | Node Exporter | Always - every Linux system needs this |
| 2 | Blackbox Exporter | When monitoring website uptime from outside |
| 3 | MySQL/PostgreSQL Exporter | When monitoring databases |
| 4 | Nginx/Apache Exporter | When monitoring web servers |
| 5 | Rsyslog Exporter | For centralized logging projects |
| 6 | cAdvisor + kube-state-metrics | For Kubernetes monitoring |

---

**Last Updated:** April 2026
**Project:** DevOps Monitoring Stack