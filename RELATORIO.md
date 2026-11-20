# Relatório do laboratório — Gerenciamento de Disco: HDD, SSD SATA e NVMe

**Sala nº:** ___

| | Integrante 1 | Integrante 2 |
| --- | --- | --- |
| Nome | Arthur Reis | Matheus Assis |
| Sistema operacional | Windows 11 (10.0.26200) | |
| Tipo de disco (HDD, SSD SATA ou NVMe) | NVMe (C:) e HDD SATA (D:) | |
| Modelo do disco | Viper M.2 VPN100 (477 GB) e Seagate ST2000DM008 (1,8 TB) | |
| Ferramenta usada (fio, CrystalDiskMark, WSL) | fio 3.43 para Windows (PowerShell) | |

As saídas completas do `fio` de cada teste estão na pasta `resultados/` (`nvme_*` e `hdd_*`).

## Passo 1 — Os dois testes

| Teste | Integrante | IOPS | Throughput (MB/s) | Latência média (com unidade) |
| --- | --- | --- | --- | --- |
| A — Sequencial 1 MB | 1 (NVMe) | 1654 | 1735 | 600,54 µs |
| B — Aleatório 4 KB | 1 (NVMe) | 59,1k | 242 | 344,50 µs |
| A — Sequencial 1 MB | 1 (HDD) | 182 | 191 | 5450,47 µs (5,45 ms) |
| B — Aleatório 4 KB | 1 (HDD) | 228 | 0,94 | 139975,47 µs (140 ms) |
| A — Sequencial 1 MB | 2 | | | |
| B — Aleatório 4 KB | 2 | | | |

Throughput em MB/s (em MiB/s: 1655, 231, 183 e 0,89). O teste B rodou com `iodepth=32`.

**Pergunta 1.** Quantas vezes o throughput do teste A é maior que o do B? Quantas vezes o IOPS do B é maior que o do A? Qual era a resposta certa da enquete?

> No NVMe, o throughput do A (1735 MB/s) é 7,2x o do B (242 MB/s), e o IOPS do B (59,1k) é 35,7x o do A (1654). No HDD a diferença de throughput é bem maior, cerca de 205x (191 MB/s contra 0,94 MB/s), e o IOPS do B é só 1,25x o do A (228 contra 182), porque cada leitura de 4 KB espalhada obriga o braço do disco a se mover. Quanto à enquete, a resposta que os nossos números sustentam é que não existe um número único do disco: sequencial com blocos grandes dá MB/s altos e aleatório com blocos pequenos dá IOPS, e o mesmo disco pode ser muito bom em um e fraco no outro.

**Conferência.** Escolham um teste e verifiquem: throughput ≈ IOPS × tamanho do bloco.

> 1654 IOPS × 1 MiB (1,048576 MB) = 1734 MB/s (medido: 1735 MB/s). No teste B: 59100 IOPS × 4 KB = 242 MB/s (medido: 242 MB/s).

## Passo 2 — O efeito das filas

| Integrante | IOPS com `iodepth=1` | IOPS com `iodepth=32` | Quantas vezes maior |
| --- | --- | --- | --- |
| 1 (NVMe) | 9736 | 59,1k | 6,1x |
| 1 (HDD) | 96 | 228 | 2,4x |
| 2 | | | |

**Pergunta 2.** O IOPS mudou muito? Por quê?

> No NVMe mudou muito (6,1x) e no HDD pouco (2,4x). O NVMe tem várias filas e seus chips de flash trabalham em paralelo, então com 32 pedidos em andamento ele atende vários ao mesmo tempo; com 1 pedido por vez ele fica ocioso entre um e outro. O preço é a latência de cada pedido, que subiu de 101 µs para 345 µs porque os pedidos esperam na fila. O HDD tem um só braço e atende um pedido por vez: a fila só deixa ele reordenar os pedidos para andar menos (2,4x), e a latência por pedido pulou de 10,4 ms para 140 ms.

## Passo 3 — A armadilha do cache

| Integrante | Throughput com `--direct=1` (do passo 1) | Throughput sem `--direct=1` (2ª rodada) |
| --- | --- | --- |
| 1 (NVMe) | 1735 MB/s | 5426 MB/s |
| 1 (HDD) | 191 MB/s | 3899 MB/s |
| 2 | | |

**Pergunta 3.** O número explodiu? O que vocês mediram de verdade?

> Explodiu: 3,1x no NVMe e 20x no HDD. Sem `--direct=1` o Windows guarda o arquivo de 1 GB no cache da RAM e as leituras saem de lá, então medimos a memória, não o disco: o HDD "leu" a quase 4 GB/s, o que nenhum HDD faz. Já a 1ª rodada veio do cache (3852 MiB/s no HDD), provavelmente porque o arquivo tinha acabado de ser criado e lido. No NVMe o ganho é menor porque o disco já é rápido, e o cache só vence muito um disco lento.

## Passo 4 — Desafio

| Aplicação | Métrica que decide | Disco escolhido | Número que justifica |
| --- | --- | --- | --- |
| A) Servidor de vídeo 4K (arquivos de 50 GB) | Throughput sequencial (MB/s) | HDD (Seagate ST2000DM008) | 191 MB/s sequencial; um arquivo de 50 GB de uns 2 h pede cerca de 7 MB/s por espectador, ou seja, uns 27 em paralelo |
| B) Banco de transações (milhares de leituras de 4 KB/s) | IOPS e latência em leitura aleatória de 4 KB | NVMe (Viper M.2 VPN100) | 59,1k IOPS contra 228 do HDD (259x); 101 µs contra 10,4 ms por leitura com 1 pedido |

**Pergunta 4.** Justifiquem as duas escolhas em até quatro linhas, usando os números medidos.

> A) O vídeo é lido do início ao fim em blocos grandes, então o que decide é o throughput sequencial: o HDD entrega 191 MB/s e cada espectador de um arquivo de 50 GB (uns 2 h) consome cerca de 7 MB/s, então ele atende em teoria uns 27 e custa menos por GB (1,8 TB contra 477 GB); o NVMe, com 1735 MB/s, seria sobra.
> B) O banco faz milhares de leituras de 4 KB espalhadas, então decidem o IOPS e a latência: o HDD faz 228 IOPS (140 ms por pedido com fila), muito abaixo de milhares por segundo, e o NVMe faz 59,1k IOPS (259x) com 101 µs por leitura.

## Para o debate

A frase que a dupla vai dizer na sala principal (1 minuto):

> Vídeo 4K é throughput: o nosso HDD faz 191 MB/s e já alimenta uns 27 espectadores de um arquivo de 50 GB. Banco de transações é IOPS: o nosso NVMe faz 59,1k leituras de 4 KB por segundo contra 228 do HDD, 259x mais. O mesmo disco é bom ou ruim conforme o padrão de acesso, e é ele que decide a escolha.
