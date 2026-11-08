# auto-neway-automacoes

Um nó para o Node-RED que concentra, num lugar só, o que normalmente exige várias *functions*, *templates* e *switches*: **recebe** valores (MQTT ou fio), **converte**, **grava em variáveis globais** (opcional), **envia por regras** (com espera e sem repetição) e pode **publicar no MQTT**. Pensado para automação residencial: sensores de ocupação, luz, termostato, interruptores, comandos de aparelhos.

Neway Automações.

## O que vem no pacote

Na paleta, no grupo **auto-neway-automações**, aparece um nó:

- **Auto**.

## Instalar

1. Baixe o arquivo `.tgz` deste repositório.
2. No Node-RED, abra o menu (três barras), **Gerenciar paleta**, aba **Instalar**, e envie o arquivo `.tgz`.
3. Confirme. Se o nó não aparecer na paleta, reinicie o Node-RED.

## Como o nó funciona

Cada mensagem passa por quatro etapas, todas opcionais:

1. **Receber**
2. **Converter**
3. **Gravar**
4. **Enviar** (e **Publicar MQTT**)

### Todos os campos do nó

| Campo | O que faz |
|---|---|
| Nome | rótulo do nó no fluxo (opcional) |
| Receber por: MQTT | marcado, o nó se inscreve sozinho em um tópico MQTT e recebe o que chegar nele |
| Broker | o servidor MQTT (nó de configuração `mqtt-broker` do Node-RED) |
| Tópico | o tópico a ouvir; aceita `+` (um nível) e `#` (todos os níveis) |
| QoS | garantia de entrega do MQTT: 0, 1 ou 2 |
| Receber por: MSG | marcado, o nó ganha uma porta de entrada para ligar por fio, recebendo mensagens de outros nós |
| Entrada | a lista de valores lidos da mensagem; cada linha tem Propriedade e Global (abaixo) |
| Conversão | regras que traduzem o valor recebido para o valor que será gravado (opcional) |
| Saída | regras que decidem quando e o que enviar pelas saídas |
| Latitude, Longitude | posição da casa, usada só pelas condições de Horário com `nascer` e `por` (do sol) |
| Publicar: MQTT | marcado, o nó também publica as saídas direto num broker, sem `mqtt out` |
| Publicar: Broker, Tópico, QoS | o servidor, o tópico e a garantia de entrega da publicação |
| Publicar: Retain | o broker guarda a última mensagem e a entrega a quem se inscrever depois |

### Receber por

Duas caixas, que podem ficar marcadas juntas:

- **MQTT**: o próprio nó se inscreve no tópico (Broker, Tópico e QoS; aceita `+` e `#`). Não precisa de nó `mqtt in`.
- **MSG**: o nó tem uma porta de entrada para ligar por fio. Sem nenhuma caixa marcada, o nó fica sem porta de entrada.
- Sem broker escolhido, o nó escuta apenas a entrada por fio. Com broker e sem tópico, o nó avisa "sem tópico".

O broker é o mesmo nó de configuração `mqtt-broker` do Node-RED. O payload pode ser objeto, texto JSON ou Buffer.

### Entradas

Uma lista numerada. Cada linha tem:

- **Propriedade**: de onde ler o valor, como no nó *switch* (`msg.`, `flow.` ou `global.` mais o caminho, por exemplo `msg.payload.temperatura`). Chave ausente na mensagem é ignorada, então mensagens de outros aparelhos no mesmo tópico não interferem.
- **Global** (opcional): o nome da variável global onde o valor é gravado. Sem Global, a entrada passa direto, sem gravação e sem a regra de não repetir.

### Converter (opcional)

Regras que decidem o valor a gravar: **Converter** (qual entrada), condição, **valor alvo** e **Gravar** (texto, número ou o valor como veio). A primeira regra que cumprir define o valor. Exemplos: `1` grava `Dia` e `0` grava `Noite`; `ONLINE` grava `ON`.

Atenção: se houver regras de conversão e nenhuma cumprir, o valor é ignorado.

### Saída

Regras como no nó *switch*. Todas são conferidas, e a regra **senão** só dispara se nenhuma anterior cumpriu. Cada regra tem:

- **Entrada alvo**: qual entrada observar, ou qualquer uma;
- **Propriedade de saída**: onde colocar o valor (`msg.payload` por padrão, ou `msg.payload.chave`, `flow.`, `global.`);
- **Condição** e **valor alvo**;
- **Esperar X segundos**: com 0, envia na hora; acima de 0, espera e **cancela** se chegar um valor que deixa de cumprir a condição (ideal para ocupação: o OFF só sai depois de 15 s sem presença);
- **Enviar**: texto, número, verdadeiro/falso, JSON, alternar, ou o valor recebido.

### Condições

`==`, `!=`, `<`, `<=`, `>`, `>=`, entre, contém, regex, é verdadeiro, é falso, é nulo, não é nulo, é vazio, não é vazio, **senão** e **Horário**.

**Horário**: verdadeiro quando a hora do momento está entre **Das** e **Até**, nos dias da semana marcados. Cada ponta é `HH:MM`, `nascer` ou `por`, com deslocamento em minutos (`por-30`, `nascer+15`). Intervalos que passam da meia-noite valem. Nascer e pôr do sol usam a Latitude e Longitude do nó.

### Sem repetição

Uma saída só é enviada se for diferente da última enviada na mesma corrente. Logo após um Deploy, a primeira mensagem igual ao valor que já está na global não é reenviada.

### Publicar MQTT

O nó pode publicar direto no broker (tópico, QoS e retain), sem `mqtt out`.

### Etiqueta

Mostra `nome=valor` das globals; fica verde quando a única entrada está em `ON` e ganha um anel amarelo durante uma espera.

## Receitas

**Sensor de ocupação** (o aparelho manda `1`/`0`): entrada com Global `ocupacao_sala`; conversão `1` → `ON` e `0` → `OFF`; saída 1 `== ON` na hora, saída 2 `== OFF` esperando 15 s, as duas enviando o valor recebido.

**Luz por MQTT:** broker, tópico do aparelho, propriedade da chave de estado, Global da luz; conversão `ON` e `OFF` pelo valor recebido; saídas `== ON` e `== OFF`.

**Luminosidade:** saídas `> 5` → `claro` e `<= 5` → `escuro`.

**Comando para um aparelho:** só a porta MSG, sem Global; uma regra por comando; Publicar MQTT no tópico de comando do aparelho.

## Problemas comuns

| Sintoma | O que fazer |
|---|---|
| Nada sai | conferir se alguma regra de conversão cumpre o valor recebido |
| Valor não repete | é a regra de não repetição: a saída só vai se mudou |
| Globals somem depois de reiniciar | o contexto global fica em memória; ative o armazenamento em arquivo no `settings.js` se precisar manter |
| Editor mostra o nó como era antes da atualização | reiniciar o Node-RED |

## Atualizar

Baixe o `.tgz` atual e instale do mesmo jeito (Gerenciar paleta, Instalar, enviar o arquivo) e reinicie o Node-RED.

## Desinstalar

1. Apague **todos os nós Auto** dos fluxos e faça o **Deploy**.
2. Menu, **Gerenciar paleta**, aba **Nós**, ache `auto-neway-automacoes` e clique em **Remover**.
