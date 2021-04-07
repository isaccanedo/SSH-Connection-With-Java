## Security - Conexão SSH com Java

# 1. Introdução
SSH, também conhecido como Secure Shell ou Secure Socket Shell, é um protocolo de rede que permite que um computador se conecte com segurança a outro computador em uma rede não segura. Neste tutorial, mostraremos como estabelecer uma conexão com um servidor SSH remoto com Java usando as bibliotecas JSch e Apache MINA SSHD.

Em nossos exemplos, primeiro abriremos a conexão SSH e, em seguida, executaremos um comando, leremos a saída e escreveremos no console e, finalmente, fecharemos a conexão SSH. Manteremos o código de amostra o mais simples possível.

# 2. JSch
JSch é a implementação Java do SSH2 que nos permite conectar a um servidor SSH e usar o encaminhamento de porta, encaminhamento X11 e transferência de arquivos. Além disso, é licenciado sob a licença do estilo BSD e nos fornece uma maneira fácil de estabelecer uma conexão SSH com Java.

Primeiro, vamos adicionar a dependência JSch Maven ao nosso arquivo pom.xml:

```
<dependency>
    <groupId>com.jcraft</groupId>
    <artifactId>jsch</artifactId>
    <version>0.1.55</version>
</dependency>
```

### 2.1. Implementação
Para estabelecer uma conexão SSH usando JSch, precisamos de um nome de usuário, senha, URL do host e porta SSH. A porta SSH padrão é 22, mas pode acontecer de configurarmos o servidor para usar outra porta para conexões SSH:

```
public static void listFolderStructure(String username, String password, 
  String host, int port, String command) throws Exception {
    
    Session session = null;
    ChannelExec channel = null;
    
    try {
        session = new JSch().getSession(username, host, port);
        session.setPassword(password);
        session.setConfig("StrictHostKeyChecking", "no");
        session.connect();
        
        channel = (ChannelExec) session.openChannel("exec");
        channel.setCommand(command);
        ByteArrayOutputStream responseStream = new ByteArrayOutputStream();
        channel.setOutputStream(responseStream);
        channel.connect();
        
        while (channel.isConnected()) {
            Thread.sleep(100);
        }
        
        String responseString = new String(responseStream.toByteArray());
        System.out.println(responseString);
    } finally {
        if (session != null) {
            session.disconnect();
        }
        if (channel != null) {
            channel.disconnect();
        }
    }
}
```

Como podemos ver no código, primeiro criamos uma sessão de cliente e a configuramos para conexão com nosso servidor SSH. Em seguida, criamos um canal de cliente usado para se comunicar com o servidor SSH, onde fornecemos um tipo de canal - neste caso, exec, o que significa que passaremos comandos shell para o servidor.

Além disso, devemos definir o fluxo de saída para nosso canal onde a resposta do servidor será gravada. Depois de estabelecermos a conexão usando o método channel.connect(), o comando é passado e a resposta recebida é gravada no console.

Vamos ver como usar os diferentes parâmetros de configuração que o JSch oferece:

- StrictHostKeyChecking - indica se o aplicativo verificará se a chave pública do host pode ser encontrada entre os hosts conhecidos. Além disso, os valores dos parâmetros disponíveis são ask, yes e no, onde ask é o padrão. Se definirmos essa propriedade como yes, JSch nunca adicionará automaticamente a chave do host ao arquivo known_hosts e se recusará a conectar-se a hosts cuja chave do host foi alterada. Isso força o usuário a adicionar manualmente todos os novos hosts. Se definirmos como não, JSch adicionará automaticamente uma nova chave de host à lista de hosts conhecidos;
- compression.s2c - especifica se deve ser usada compactação para o fluxo de dados do servidor para nosso aplicativo cliente. Os valores disponíveis são zlib e nenhum onde o segundo é o padrão
- compression.c2s - especifica se deve ser usada compactação para o fluxo de dados na direção cliente-servidor. Os valores disponíveis são zlib e none, onde o segundo é o padrão.

É importante fechar a sessão e o canal SFTP após o término da comunicação com o servidor para evitar vazamentos de memória.

# 3. Apache MINA SSHD

Apache MINA SSHD fornece suporte SSH para aplicativos baseados em Java. Esta biblioteca é baseada no Apache MINA, uma biblioteca IO assíncrona escalonável e de alto desempenho.

Vamos adicionar a dependência Apache Mina SSHD Maven:

<dependency>
    <groupId>org.apache.sshd</groupId>
    <artifactId>sshd-core</artifactId>
    <version>2.5.1</version>
</dependency>

### 3.1. Implementação
Vamos ver o exemplo de código de conexão ao servidor SSH usando Apache MINA SSHD:

```
public static void listFolderStructure(String username, String password, 
  String host, int port, long defaultTimeoutSeconds, String command) throws IOException {
    
    SshClient client = SshClient.setUpDefaultClient();
    client.start();
    
    try (ClientSession session = client.connect(username, host, port)
      .verify(defaultTimeoutSeconds, TimeUnit.SECONDS).getSession()) {
        session.addPasswordIdentity(password);
        session.auth().verify(defaultTimeoutSeconds, TimeUnit.SECONDS);
        
        try (ByteArrayOutputStream responseStream = new ByteArrayOutputStream(); 
          ClientChannel channel = session.createChannel(Channel.CHANNEL_SHELL)) {
            channel.setOut(responseStream);
            try {
                channel.open().verify(defaultTimeoutSeconds, TimeUnit.SECONDS);
                try (OutputStream pipedIn = channel.getInvertedIn()) {
                    pipedIn.write(command.getBytes());
                    pipedIn.flush();
                }
            
                channel.waitFor(EnumSet.of(ClientChannelEvent.CLOSED), 
                TimeUnit.SECONDS.toMillis(defaultTimeoutSeconds));
                String responseString = new String(responseStream.toByteArray());
                System.out.println(responseString);
            } finally {
                channel.close(false);
            }
        }
    } finally {
        client.stop();
    }
}
```

Ao trabalhar com o Apache MINA SSHD, temos uma sequência de eventos bastante semelhante à do JSch. Primeiro, estabelecemos uma conexão com um servidor SSH usando a instância da classe SshClient. Se o inicializarmos com SshClient.setupDefaultClient(), poderemos trabalhar com a instância que possui uma configuração padrão adequada para a maioria dos casos de uso. Isso inclui cifras, compressão, MACs, trocas de chaves e assinaturas.

Depois disso, criaremos ClientChannel e anexaremos o ByteArrayOutputStream a ele, para que possamos usá-lo como um fluxo de resposta. Como podemos ver, o SSHD requer tempos limites definidos para cada operação. Também nos permite definir quanto tempo ele aguardará pela resposta do servidor após o comando ser passado usando o método Channel.waitFor().

É importante observar que o SSHD gravará a saída completa do console no fluxo de resposta. JSch fará isso apenas com o resultado da execução do comando.

A documentação completa sobre Apache Mina SSHD está disponível no repositório GitHub oficial do projeto.

# 4. Conclusão
Este artigo ilustrou como estabelecer uma conexão SSH com Java usando duas das bibliotecas Java disponíveis - JSch e Apache Mina SSHD. Também mostramos como passar o comando para o servidor remoto e obter o resultado da execução.


