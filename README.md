# 🖥️ 2f-agro-os

> Documentação técnica de **Operating Systems** — estação meteorológica edge solar.
> Matéria: **OS** (10 pts) · FIAP 3ES · GS 2026.1

[![Hub](https://img.shields.io/badge/hub-2f--agro-success)](https://github.com/GS-SPACE-CONNECT/2f-agro)

## 🎯 Objetivo
Documento técnico justificando estratégias de OS para a estação meteo autônoma do 2F-AGRO, que opera em locais sem energia confiável.

## 👥 Owner
[@lucksza](https://github.com/lucksza) · Team [`ml-os`](https://github.com/orgs/GS-SPACE-CONNECT/teams/ml-os)

## 📄 Documento técnico
> **[docs/documento-tecnico.md](docs/documento-tecnico.md)** · [PDF para entrega](docs/documento-tecnico.pdf)

## 📦 Entregáveis (10 pts)
| Item | Pts |
|---|---|
| Cenário + Arquitetura | 2 |
| 3 estratégias (Memória + CPU + I/O) | 4 |
| Conexão ODS 9 + impacto na Terra | 3 |
| Clareza | 1 |

## 🏗️ Cenário escolhido
**Estação meteorológica solar autônoma em fazenda no semiárido do Nordeste**

| Componente | Especificação |
|---|---|
| Compute | Raspberry Pi 4 (4 GB RAM, eMMC 32 GB) |
| Sensores | DHT22, BME280, anemômetro, pluviômetro, sensor de solo |
| Energia | Solar 20 W + LiFePO4 12.8V/10Ah + MPPT |
| Rede | LoRaWAN + GSM/4G (fallback) |
| OS | Raspbian Lite + kernel PREEMPT-RT |

## 🛠️ Tuning (4 pts)
1. **Memória:** `vm.swappiness=10` · `cgroups v2` · `earlyoom` · `min_free_kbytes`
2. **CPU:** `SCHED_FIFO` prio 99 (telemetria) · `taskset` core 0 · `nice +10` secundário
3. **I/O:** F2FS · ring buffer `tmpfs` · write barriers · `logrotate` com compressão lzma

## 🌍 Conexão ODS 9
Otimização que mantém satélite ativo por anos sem reinício = estação meteo 24/7 sem manutenção = democratização da tecnologia espacial pro pequeno agricultor.

## 🔗 Links
- [Spec § 4.3 OS](https://github.com/GS-SPACE-CONNECT/2f-agro/blob/main/docs/specs/2026-05-27-2f-agro-design.md)
- [IoT (sensores)](https://github.com/GS-SPACE-CONNECT/2f-agro-iot)
