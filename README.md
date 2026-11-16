# Filtros de Bloom para Otimização de I/O em Big Data

Código, experimentos e resultados do Trabalho de Conclusão de Curso
"Filtros de Bloom para Otimização de I/O em Big Data: Estudo Comparativo e Proposta
de um Modelo Híbrido" (Universidade de Caxias do Sul).

Autor: Lorenzo Mottin Jablonski · Orientadora: Helena Graziottin Ribeiro

## O que tem aqui

Quatro variantes de Filtro de Bloom da literatura, um modelo híbrido proposto e uma
linha de base exata, todos implementados em C99 do zero, com a mesma interface
(`include/bf.h`) e o mesmo protocolo de medição:

| estrutura | nome no código | resumo |
|---|---|---|
| Filtro de Bloom Padrão | `sbf` | vetor de m bits, k chamadas de MurmurHash2 |
| Filtro de Bloom com Contagem | `cbf` | contadores de 4 bits (saturam em 15); permite remoção |
| Filtro de Bloom Particionado | `pbf` | k partições alinhadas a 64 bytes, uma por função de hash |
| Filtro de Bloom r-Dimensional | `rdbf` | matriz X × Y de células de 64 bits (robustBF, Eq. 1), uma chamada de hash |
| Modelo híbrido | `hybrid_512`, `hybrid_4096` | blocos de 512 ou 4096 bits alinhados à linha de cache; uma chamada de MurmurHash64A escolhe o bloco e as k posições (hash duplo) numa matriz de primos dentro dele |
| Modelo híbrido com contadores | `hybrid_c_512`, `hybrid_c_4096` | o mesmo, com contadores de 4 bits; permite remoção |
| Linha de base exata | `baseline` | tabela hash (uthash) que guarda cópia de cada chave |

E três experimentos:

1. **Benchmark em memória** (`src/benchmark.c`): inserção, busca de chaves existentes
   e inexistentes, remoção, memória, FPR medida × teórica, chamadas de hash, colisões
   e faltas de página. Cada estrutura roda num processo próprio, e as chaves são lidas
   em lotes por um buffer fixo de 1 MiB.
2. **Filtros como porteiros de I/O** (`src/iobench.c`, `src/store.c`): T tabelas
   ordenadas em disco, no formato de uma SSTable, com blocos de 4 KB lidos com
   `O_DIRECT`; o filtro de cada tabela decide se o disco é lido.
3. **Concorrência** (`src/threadbench.c`): inserção e busca com 1 a 6 threads, com
   divisão por partição, OR atômico, trava global ou trava por bloco.

## Requisitos

- GCC (C99) e make; Linux (usa `/proc`, `O_DIRECT` e pthreads).
- Python 3, só com a biblioteca padrão, para gerar dados, executar e analisar.
- matplotlib, só para as figuras (`make venv` cria um `.venv` com ela).
- Para o ambiente controlado: QEMU/KVM, libvirt (`qemu:///session`, sem root) e rsync.

## Uso rápido

```bash
make all          # compila o bloom_app
make check        # testes unitários em C (tests/)
make test         # invariantes de todas as estruturas pela interface genérica
make demo         # demonstração rápida
make data N=1000000 TYPES="uuid cpf cnpj url" SEED=42   # conjuntos de dados
make benchmark    # experimento 1 -> results/bloom.db (REPS=5 P=0.01 LABEL=...)
make iobench      # experimento 2 (TABLES="1 8" NEG="0.5 0.9" QUERIES=10000 STORE_DIR=data/store)
python3 scripts/run.py threadbench --filters pbf hybrid_512 hybrid_c_512 --threads 1 2 4 6   # experimento 3
python3 scripts/analyze.py runs|bench|io|threads   # tabelas no terminal, --csv ou --latex
make venv && make plots                            # figuras em results/plots
```

## Resultados

Os resultados usados no trabalho estão congelados em `results/campaigns/`, com os
bancos SQLite, o CSV bruto de cada execução, os logs, as tabelas e as figuras:

| pasta | conteúdo |
|---|---|
| `vm-nvme/` | campanha principal (experimentos 1 e 2): 1 786 execuções, 4 tipos de chave, 1 a 20 milhões de chaves |
| `vm-nvme-extra/` | colisões estruturais e remoção |
| `vm6-nvme/` | concorrência (experimento 3), numa VM com 6 vCPUs |
| `fprlab/` | investigação da FPR do híbrido acima da sua fórmula (`tools/fprlab.c`) |

O `results/campaigns/README.md` descreve cada campanha, o plano de execução e as
ressalvas, e traz os comandos para refazer tabelas e figuras a partir dos bancos.
O `analyze.py` abre os bancos só para leitura. Por exemplo:

```bash
python3 scripts/analyze.py --db results/campaigns/vm-nvme/bloom.db bench --label oficial --type uuid --sizing same_fpr none
```

As medições foram feitas numa VM Debian 13 com 1 GB de RAM, criada por
`scripts/vm/vm.sh`, para que o índice exato deixe de caber na memória a partir de 10
milhões de chaves. Cada execução registra o *commit*, o *hash* do executável e do
conjunto de dados e a configuração da máquina e dos discos.

## Estrutura

```
include/, src/   estruturas, interface genérica (bf.h, bf.c) e programas de medição
tests/           testes unitários de cada estrutura
scripts/         geração de dados (gen_datasets.py, partition.py), execução (run.py),
                 banco de resultados (resultsdb.py), análise (analyze.py, plots.py)
scripts/vm/      criação da VM e roteiros das campanhas
tools/           fprlab.c: investigação da FPR do modelo híbrido
results/         campanhas congeladas (results/campaigns/); execuções novas vão para
                 results/bloom.db, fora do git
data/            conjuntos de dados e tabelas gerados (fora do git)
```

## Origem

Este repositório é uma cópia, sem o histórico, do repositório de desenvolvimento no
*commit* `54a9853`. Os *commits* gravados nos resultados se referem àquele repositório.

## Terceiros

`include/uthash.h` é a biblioteca uthash, de Troy D. Hanson, distribuída sob a sua
própria licença (BSD), que está no cabeçalho do arquivo. A MurmurHash2 e a
MurmurHash64A foram reimplementadas a partir do algoritmo de Austin Appleby (domínio
público) e conferidas com os valores de verificação do SMHasher.
