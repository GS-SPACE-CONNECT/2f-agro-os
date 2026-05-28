# 2f-agro-os

Doc tecnica de **Operating Systems** - estacao meteo edge solar.
Materia: **OS** (10 pts) | FIAP 3ES | GS 2026.1

[![Hub](https://img.shields.io/badge/hub-2f--agro-success)](https://github.com/GS-SPACE-CONNECT/2f-agro)

## Objetivo
Doc tecnico justificando estrategias de OS para a estacao meteo autonoma do 2F-AGRO, sem energia confiavel.

## Owner
[@lucksza](https://github.com/lucksza) | Team [`ml-os`](https://github.com/orgs/GS-SPACE-CONNECT/teams/ml-os)

## Entregaveis (10 pts)
- Cenario + Arquitetura (2 pts)
- 3 estrategias (Mem + CPU + I/O) (4 pts)
- Conexao ODS 9 + impacto Terra (3 pts)
- Clareza (1 pt)

## Cenario escolhido
**Estacao meteo solar autonoma em fazenda no semiarido NE**

- Compute: Raspberry Pi 4 (4GB RAM, eMMC 32GB)
- Sensores: DHT22, BME280, anemometro, pluviometro, sensor solo
- Energia: Solar 20W + LiFePO4 12.8V/10Ah + MPPT
- Rede: LoRaWAN + GSM/4G fallback
- OS: Raspbian Lite + kernel PREEMPT-RT

## Tuning (4 pts)
1. **Memoria:** vm.swappiness=10 | cgroups v2 | earlyoom | min_free_kbytes
2. **CPU:** SCHED_FIFO prio 99 (telemetria) | taskset core 0 | nice +10 secundario
3. **I/O:** F2FS | ring buffer tmpfs | write barriers | logrotate compress lzma

## ODS 9
Otimizacao que mantem satelite ativo anos sem reinicio = estacao meteo 24/7 sem manutencao = democratizacao da tecnologia espacial pro pequeno agricultor.

## Links
- [Spec OS](https://github.com/GS-SPACE-CONNECT/2f-agro/blob/main/docs/specs/2026-05-27-2f-agro-design.md)
- [IoT (sensores)](https://github.com/GS-SPACE-CONNECT/2f-agro-iot)
