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
| **Leitura** | entrada e saída | a mensagem que chega é só o gatilho: o nó lê as Entradas e roda as regras |
| **Comando** | só entrada | recebe uma mensagem e publica no MQTT pelas regras |

## Broker MQTT

Onde houver o campo **Broker** (Entrada e Saída), o botão **+** cadastra um broker novo e o **lápis** edita o escolhido, com a mesma janela do nó MQTT do Node-RED.

## Entrada

Uma lista numerada. Cada Entrada tem uma **Propriedade**: de onde ler o valor, como no nó *switch* (`msg.`, `flow.` ou `global.` mais o caminho, por exemplo `msg.payload.temperatura`; `msg.payload` sozinho é o payload inteiro; texto JSON é lido como objeto). Chave ausente na mensagem é ignorada, então mensagens de outros aparelhos no mesmo tópico não interferem. A Entrada nunca grava variável global: isso é feito na Saída.

No **Estado**, cada Entrada tem três caixas que não funcionam juntas (marcar uma desmarca as outras):

- **Ping**: pede o **Endereço** (IP ou nome, sem `http://`), a **Porta** (opcional) e o **Intervalo** em segundos (padrão 10, mínimo 5). Com a Porta vazia é o ping comum; com uma porta (por exemplo `443`) o nó mede o tempo para abrir a conexão, útil para saber se um site ou a internet estão no ar quando a rede bloqueia ping. A Propriedade recebe o **tempo de resposta em milissegundos**, número inteiro; **`0` significa sem resposta** em 2 segundos (uma resposta abaixo de 1 ms vale 1). Na Saída: `> 0` está no ar, `== 0` caiu (com Esperar Por para não acusar uma falha isolada), `> 200` está lento. Depende de o Node-RED ter o comando `ping` e de a rede permitir ICMP: algumas redes bloqueiam ping para a internet, e então um endereço de fora sempre vale 0.

- **MQTT**: o próprio nó se inscreve num tópico (Broker, Tópico com `+` e `#`, QoS), sem precisar de `mqtt in`. A Propriedade é o que será lido da mensagem.
- **Hora | Sol**: abre uma lista com **Intervalo** (a cada N segundos), **Hora fixa** (HH:MM), **Nascer do sol** ou **Pôr do sol**. A Propriedade é o que vai para a Saída (uma mensagem com a hora ou o valor atual de uma variável). Hora fixa e sol têm os **Dias** da semana. Dispara também uma vez ao iniciar. Dispensa o `inject`.

Sem nenhuma das caixas, com Propriedade `global.` ou `flow.`, a Entrada observa mudanças da variável.

No **Comando** a Entrada também pode ser MQTT (ela é alimentada só pelo tópico; a mensagem do fio alimenta as outras Entradas). A **Leitura** não tem MQTT na Entrada: ela lê `msg.` (da mensagem que dispara o nó), `flow.` ou `global.`; para receber de um tópico use o **Estado**.

## Saída

Uma lista de regras. **Todas as regras são conferidas**: cada uma que cumpre a condição dispara. Cada regra tem:

| Campo | O que faz |
|---|---|
| Selecionar | qual Entrada a regra observa (ou qualquer uma) |
| Se | condição: **Ignorar** (padrão, sempre dispara), `==`, `!=`, `<`, `<=`, `>`, `>=`, Entre, Contém, Regex, É verdadeiro, É falso, É nulo, Não é nulo, É vazio, Não é vazio (número e verdadeiro/falso contam como não vazios, inclusive `0` e `false`) e **Hora \| Sol** |
| Valor alvo | com o que comparar (texto ou número) |
| E | abre mais um degrau de condição (veja **Escada de condições**) |
| Esperar Por | (só aparece quando o Se tem condição, não em Ignorar) segundos de espera antes de enviar ("X segundos se não mudar"); se o valor deixar de cumprir a condição durante a espera, o envio é cancelado (ocupação com atraso para desligar); com `global.` ou `flow.`, o nó relê a variável no fim da espera. A espera continua depois de um deploy ou reinício do Node-RED |
| Enviar | texto, número, booleano, JSON ou **Entrada** (o valor recebido) |
| Enviar Sempre | (Estado) marcado, envia a cada disparo; desmarcado, a regra ignora o valor igual ao último que ela mesma enviou (volta a enviar depois que outra regra mudar o estado) |
| MQTT | publica direto no broker (Broker, Tópico, QoS, Retain) |
| Gravar Global | grava o valor numa variável global |
| MSG | envia o valor pela porta de saída (Propriedade `msg.`, `payload` por padrão); só regras com MSG marcada têm porta |

O operador **Hora | Sol** é verdadeiro quando a hora do momento está entre **Das** e **Até**, nos **Dias** marcados. Cada ponta é uma hora fixa ou, na Leitura e no Comando, também nascer ou pôr do sol, com deslocamento em minutos. O intervalo pode passar da meia-noite.

### Escada de condições

Marcando **E** abaixo do **Se**, a regra ganha mais um degrau, e cada degrau pode abrir outro. A regra só vale se **todos** os degraus valerem.

- **Só o primeiro degrau dispara** (o Selecionar da regra, que aceita qualquer tipo de Entrada).
- Os degraus **E** apenas **conferem o valor atual** de Entradas `global.` ou `flow.`, com comparações (`==`, `!=`, `<`, `<=`, `>`, `>=`, Entre, Contém, Regex, É verdadeiro, É falso, É vazio, Não é vazio). Variável que não existe torna a condição falsa.
- Todas as regras leem as variáveis antes de qualquer uma gravar.
- Uma mudança numa variável de degrau **nunca dispara** a regra; quando a escada passa a valer ou deixa de valer, a regra esquece o que já enviou, para o mesmo comando voltar a valer (conferido a cada 1 s e a cada disparo). Um valor que muda sem alterar o resultado (uma temperatura que oscila acima do limite) não conta.
- Com **Esperar Por**, o nó confere a escada de novo no fim da espera; uma mudança que vai e volta no meio da espera não é vista.
- Com Selecionar em **Qualquer entrada** não há E.

Exemplo, liga e desliga por voz: regra 1: comando `== quarto` **E** interruptor `== OFF` enviando `ON`; regra 2: comando `== quarto` **E** interruptor `== ON` enviando `OFF`.

Atenção em "presença **E** noite": se anoitecer com a pessoa já no cômodo, nada acontece até a presença mudar.

## Localização

**Latitude** e **Longitude** da casa, usadas para calcular nascer e pôr do sol. Vazias, valem nascer às 6h e pôr às 18h.

## Exemplos

- **Luz Zigbee2MQTT** (Estado): Entrada MQTT com o tópico do aparelho e Propriedade `msg.payload.state`; regras `== ON` e `== OFF` gravando a variável do aparelho.
- **Ocupação com atraso** (Estado): regra `== 1` grava ON na hora; regra `== 0` grava OFF com Esperar Por 15 segundos. Um `1` no meio cancela a espera.
- **Dia e noite** (Estado): uma Entrada Hora | Sol → Nascer do sol com regra Ignorar enviando `Dia` e outra → Pôr do sol enviando `Noite`, gravando a mesma variável.
- **Gatilho a cada 10 s** (Estado): Entrada Hora | Sol → Intervalo 10, regra Ignorar enviando `start` com Enviar Sempre marcado (no lugar de um `inject`).
- **Aparelho no ar** (Estado): Entrada Ping com o IP; regra `> 0` enviando `ON` e regra `== 0` com Esperar Por 30 enviando `OFF`, gravando a mesma variável.
- **Publicar no MQTT** (Comando): Entrada `msg.payload`, regra Ignorar com MQTT (Broker e Tópico) enviando a Entrada (no lugar de um `mqtt out`).
- **Mensagem fixa** (Leitura): Entrada `msg._msgid`, regra Ignorar com Enviar JSON (no lugar de um `template`).

## Etiqueta

Abaixo do nó aparece o valor atual das variáveis que ele grava; fica verde quando o valor é `ON` e com anel amarelo durante uma espera.
