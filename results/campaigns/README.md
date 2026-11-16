# Campanhas congeladas

Cada pasta é uma cópia congelada (`analyze.py freeze`) de um banco de resultados,
com os CSVs brutos de cada execução, os logs, as tabelas e os gráficos gerados a
partir dele. As rodadas novas gravam em `results/bloom.db`, que fica fora do git;
uma campanha só entra aqui depois de terminada.

Os *commits* citados abaixo e gravados em cada execução (`runs.git_commit`) são do
repositório de desenvolvimento, do qual este é uma cópia (ver o README da raiz).

| pasta | onde | o que tem |
|---|---|---|
| `vm-nvme/` | VM Debian 13, 1 GB de RAM, 2 vCPUs fixas, swap e tabelas no NVMe do host | rótulos `oficial` (campanha oficial, 2026-11-06/07, commit c2c52ec, protocolo 4), `teste-vm-nov05` (teste, commit 59a907d) e `pilot` (protocolo 3) |
| `vm-nvme-extra/` | mesma VM | rótulos `colisoes` e `delecao` (2026-11-07, commit 30fc9a0, protocolo 5): taxa de colisões estruturais e deleção, medidas depois da campanha oficial |
| `vm6-nvme/` | a mesma VM com 6 vCPUs (`scripts/vm/vm6.sh`) | rótulos `threads` (experimento 3, concorrência, 2026-11-08, commit 002a50c, schema 6) e `ponte` (trecho da campanha oficial com 1 thread, para comparar as duas VMs) |
| `fprlab/` | host, fora da VM (`tools/fprlab.c`) | investigação da FPR do híbrido acima da fórmula: variantes do hash dentro do bloco, 1M e 20M chaves, UUID e CPF (determinístico; sem tempos) |

## Como consultar

O `analyze.py` abre os bancos só para leitura: consultar, gerar tabelas ou gráficos
não altera os arquivos daqui, nem migra um schema antigo (colunas que ele não tem
aparecem como `-`).

```bash
DB=results/campaigns/vm-nvme/bloom.db
python3 scripts/analyze.py --db $DB runs --label oficial
python3 scripts/analyze.py --db $DB bench --label oficial --type uuid --sizing same_fpr none
python3 scripts/analyze.py --db $DB io --label oficial --type uuid --tables 8 --neg 0.9
```

Sem `--label`, a análise usa todas as execuções limpas do protocolo mais novo; o
rótulo separa a campanha oficial do teste. Os gráficos em `vm-nvme/plots/` e as
tabelas em `vm-nvme/tables/` (CSV com média, desvio e IC 95%; LaTeX booktabs) são
da campanha `oficial`. Para refazê-los:

```bash
make venv
.venv/bin/python scripts/analyze.py --db $DB bench --label oficial --plot DIR --format pdf png
.venv/bin/python scripts/analyze.py --db $DB io --label oficial --plot DIR --format pdf png
```

Complementar (cada banco tem um só protocolo, então não precisa de `--protocol`):

```bash
X=results/campaigns/vm-nvme-extra/bloom.db
python3 scripts/analyze.py --db $X bench --label colisoes --sizing same_fpr --metrics k fill collision_rate
python3 scripts/analyze.py --db $X bench --label delecao --sizing same_fpr none --metrics delete del_still fn_delete insert
```

`colisoes`: todos os filtros, tipos e N, 1 repetição (a contagem é determinística).
`delecao`: CBF e hybrid_c_* em UUID, 1M a 20M, 5 × 3; baseline só em 1M e 5M
(apagar exige N buscas a mais, horas quando ele pagina). Em `vm-nvme-extra/plots/`
e `tables/`, um subconjunto por rótulo.

Concorrência (`vm6-nvme/`):

```bash
V=results/campaigns/vm6-nvme/bloom.db
python3 scripts/analyze.py --db $V threads --label threads
python3 scripts/analyze.py --db $V bench --label ponte --sizing same_fpr
```

`threads`: UUID 1M e 20M; sbf (só serial), pbf (serial, partition, atomic), hybrid_512
e hybrid_4096 (serial, atomic), hybrid_c_512 e hybrid_c_4096 (serial, lock_global,
lock_shard); T = 1, 2, 4 e 6; 5 × 3. A aceleração é a média serial ÷ a média com T
threads, sem IC. Os tempos seriais do experimento 3 não se comparam com os do
benchmark (lotes de 8 MiB em vez de 1 MiB, outro laço); a comparação entre as VMs é a
`ponte`. Em `vm6-nvme/logs/`, a saída da campanha e o estado do host.

## Logs de `vm-nvme/logs/`

- `oficial-campaign.log`, `oficial-host.log`: saída da campanha e estado do host
  (perfil de energia, kernel, swap da RAM da VM no host) no início e no fim.
- `teste-vm-nov05-*.log`: o mesmo para o teste de 05/11.
- `latency-rewrite.log`, `latency-age-*.log`: medições da latência de leitura
  em função da idade das tabelas no SSD (ver "Plano da campanha oficial", abaixo).

## Ressalvas

- Baseline de UUID em 10M e 20M: 3 repetições × 1 rodada (ele pagina e leva horas);
  nos gráficos aparece como "linha de base (3 amostras)". O resto: 5 × 3; CPF, CNPJ e URL
  em 10M e 20M, só filtros e só benchmark, 2 × 1.
- A latência absoluta de leitura depende da idade das tabelas no SSD: compare
  estruturas no mesmo tipo e N, não N ou tipos entre si. Contagens de leitura e FPR
  não são afetadas.

## Plano da campanha oficial

| tipo | N | o que roda | repetições × rodadas |
|---|---|---|---|
| UUID | 1, 5, 10 e 20M | filtros: benchmark e I/O | 5 × 3 |
| UUID | 1 e 5M | linha de base: benchmark e I/O | 5 × 3 |
| UUID | 10 e 20M | linha de base: benchmark e I/O | 3 × 1 |
| CPF, CNPJ, URL | 1 e 5M | tudo, com linha de base e I/O | 5 × 3 |
| CPF, CNPJ, URL | 10 e 20M | só filtros, só benchmark | 2 × 1 |

O plano está em `scripts/vm/official.sh`, que confere todos os dados antes de começar
(`scripts/vm/vm.sh push-code && scripts/vm/vm.sh official`; dados com
`vm.sh data N TIPO`, ou `vm.sh data N TIPO none` sem tabelas). Levou ~8,5 h.

- **Linha de base com 3 × 1 em 10M e 20M:** ela pagina e fica de 50× a 120× mais
  lenta por operação; com 5 × 3 o total passaria de 18 h só em UUID. Um efeito desse
  tamanho aparece mesmo com intervalo largo. As 3 repetições são do mesmo processo e
  não captam variação entre rodadas.
- **CPF, CNPJ e URL com 2 × 1 em 10M e 20M, sem I/O:** servem para testar a dispersão
  do hash, que não depende de paginação, e a FPR não varia entre repetições.
- **Consultas novas a cada repetição no I/O (protocolo 4):** repetir as mesmas deixava
  na RAM, depois do aquecimento, as páginas que elas usam, e a paginação sumia. Medido
  na VM (10M, T = 1, 90% negativas): com as mesmas consultas, a linha de base teve 0
  faltas de página maiores por repetição e 12,5 µs; com consultas novas, ~4 850 e
  ~49 µs. As mesmas consultas valem para todas as estruturas da repetição.
- **A latência de leitura do SSD depende de quando a tabela foi escrita, não de N.**
  Medido na VM com o controle sem filtro, T = 1 (`logs/latency-*.log`): uma tabela de
  1M lia a ~100 µs; reescrita com conteúdo idêntico, a ~59 µs, enquanto a de 5M,
  intacta, ficava em ~114 µs. O mecanismo não foi confirmado (cache SLC ou mapeamento
  do SSD). Por isso nenhum dado foi escrito depois do início da campanha, e as
  latências só se comparam entre estruturas com o mesmo tipo e N. O efeito muda o
  tempo de cada leitura, não o número de leituras.
