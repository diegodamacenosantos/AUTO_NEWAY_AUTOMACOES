# auto-neway-automacoes

Um nó para o Node-RED que concentra, num lugar só, o que normalmente exige várias *functions*, *templates*, *switches*, *injects* e nós `mqtt in`/`mqtt out`: **lê** valores (MQTT, fio, variáveis ou horário), decide por **regras**, e **envia** por uma saída, **publica no MQTT** ou **grava numa variável global**. Pensado para automação residencial: sensores de ocupação, luz, termostato, interruptores, comandos de aparelhos, dia e noite.

Neway Automações.

## O que vem no pacote

Na paleta, no grupo **Neway Auto**, aparece um nó:

- **Neway Auto**.

## Instalar

1. Baixe o arquivo `.tgz` deste repositório.
2. No Node-RED, abra o menu (três barras), **Gerenciar paleta**, aba **Instalar**, e envie o arquivo `.tgz`.
3. Confirme. Reinicie o Node-RED se o nó não aparecer na paleta ou se estiver atualizando uma versão antiga.

## Desinstalar

No menu, **Gerenciar paleta**, aba **Nós**, ache `auto-neway-automacoes` e clique em **Remover** (apague antes os nós Neway Auto do fluxo).

## Tipo de ação

O campo **Tipo de ação** (logo abaixo de Nome) define o que o nó faz e quais portas ele tem.

| Tipo | Portas | Para que serve |
|---|---|---|
| **Estado** | só saída | observa algo (MQTT, horário, intervalo, variável) e envia o estado |
| **Leitura de Estado** | entrada e saída | a mensagem que chega é só o gatilho: o nó lê as Entradas e roda as regras |
| **Comando** | só entrada | recebe uma mensagem e publica no MQTT pelas regras |

## Entrada

Uma lista numerada. Cada Entrada tem uma **Propriedade**: de onde ler o valor, como no nó *switch* (`msg.`, `flow.` ou `global.` mais o caminho, por exemplo `msg.payload.temperatura`; `msg.payload` sozinho é o payload inteiro; texto JSON é lido como objeto). Chave ausente na mensagem é ignorada, então mensagens de outros aparelhos no mesmo tópico não interferem. A Entrada nunca grava variável global: isso é feito na Saída.

No **Estado**, cada Entrada tem duas caixas que não funcionam juntas (marcar uma desmarca a outra):

- **MQTT**: o próprio nó se inscreve num tópico (Broker, Tópico com `+` e `#`, QoS), sem precisar de `mqtt in`. A Propriedade é o que será lido da mensagem.
- **Hora | Sol**: abre uma lista com **Intervalo** (a cada N segundos), **Hora fixa** (HH:MM), **Nascer do sol** ou **Pôr do sol**. A Propriedade é o que vai para a Saída (uma mensagem com a hora ou o valor atual de uma variável). Hora fixa e sol têm os **Dias** da semana. Dispara também uma vez ao iniciar. Dispensa o `inject`.

Sem nenhuma das duas caixas, com Propriedade `global.` ou `flow.`, a Entrada observa mudanças da variável.

Na **Leitura de Estado** e no **Comando** a Entrada também pode ser MQTT.

## Saída

Uma lista de regras. **Todas as regras são conferidas**: cada uma que cumpre a condição dispara. Cada regra tem:

| Campo | O que faz |
|---|---|
| Selecionar | qual Entrada a regra observa (ou qualquer uma) |
| Se | condição: **Ignorar** (padrão, sempre dispara), `==`, `!=`, `<`, `<=`, `>`, `>=`, Entre, Contém, Regex, É verdadeiro, É falso, É nulo, Não é nulo, É vazio, Não é vazio e **Hora \| Sol** |
| Valor alvo | com o que comparar (texto ou número) |
| Esperar Por | (só aparece quando o Se tem condição, não em Ignorar) segundos de espera antes de enviar ("X segundos se não mudar"); se o valor deixar de cumprir a condição durante a espera, o envio é cancelado (ocupação com atraso para desligar) |
| Enviar | texto, número, booleano, JSON ou **Entrada** (o valor recebido) |
| Enviar Sempre | (Estado) marcado, envia a cada disparo; desmarcado, ignora o resultado igual ao último enviado |
| MQTT | publica direto no broker (Broker, Tópico, QoS, Retain) |
| Gravar Global | grava o valor numa variável global |
| MSG | envia o valor pela porta de saída (Propriedade `msg.`, `payload` por padrão); só regras com MSG marcada têm porta |

O operador **Hora | Sol** é verdadeiro quando a hora do momento está entre **Das** e **Até**, nos **Dias** marcados. Cada ponta é uma hora fixa ou, na Leitura de Estado e no Comando, também nascer ou pôr do sol, com deslocamento em minutos. O intervalo pode passar da meia-noite.

## Localização

**Latitude** e **Longitude** da casa, usadas para calcular nascer e pôr do sol. Vazias, valem nascer às 6h e pôr às 18h.

## Exemplos

- **Luz Zigbee2MQTT** (Estado): Entrada MQTT com o tópico do aparelho e Propriedade `msg.payload.state`; regras `== ON` e `== OFF` gravando a variável do aparelho.
- **Ocupação com atraso** (Estado): regra `== 1` grava ON na hora; regra `== 0` grava OFF com Esperar Por 15 segundos. Um `1` no meio cancela a espera.
- **Dia e noite** (Estado): uma Entrada Hora | Sol → Nascer do sol com regra Ignorar enviando `Dia` e outra → Pôr do sol enviando `Noite`, gravando a mesma variável.
- **Gatilho a cada 10 s** (Estado): Entrada Hora | Sol → Intervalo 10, regra Ignorar enviando `start` com Enviar Sempre marcado (no lugar de um `inject`).
- **Publicar no MQTT** (Comando): Entrada `msg.payload`, regra Ignorar com MQTT (Broker e Tópico) enviando a Entrada (no lugar de um `mqtt out`).
- **Mensagem fixa** (Leitura de Estado): Entrada `msg._msgid`, regra Ignorar com Enviar JSON (no lugar de um `template`).

## Etiqueta

Abaixo do nó aparece o valor atual das variáveis que ele grava; fica verde quando o valor é `ON` e com anel amarelo durante uma espera.
