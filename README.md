# Esteira Seletora de Materiais Metálicos

Trabalho Interdisciplinar 2026.2

Instituto Federal de Minas Gerais (IFMG), Campus Itabirito
Bacharelado em Engenharia Elétrica

Protótipo de uma esteira que recebe uma peça, a posiciona sob um sensor indutivo, identifica se ela é metálica e a direciona: peças **metálicas** são retiradas pelo M.R.P., peças **não metálicas** seguem pela esteira até serem descartadas. O sistema também detecta e sinaliza falhas de operação.

O projeto é desenvolvido em duas frentes: a lógica e o circuito são validados no simulador [Wokwi](https://wokwi.com) e depois a esteira é construída fisicamente como protótipo. O mesmo código (`Código C++`) serve às duas.

**Projeto no Wokwi:** `https://wokwi.com/projects/478442287129532417`

## Equipe

- Antônio Martins Junior
- Cairo Araújo Lima
- Guilherme Henrique Faria
- João Paulo Braga
- Lucas Salles Santana Ferreira
- Philipe Eduardo Faria

## Arquivos

| Arquivo | Descrição |
|---|---|
| `Código C++` | Código do Arduino Nano (máquina de estados) |
| `README.md` | Este documento |
| `LICENSE` | Licença MIT |

## Componentes

| Componente | Função | No Wokwi | No protótipo |
|---|---|---|---|
| Arduino Nano | Controlador, alimentado por 12 V no VIN | Arduino Nano | Arduino Nano |
| Sensor indutivo (PNP, 12 V) | Identifica metal | Botão 1 (tecla `1`) | Sensor indutivo |
| Sensor infravermelho de presença (PNP, 5 V) | Início do processo, peça colocada na esteira | Botão 2 (tecla `2`) | Sensor infravermelho |
| Sensor infravermelho de detecção (PNP, 5 V) | Ponto de análise | Botão 3 (tecla `3`) | Sensor infravermelho |
| Módulo relé | Aciona o motor da esteira | Módulo relé | Módulo relé |
| Motor da esteira | Move a peça | LED ciano (12 V) | Motor da esteira |
| M.R.P. | Retira a peça metálica | LED no pino 6 | Mecanismo de retirada |
| LEDs indicadores | Metal, não metal e erro | LEDs | LEDs |

## Pinagem

| Pino | Tipo | Função |
|---|---|---|
| 2 | Saída | Relé do motor da esteira |
| 3 | Saída | LED de material **metálico** |
| 4 | Saída | LED de material **não metálico** |
| 5 | Saída | LED amarelo de **erro** |
| 6 | Saída | Aciona o **M.R.P.** |
| 10 | Entrada | Sensor infravermelho de **detecção** (ponto de análise) |
| 11 | Entrada | Sensor infravermelho de **presença** (início do processo) |
| 12 | Entrada | Sensor **indutivo** (identifica metal) |

## Notas sobre o circuito

- **Entradas com pull-down externo de 10 kΩ.** No código elas são `INPUT` (nunca `INPUT_PULLUP`) e a lógica é **ativa em HIGH**, compatível com sensores de saída PNP.
- **Sensores infravermelhos (pinos 10 e 11):** alimentados com 5 V, sem necessidade de resistor em série.
- **Sensor indutivo (pino 12):** alimentado com 12 V. O pino do Nano suporta no máximo 5 V, então use **15 kΩ em série** com o pull-down de 10 kΩ (divisor, cerca de 4,8 V no pino). Com 22 kΩ o resultado é cerca de 3,75 V.
- **GND comum** para todo o circuito, inclusive para os sensores.
- **Relé:** o código assume módulo ativo em HIGH. Se o motor ligar na partida ou funcionar ao contrário, mude `RELE_ATIVO_EM_ALTO` para `false` no início do `sketch.ino`.
- **LEDs:** use resistor em série em todos. O LED do pino 6 no Wokwi precisa de cerca de 220 Ω e o LED do motor, alimentado em 12 V, de cerca de 680 Ω a 1 kΩ.

## Como executar

**No Wokwi**
1. Abra o projeto no Wokwi, ou crie um novo com Arduino Nano e cole o `sketch.ino` e o `diagram.json`.
2. Clique em **Start Simulation**.
3. Use o mouse nos botões ou as teclas `1`, `2` e `3`. As teclas são úteis para manter dois botões pressionados ao mesmo tempo.

**No protótipo**
1. Monte o circuito conforme a pinagem e as notas acima.
2. Abra o `Código C++` na Arduino IDE, selecione a placa **Arduino Nano** e o processador correspondente ao seu modelo.
3. Envie o código e teste primeiro **sem carga** (motor e M.R.P. desconectados), observando os LEDs, antes de ligar os atuadores.

## Funcionamento

O programa é uma **máquina de estados** executada no `loop()`. Os tempos são medidos com `millis()`, sem `delay()`, então os sensores são lidos continuamente, inclusive durante a sinalização de erro. O sensor de presença tem **debounce de 50 ms** para evitar leituras duplas.

### Estados

| Estado | O que acontece | Próximo estado |
|---|---|---|
| `OCIOSO` | Espera uma peça. Borda de subida da presença liga o motor. | `TRANSPORTE` |
| `TRANSPORTE` | Peça anda até o sensor de detecção (máximo de 3 s). | `PARANDO` |
| `PARANDO` | Espera 50 ms e desliga o motor. | `ANALISE` |
| `ANALISE` | Esteira parada por 2 s. O indutivo é lido durante todo esse tempo e basta um acionamento para classificar como metal. | `METAL_RETIRADA` ou `NAO_METAL_PAUSA` |
| `METAL_RETIRADA` | LED de metal e M.R.P. ligados por 5 s. | `METAL_RETORNO` |
| `METAL_RETORNO` | M.R.P. desligado, aguarda 2 s o retorno do mecanismo e apaga o LED de metal. | `METAL_FIM` |
| `METAL_FIM` | Aguarda 500 ms e confere se a peça saiu da detecção. | `OCIOSO` |
| `NAO_METAL_PAUSA` | LED de não metal aceso, aguarda 500 ms. | `NAO_METAL_ESTEIRA` |
| `NAO_METAL_ESTEIRA` | Motor liga por 5 s para descartar a peça. | `OCIOSO` |
| `ERRO` | Pisca o LED amarelo conforme o código do erro. | `AGUARDA_LIBERAR` |
| `AGUARDA_LIBERAR` | Espera os dois infravermelhos ficarem livres. | `OCIOSO` |

### Linha do tempo

- **Metal:** entrada → motor liga → detecção → 50 ms → motor para → 2 s de análise → M.R.P. por 5 s → 2 s de retorno → 0,5 s → pronto.
- **Não metal:** entrada → motor liga → detecção → 50 ms → motor para → 2 s de análise → 0,5 s → motor liga por 5 s → pronto.

### Tempos configuráveis

Ficam no início do `sketch.ino`:

| Constante | Valor | Uso |
|---|---|---|
| `T_DEBOUNCE` | 50 ms | Filtro do sensor de presença |
| `T_PISCA` | 250 ms | Tempo aceso e apagado do LED de erro |
| `T_PARADA` | 50 ms | Folga entre a detecção e a parada do motor |
| `T_ANALISE` | 2000 ms | Duração da análise |
| `T_MRP` | 5000 ms | M.R.P. ligado |
| `T_RETORNO` | 2000 ms | Retorno do M.R.P. |
| `T_FIM_METAL` | 500 ms | Conferência final do caminho metálico |
| `T_NAO_METAL_PAUSA` | 500 ms | Pausa antes de religar o motor |
| `T_NAO_METAL_ESTEIRA` | 5000 ms | Esteira ligada para descarte |
| `T_MAX_TRANSPORTE` | 3000 ms | Limite da entrada até a detecção |
| `T_MAX_LIBERACAO` | 2000 ms | Limite para a peça sair da detecção |
| `T_REPETE_ERRO` | 3000 ms | Repetição da sinalização de erro |

## Detecção de erros

Em qualquer erro, o sistema faz **parada segura**: desliga motor, LEDs de resultado e M.R.P., e pisca o LED amarelo. O código do erro é o número de piscadas.

| Piscadas | Erro | Condição de detecção |
|---|---|---|
| 3 | Peça parada na entrada | Em `TRANSPORTE`, 3 s sem chegar à detecção e a presença ainda acionada |
| 4 | Peça não chegou à detecção | Em `TRANSPORTE`, 3 s sem chegar à detecção e a presença já livre |
| 5 | Peça fora do padrão | Em `TRANSPORTE`, presença e detecção acionadas ao mesmo tempo |
| 6 | Dois objetos | Nova peça na entrada com o ciclo em andamento (de `TRANSPORTE` a `NAO_METAL_ESTEIRA`) |
| 7 | Detecção sem entrada | Em `OCIOSO`, detecção acionada sem passar pela entrada |
| 8 | Peça não saiu da detecção | 500 ms após o ciclo metálico, ou 2 s após religar o motor no ciclo não metálico, com a detecção ainda acionada |

Depois do erro, o sistema só volta a operar quando **os dois infravermelhos estiverem livres**. Enquanto a peça não for retirada, a sinalização se repete a cada 3 s.

## Roteiro de testes

Vale para o Wokwi (acionando os botões) e para o protótipo (acionando os sensores com peças ou com a mão). Antes de cada teste, deixe todos os sensores livres e espere os LEDs apagarem.

**Ciclo normal**
1. Acione a presença (botão 2) e libere: o motor liga.
2. Acione a detecção (botão 3) em até 3 s: após 50 ms o motor para.
3. Metal: acione o indutivo (botão 1) em qualquer momento dos 2 s de análise. Não metal: não acione.
4. Libere a detecção antes de o motor religar (não metal) ou antes do fim do ciclo (metal).

**Erros**

| Erro | Como provocar | Esperado |
|---|---|---|
| 3 | Mantenha a presença acionada por mais de 3 s, sem acionar a detecção | 3 piscadas, cerca de 3 s depois |
| 4 | Acione e libere a presença, sem acionar a detecção | 4 piscadas, cerca de 3 s depois |
| 5 | Mantenha a presença acionada e acione a detecção em menos de 3 s | 5 piscadas imediatas |
| 6 | Inicie o ciclo e acione a presença de novo antes de o ciclo terminar | 6 piscadas imediatas |
| 7 | Em repouso, acione a detecção sem ter acionado a presença | 7 piscadas imediatas |
| 8 (não metal) | Acione a presença, depois mantenha a detecção acionada sem acionar o indutivo | 8 piscadas cerca de 4,5 s após acionar a detecção |
| 8 (metal) | Acione a presença, mantenha a detecção acionada e acione o indutivo durante a análise | 8 piscadas cerca de 9,5 s após acionar a detecção |

## Limitações conhecidas

- Depois de `PARANDO`, o sensor de detecção não é verificado até o fim do ciclo, então a retirada da peça durante a análise não gera erro.
- Uma falha do sensor indutivo (por exemplo, preso em HIGH) seria interpretada como metal.
- Não há realimentação do relé nem do M.R.P., então não é possível detectar falha de atuação.
- O módulo de relé é tratado como ativo em HIGH por padrão (ver `RELE_ATIVO_EM_ALTO`).
- Não há botão de parada de emergência no projeto atual.

## Possíveis melhorias

- Erro para peça removida durante a análise.
- Botão de emergência.
- Display que funcione como um tipo de "supervisório".

## Licença

Copyright (c) 2026 Antônio Martins Junior, Cairo Araújo Lima, Guilherme Henrique Faria, João Paulo Braga, Lucas Salles Santana Ferreira e Philipe Eduardo Faria.

Distribuído sob a **Licença MIT**. Você pode usar, copiar, modificar e redistribuir o projeto, inclusive em trabalhos fechados ou comerciais, desde que mantenha o aviso de copyright e o texto da licença. O projeto é fornecido sem nenhuma garantia. Veja o arquivo `LICENSE`.
