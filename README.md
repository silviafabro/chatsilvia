# Chat Multiusuário com Histórico (Java + Sockets + JDBC)

## Requisitos
- Java 17+ e Maven 3.8+

## Como executar
```bash
mvn test                                              # roda os testes
mvn compile exec:java -Dexec.mainClass=server.Servidor    # terminal 1: servidor (porta 5000)
mvn compile exec:java -Dexec.mainClass=client.ClienteChat # terminais 2, 3, 4...: clientes
```
Opcional: `-Dexec.args="localhost 5000"` no cliente, ou `-Dexec.args="6000"` no servidor.

O banco `chat.db` (SQLite) é criado automaticamente na pasta de execução.
O script equivalente está em `sql/criar_tabela.sql`.

## Comandos no cliente
`/usuarios` lista quem está online · `/sair` desconecta.

## Arquitetura
- **model**: `Usuario` (valida nome), `Mensagem` (remetente, conteúdo, horário, formatação).
- **persistence**: `MensagemRepository` — único lugar com SQL/JDBC (`salvar`, `buscarUltimas`).
- **server**: `Servidor` (aceita conexões; uma thread por cliente), `ClienteHandler`
  (Runnable; uma conexão), `GerenciadorClientes` (lista sincronizada + broadcast),
  `Destinatario` (interface que permite testar o broadcast sem rede).
- **client**: `ClienteChat` (thread receptora + laço de teclado).

## Concorrência
- Cada cliente roda em sua `Thread` (`ClienteHandler`), então um cliente lento ou desconectado não bloqueia os outros.
- A lista de clientes é `Collections.synchronizedList`, e operações compostas
  (registrar nome único, broadcast com iteração) usam `synchronized (clientes)`
  para evitar race conditions e `ConcurrentModificationException`.
- O broadcast dentro do monitor garante que todos recebam as mensagens na mesma ordem.
- `ClienteHandler.enviar` é `synchronized` por cliente: duas threads nunca escrevem ao mesmo tempo no mesmo socket.
- `MensagemRepository.salvar/buscarUltimas` são `synchronized` para serializar o acesso ao banco.
- Desconexões (normal ou abrupta) são tratadas no `finally` do `run()`: o cliente é removido, os demais são avisados e o servidor segue funcionando. Falhas de envio no broadcast removem apenas o cliente com problema.

## Testes (JUnit 5)
`UsuarioTest`, `MensagemTest`, `GerenciadorClientesTest` (inclui teste concorrente com 8 threads) e
`MensagemRepositoryTest` — nenhum depende de rede real.
