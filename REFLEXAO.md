
## Tarefa 1

- A serialização acontece imediatamente antes de cada POST, um por chamada de método.
- `xmlrpc.client.Fault` é uma exceção lançada no cliente quando o servidor retorna um erro durante a execução do método remoto. Diferente de uma exceção Python convencional (que tem tipo, mensagem e traceback), o Fault carrega apenas `faultCode` e `faultString`, porque esses dois campos são o que consegue ser serializado em XML e transmitido pela rede. O mecanismo especial existe porque cliente e servidor são processos independentes — uma exceção Python não atravessa a rede; ela precisa ser convertida em XML pelo servidor e reconstruída como Fault pelo cliente, mantendo a ilusão de chamada local.
- O método `system.listMethods()` relaciona-se com a transparência de acesso, pois oculta do cliente a necessidade de conhecer previamente a interface do servidor. O cliente não precisa ter o contrato definido em código, ele consulta o próprio servidor e descobre os métodos disponíveis em tempo de execução. Isso reforça a ilusão de que o objeto remoto se comporta como um objeto local, cuja interface pode ser inspecionada dinamicamente.

## Tarefa 2

- Marshalling (cliente) converte os parâmetros em um formato transmissível pela rede. Dispatching (servidor) recebe os bytes, identifica qual método foi chamado e encaminha para a função correta. Unmarshalling (servidor) converte os bytes de volta para os tipos nativos da linguagem, para passar à função real. Isso já está nos comentários do código.
- A diferença é que o JSON é um formato feito para leitura humana e o protobuf para comunicação entre máquinas já que é binário. Isso implica que o JSON vai possuir um overhead maior ( mais verboso e maior), implicando em uma desserialização e serialização mais lenta ( marshalling e unmarshalling) em relação ao Protobuf, sendo assim, ele será menos performático.
- O TCP é um protocolo de stream de bytes, não preserva fronteiras de mensagem, podendo entregar dois envios juntos ou um envio fragmentado em partes. Sem framing, o servidor não saberia onde uma mensagem termina e a próxima começa, podendo tentar fazer unmarshalling de bytes incompletos ou de duas mensagens fundidas, corrompendo os dados ou travando esperando bytes que já chegaram mas foram interpretados como parte da mensagem anterior.

## Tarefa 3

- `201` carrega semântica adicional — um proxy ou cache genérico ao receber 201 sabe que:

  - Um novo recurso foi criado
  - O header Location deve indicar onde esse recurso está (ex: Location: `/produtos/42`)
  - Não deve cachear a resposta

Com `200` o cliente não saberia distinguir "busquei algo" de "criei algo". A semântica do código permite que qualquer intermediário na rede tome decisões corretas sem entender o domínio da aplicação.
- O _produtos em memória precisa ir para um banco de dados para garantir persistência. Isso não viola o stateless de Fielding porque os dados persistidos são estado do recurso, não estado de sessão. O que tornaria o servidor não-stateless seria guardar contexto da interação com o cliente entre requisições, como sessões ou fluxos de múltiplos passos dependentes.
- A interface uniforme do REST é mais clara porque:

  - O verbo HTTP (POST, GET, DELETE) já comunica a intenção
  - A URL (/calculos) identifica o recurso
  - O corpo JSON descreve os dados

Qualquer cliente genérico entende o contrato sem documentação extra. No RPC, proxy.calcular poderia fazer qualquer coisa, só quem escreveu o servidor sabe.


## Tarefa 4

- Ter um contrato explícito no .proto garante que cliente e servidor estejam sincronizados de forma verificável. O protoc gera código tipado para ambos os lados, então se o servidor mudar o campo resultado de double para string sem atualizar o .proto, o erro é detectado em tempo de compilação, o cliente simplesmente não compila.
No REST não existe essa garantia. O contrato é uma convenção implícita que vive em documentação externa (Swagger, README) que pode estar desatualizada. Se o servidor mudar o tipo do campo, o cliente só descobre em tempo de execução, recebe uma string onde esperava um número, podendo gerar uma exceção ou, pior, um cálculo silenciosamente errado sem nenhum aviso.

- São parecidos mas não equivalentes, o gRPC é mais rico semanticamente. O HTTP 400 é um código genérico que cobre qualquer erro do cliente, independente da causa. A explicação específica fica no corpo JSON, que é texto livre e não padronizado, cada API inventa seu próprio formato. O gRPC possui códigos específicos para cada situação: INVALID_ARGUMENT para campos inválidos, OUT_OF_RANGE para valores fora do intervalo esperado, NOT_FOUND para recurso inexistente, PERMISSION_DENIED para falta de permissão. O código em si já carrega a semântica, qualquer cliente genérico sabe exatamente o tipo do erro sem precisar parsear o corpo da resposta, tornando o tratamento de erros mais preciso e previsível.
- Os dois são mecanismos de propagação de erro entre processos, mas o grpc.RpcError oferece mais informação estruturada.
O xmlrpc.client.Fault carrega apenas dois campos: faultCode (inteiro genérico) e faultString (texto livre). O código é arbitrário, cada servidor define os seus próprios números sem padronização, então o cliente precisa conhecer a convenção específica daquele servidor para interpretar o erro.
O grpc.RpcError carrega um StatusCode padronizado do protocolo (como INVALID_ARGUMENT, NOT_FOUND, PERMISSION_DENIED) mais uma mensagem descritiva. O StatusCode é parte da especificação do gRPC, qualquer cliente, em qualquer linguagem, interpreta NOT_FOUND da mesma forma sem precisar conhecer convenções do servidor específico.
A diferença prática é que com grpc.RpcError o cliente pode tomar decisões programáticas baseadas no código, por exemplo, só fazer retry em UNAVAILABLE e não em INVALID_ARGUMENT. Com Fault o cliente teria que parsear o faultString como texto para inferir isso, o que é frágil e não padronizado.