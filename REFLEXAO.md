
## Tarefa 1

- A serialização acontece imediatamente antes de cada POST, um por chamada de método.
- `xmlrpc.client.Fault` é uma exceção lançada no cliente quando o servidor retorna um erro durante a execução do método remoto. Diferente de uma exceção Python convencional (que tem tipo, mensagem e traceback), o Fault carrega apenas `faultCode` e `faultString`, porque esses dois campos são o que consegue ser serializado em XML e transmitido pela rede. O mecanismo especial existe porque cliente e servidor são processos independentes — uma exceção Python não atravessa a rede; ela precisa ser convertida em XML pelo servidor e reconstruída como Fault pelo cliente, mantendo a ilusão de chamada local.
- O método `system.listMethods()` relaciona-se com a transparência de acesso, pois oculta do cliente a necessidade de conhecer previamente a interface do servidor. O cliente não precisa ter o contrato definido em código, ele consulta o próprio servidor e descobre os métodos disponíveis em tempo de execução. Isso reforça a ilusão de que o objeto remoto se comporta como um objeto local, cuja interface pode ser inspecionada dinamicamente.
