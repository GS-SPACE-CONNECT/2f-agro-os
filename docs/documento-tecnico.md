# Sistemas Operacionais em Ambientes Extremos — A Fronteira Espacial

**Disciplina:** Operating Systems
**Professor:** Prof. Ricardo Black
**Curso:** 3º Ano de Engenharia de Software — FIAP
**Projeto:** 2F-AGRO — Estação meteorológica solar autônoma (edge) para o pequeno agricultor
**ODS alvo:** ODS 9 — Indústria, Inovação e Infraestrutura

**Integrantes:**

| Nome | RM |
|---|---|
| Lucca Borges | 554608 |
| Ruan Melo | 557599 |
| Rodrigo Jimenez | 558148 |
| João Victor Franco | 556790 |
| Bruno Leão | 555563 |

---

## Sumário Executivo

Este documento propõe e justifica um conjunto de estratégias de **Operating Systems (OS Tuning)** para a
**estação meteorológica solar autônoma do projeto 2F-AGRO**, instalada em fazendas do **semiárido do
Nordeste**, sem rede elétrica nem internet confiáveis. O nó é, na prática, um **dispositivo de borda
(*edge computing*) projetado com a mesma filosofia de engenharia das sondas e satélites espaciais**: deve
sobreviver por anos sem manutenção, sem reinício físico imediato, com energia e armazenamento limitados e
com confiabilidade primordial.

Aplicamos diretamente o aprendizado da exploração espacial — onde otimizar o sistema operacional é questão
de sobrevivência da missão — a um problema concreto da Terra. Apresentamos a arquitetura de hardware,
justificamos a escolha de **Raspbian Lite com kernel PREEMPT-RT**, detalhamos três estratégias de *tuning*
(**Memória, CPU e I/O**) com comandos reais, e fechamos com a conexão ao **ODS 9** e seu impacto
quantitativo no campo.

---

## 1. Definição do Cenário e Arquitetura

### 1.1 Cenário de aplicação

**Estação meteorológica solar autônoma (2F-AGRO)** instalada em propriedades rurais do semiárido. O
dispositivo coleta dados ambientais (temperatura, umidade do ar e do solo, pressão, vento, chuva), executa
**processamento na borda** — calcula índices agrometeorológicos, detecta eventos críticos (risco de geada,
estresse hídrico, janela de pulverização) — e transmite **apenas o resultado útil** por LoRaWAN, com
*fallback* GSM/4G. O objetivo é operar **24/7 por anos, sem energia da rede e sem alguém para reiniciá-la**.

Esse é exatamente o desafio de um sistema espacial trazido para a Terra: **autonomia, eficiência energética
radical e tolerância a falhas**. A largura de banda (LoRa) e a energia (painel solar) são os recursos mais
escassos — por isso processar na borda e otimizar o SO não é luxo, é requisito de sobrevivência do nó.

### 1.2 Arquitetura de hardware

| Componente | Especificação | Justificativa |
|---|---|---|
| **Compute** | Raspberry Pi 4 (Broadcom BCM2711, quad-core Cortex-A72 @ 1.5 GHz, 4 GB RAM) + eMMC 32 GB | SoC ARM de baixo consumo, 4 núcleos para isolar tarefas críticas, amplo suporte a Linux e drivers. |
| **Sensores** | DHT22 (temp/umidade), BME280 (pressão/umidade), anemômetro, pluviômetro, sensor de umidade de solo | Cobrem as variáveis agrometeorológicas; interfaces I²C/1-Wire/GPIO nativas do Pi. |
| **Energia** | Painel solar 20 W + bateria LiFePO4 12,8 V / 10 Ah + controlador MPPT | LiFePO4 tem milhares de ciclos e tolera calor do semiárido; MPPT maximiza colheita solar. Orçamento típico de poucos watts. |
| **Armazenamento** | eMMC 32 GB (NAND) | *Flash* tem ciclos de escrita finitos → exige tuning de I/O para não “queimar”. |
| **Rede** | LoRaWAN (primário, longo alcance/baixo consumo) + GSM/4G (*fallback*) | Conectividade em área sem cobertura confiável; LoRa transmite poucos bytes a quilômetros. |
| **Resiliência de energia** | Monitor de tensão (GPIO *power-good*) + *supercapacitor* para *flush* final | Permite desligamento limpo em queda de energia, garantindo integridade dos dados. |

### 1.3 Justificativa do Sistema Operacional base

**Escolha: Raspbian Lite (Debian *headless*) com kernel *patcheado* com PREEMPT-RT.**

**Por que Linux (Raspbian) e não um RTOS puro?**

1. **Carga de trabalho mista e rica em I/O.** A estação precisa de pilha **LoRaWAN**, modem **GSM/4G**,
   drivers de sensores (**I²C, 1-Wire, GPIO**), sistema de arquivos robusto (**F2FS**), `cgroups`, e
   bibliotecas de processamento (Python/NumPy). Um RTOS minimalista não entrega esse ecossistema.
2. **Tuning profundo disponível.** O Linux expõe exatamente os mecanismos que este projeto usa:
   `swappiness`, **cgroups v2**, `earlyoom`, `min_free_kbytes`, `SCHED_FIFO`, `taskset`, **F2FS**,
   *write barriers* e `logrotate` — permitindo otimização fina sem reescrever o sistema.
3. **Determinismo onde importa, via PREEMPT-RT.** O *patch* PREEMPT-RT torna o kernel majoritariamente
   preemptível, reduzindo a latência de interrupção a faixas previsíveis — suficiente para o tempo real
   *soft* da telemetria e da janela de transmissão LoRa.
4. **Raspbian Lite = mínimo e auditável.** Versão *headless* (sem ambiente gráfico), com poucos serviços
   ativos, *boot* rápido, menor superfície de ataque e menor consumo de RAM/energia.

> **Conexão espacial:** essa é a mesma decisão de arquitetura adotada em rovers e satélites modernos —
> **Linux endurecido para a maior parte das tarefas, com camadas de tempo real para o que é crítico**.
> Trazemos esse padrão de engenharia espacial para um Raspberry Pi numa fazenda.

---

## 2. Estratégias de Tuning de Kernel e Gerenciamento de Recursos

> Princípio orientador: **a estação fica em poste no meio da roça — não há “desligar e ligar de novo”
> imediato.** Cada estratégia prioriza **prevenção de falhas, previsibilidade e proteção do hardware**
> sobre desempenho de pico, exatamente como num sistema espacial.

### 2.1 Gerenciamento de Memória — evitar o OOM sem reinício físico

**Problema:** mesmo com 4 GB, picos de processamento (lote de leituras, compressão, inferência) podem
esgotar a RAM. Um *Out of Memory* descontrolado aciona o OOM Killer do kernel, que pode matar o processo de
**telemetria** — e a estação fica muda no campo, sem ninguém para reiniciá-la.

**Estratégias aplicadas:**

1. **`vm.swappiness` baixo — manter o crítico na RAM.**
   - `sysctl -w vm.swappiness=10` (persistente em `/etc/sysctl.d/99-agro.conf`)
   - Evita *swap* agressivo para a eMMC (lenta e com desgaste por escrita), mantendo páginas ativas na RAM.

2. **Isolamento por cgroups v2 — conter o que pode estourar.**
   ```bash
   echo "+memory" > /sys/fs/cgroup/cgroup.subtree_control
   mkdir /sys/fs/cgroup/proc          # grupo das tarefas pesadas (processamento)
   echo 1G  > /sys/fs/cgroup/proc/memory.max    # teto rígido (OOM local)
   echo 768M > /sys/fs/cgroup/proc/memory.high  # throttle antes do teto
   echo $PID > /sys/fs/cgroup/proc/cgroup.procs # move o processo
   ```
   Se o processamento estourar o limite, **apenas ele** é contido — a telemetria, num grupo com
   `memory.min` reservado, nunca é pressionada.

3. **`earlyoom` — matar o sacrificável antes do *lockup*.**
   - Um *daemon* leve em espaço de usuário monitora a RAM livre e, ao cair de um limiar, mata
     **proativamente** o processo menos importante (ex.: compressão de histórico) **antes** que o kernel
     trave o sistema inteiro. Configuração: `earlyoom -m 8 -s 100` (age com 8% de RAM livre), com
     `--prefer` apontando para tarefas secundárias e `--avoid` protegendo telemetria.

4. **`min_free_kbytes` — reserva contra rajadas.**
   - `sysctl -w vm.min_free_kbytes=65536` aumenta a reserva de memória livre que o kernel mantém,
     dando folga para alocações de rajada (interrupções, rede) sem disparar OOM.

5. **`oom_score_adj` — escolher a vítima certa.**
   - `echo -1000 > /proc/$(pidof telemetria)/oom_score_adj` torna a telemetria praticamente imune ao OOM
     Killer; tarefas descartáveis recebem valor positivo (`echo 500 > ...`).

### 2.2 Escalonamento de Processos (CPU) — prioridade absoluta para a telemetria

**Problema:** a leitura/transmissão de telemetria não pode atrasar porque uma compressão ou um cálculo
pesado monopolizou a CPU. É preciso **prioridade absoluta e previsível**.

**Estratégias aplicadas:**

1. **`SCHED_FIFO` de alta prioridade para o crítico.**
   - `chrt -f 99 ./telemetria` (ou `chrt -f -p 99 <PID>`): a telemetria roda em tempo real e **sempre**
     preempta qualquer tarefa comum (`SCHED_OTHER`). A coleta/compressão secundária só executa quando
     **não há** trabalho crítico pendente — a “prioridade absoluta” exigida.

2. **Fixar o crítico a um núcleo dedicado (`taskset`/isolamento).**
   - `taskset -c 0 ./telemetria` confina a telemetria ao **núcleo 0**, eliminando interferência de cache e
     *context switch* das tarefas pesadas (que rodam nos núcleos 1–3).
   - Opcionalmente, isolar o núcleo no *boot* (em `cmdline.txt`): `isolcpus=0 nohz_full=0` retira o núcleo 0
     do balanceador do scheduler.

3. **Rebaixar o secundário com `nice`.**
   - `nice -n 10 ./processa_historico` (ou `renice +10 -p <PID>`): processamento não essencial roda com
     baixa prioridade, cedendo CPU instantaneamente quando a telemetria precisa.

4. **Reserva de banda RT + DVFS (energia solar).**
   - `sysctl -w kernel.sched_rt_runtime_us=950000` garante ~5% de CPU às tarefas comuns mesmo sob carga RT
     máxima (evita *runaway*).
   - *Governor* de frequência conforme o orçamento solar: `cpupower frequency-set -g ondemand` de dia e
     `-g powersave` à noite/bateria baixa — reduzir frequência e adormecer núcleos ociosos **economiza
     energia**, recurso tão crítico quanto CPU numa estação solar.

### 2.3 I/O e Armazenamento — desgaste mínimo e integridade na falta de energia

**Problema:** a eMMC (NAND) tem ciclos de escrita finitos; gravar a cada leitura de sensor a “queima” em
meses. E uma queda de energia (noite longa, falha de bateria) não pode corromper os dados nem o sistema.

**Estratégias aplicadas:**

1. **F2FS — sistema de arquivos feito para *flash*.**
   - `mkfs.f2fs -l DADOS /dev/mmcblk0p2`; em `/etc/fstab`:
     `/dev/mmcblk0p2  /dados  f2fs  noatime,nodiratime,fsync_mode=strict  0 2`
   - O F2FS faz *wear leveling* e escrita sequencial (distribui o desgaste); `noatime` elimina a escrita de
     *timestamp* a cada leitura; `fsync_mode=strict` dá integridade mais forte.

2. **Ring buffer em `tmpfs` — agrupar escritas, poupar a NAND.**
   - As leituras de sensores entram primeiro num **buffer circular em RAM** (`tmpfs`, ex.: `/run/sensors`),
     e só são gravadas na eMMC **em lote** (ex.: a cada N minutos). Isso transforma milhares de
     micro-escritas em poucas escritas grandes, reduzindo drasticamente o desgaste.
     ```bash
     # /etc/fstab — buffer volátil de 64 MB em RAM
     tmpfs  /run/sensors  tmpfs  size=64M,noatime,mode=0755  0 0
     ```

3. **Write barriers + *flush* controlado — integridade em queda de energia.**
   - Montagem com *write barriers* (padrão no F2FS) garante a **ordem** das gravações, evitando estado
     corrompido após queda abrupta. Dados críticos usam `fsync()` explícito + **escrita atômica**
     (gravar em arquivo temporário e `rename()`).
   - O **monitor de tensão (GPIO *power-good*)** dispara um *shutdown* limpo via `systemd` ao detectar
     queda; o **supercapacitor** segura energia por alguns segundos para o `sync` final do ring buffer.

4. **`logrotate` com compressão LZMA — não deixar log encher a eMMC.**
   - `logrotate` diário com `compress`/`compresscmd xz` (LZMA): comprime e descarta logs antigos,
     limitando o volume gravado e a quantidade de blocos reescritos.
   - I/O *scheduler* leve para *flash*: `echo none > /sys/block/mmcblk0/queue/scheduler` (sem reordenação
     custosa de discos rotativos).

> **Síntese da seção 2:** as três frentes trabalham juntas — cgroups/`earlyoom` blindam a memória,
> `SCHED_FIFO`+`taskset` blindam a CPU da telemetria, e o ring buffer + F2FS + *write barriers* blindam o
> armazenamento e os dados. Sob qualquer pressão, **a telemetria nunca cai e os dados nunca corrompem** —
> o mesmo objetivo de um sistema espacial.

---

## 3. Conexão com o ODS 9 e Impacto na Terra

**ODS 9 — Indústria, Inovação e Infraestrutura:** “Construir infraestruturas resilientes, promover a
industrialização inclusiva e sustentável e fomentar a inovação.”

A tese central inverte o olhar habitual: **as otimizações de SO que mantêm um satélite vivo por anos, sem
reinício e com pouca energia, são exatamente o que faz uma estação meteorológica funcionar 24/7 numa
fazenda sem infraestrutura.** A fronteira espacial vira ferramenta de inclusão produtiva no campo.

### 3.1 Da órbita para a roça: a mesma engenharia

| Otimização de origem espacial | Aplicação na estação 2F-AGRO |
|---|---|
| Autonomia sem reinício físico (sonda/satélite) | `earlyoom` + cgroups + `oom_score_adj`: a estação não trava sozinha no poste, sem ninguém para religá-la. |
| Prioridade absoluta de telemetria (controle de missão) | `SCHED_FIFO` + `taskset` core 0: o alerta de geada/seca tem prioridade sobre tudo. |
| Eficiência energética extrema (orçamento de watts) | DVFS + Raspbian Lite + processamento na borda: opera com painel solar de 20 W. |
| Telemetria de baixa banda (downlink escasso) | LoRaWAN: envia só o resultado processado, poucos bytes, a quilômetros. |
| Proteção de armazenamento e dados (radiação/falha) | F2FS + ring buffer + *write barriers*: a eMMC dura anos e os dados sobrevivem à queda de energia. |

### 3.2 Impacto e alinhamento com o ODS 9

- **Infraestrutura resiliente (meta 9.1):** um nó autônomo e confiável onde não há energia nem internet
  democratiza o acesso a dados agrometeorológicos — tecnologia de ponta no semiárido, não só no agronegócio
  de grande porte.
- **Industrialização sustentável (meta 9.4):** otimização profunda = **menos energia, menos hardware, menos
  desperdício**. Uma estação que faz mais com poucos watts reduz a pegada de carbono e dispensa
  infraestrutura cara.
- **Inovação (metas 9.5 / 9.b):** o mesmo *stack* serve a alerta de geada, gestão de irrigação, previsão de
  janela de plantio/colheita e monitoramento de microclima — levando P&D de origem espacial para resolver
  problemas locais e aumentar a produtividade do pequeno agricultor.

**Estimativas quantitativas de impacto (ordem de grandeza)**

Estimativas de ordem de grandeza, baseadas em parâmetros típicos do hardware e de rádio LoRa — servem para
dimensionar o ganho, não como medições exatas:

- **Consumo médio:** Raspberry Pi 4 *tunado* (Lite, núcleos ociosos rebaixados, DVFS, picos curtos) opera na
  faixa de **~2 a 4 W médios**, viável com painel de 20 W mesmo em dias nublados.
- **Banda transmitida:** *payload* LoRa de dezenas de bytes a no máximo ~**240 bytes**/mensagem (limite
  prático LoRaWAN), contra dados brutos da ordem de **MB** — processar na borda reduz a banda em
  **~3 a 5 ordens de grandeza**.
- **Autonomia:** com solar 20 W + LiFePO4 10 Ah, a operação tende a ser energeticamente autossustentável,
  com autonomia de **dias** sem sol e vida útil da bateria de **anos** (LiFePO4 ~ milhares de ciclos).
- **Alcance:** enlace LoRa de **~2 a 15 km** em campo aberto, dispensando torres densas e cabeamento no
  campo.
- **Vida útil da eMMC:** o *ring buffer* em `tmpfs` + escrita em lote + `logrotate` reduz o volume de
  escritas em **~1 a 2 ordens de grandeza**, estendendo na mesma ordem a vida do *flash*.

### 3.3 O ciclo virtuoso

A tecnologia que mantém um satélite operando sozinho por anos é a mesma que mantém a estação 2F-AGRO viva no
semiárido. **Otimização profunda de recursos computacionais é a chave da inovação sustentável — seja na
órbita da Terra, na superfície de Marte ou numa fazenda sem energia no interior do Brasil.**

---

## 4. Conclusão

A escolha de **Raspbian Lite + PREEMPT-RT** é tecnicamente coerente com uma estação meteorológica solar
autônoma, e três frentes de *tuning* — **memória** (`swappiness`, cgroups v2, `earlyoom`, `min_free_kbytes`,
`oom_score_adj`), **CPU** (`SCHED_FIFO` 99, `taskset` core 0, `nice +10`, reserva RT, DVFS) e **I/O**
(F2FS, ring buffer em `tmpfs`, *write barriers*, `logrotate` LZMA) — resolvem diretamente os riscos do
ambiente: esgotamento de memória, perda de prioridade da telemetria e desgaste/corrupção do armazenamento.
Importando a engenharia que mantém sistemas espaciais vivos por anos, o projeto 2F-AGRO entrega
infraestrutura resiliente e sustentável ao pequeno agricultor, em alinhamento direto com o **ODS 9**.

---

### Glossário rápido

- **OOM (Out of Memory):** esgotamento de RAM; o kernel mata processos para sobreviver.
- **cgroups v2:** mecanismo do Linux para limitar/isolar CPU, memória e I/O por grupo de processos.
- **swappiness:** parâmetro (0–100) do quão agressivamente o kernel usa *swap*.
- **earlyoom:** *daemon* que mata o processo menos importante antes de o sistema travar por falta de RAM.
- **min_free_kbytes:** quantidade de RAM que o kernel mantém sempre livre como reserva.
- **SCHED_FIFO:** política de escalonamento de tempo real; preempta processos comuns.
- **taskset / isolcpus:** fixam um processo a um núcleo / isolam um núcleo do escalonador.
- **DVFS:** ajuste dinâmico de frequência/tensão da CPU para poupar energia.
- **F2FS:** sistema de arquivos otimizado para memória *flash* (NAND/eMMC).
- **tmpfs:** sistema de arquivos em RAM (volátil), usado como buffer para poupar a *flash*.
- **write barrier:** garantia de ordem de gravação, preservando a integridade em queda de energia.
- **PREEMPT-RT:** *patch* do kernel Linux que reduz latências e o torna mais determinístico.
- **LoRaWAN:** rádio de longo alcance e baixíssimo consumo para *payloads* pequenos.
