# Spring Boot com Fila SQS e Localstack

Nessa POC, simularemos como podemos integrar Spring boot com SQS com o uso de LocalStack.


- LocalStack
    - LocalStack é uma pilha de nuvem local totalmente funcional. É um emulador de serviço em nuvem executado em um único contêiner em seu notebook ou ambiente de CI. Com LocalStack, você pode executar seus aplicativos AWS ou Lambdas inteiramente em sua máquina local sem se conectar a um provedor de nuvem remoto! LocalStack oferece suporte a um número crescente de serviços AWS, como AWS Lambda, S3, Dynamodb, Kinesis, SQS, SNS.

- Configurar LocalStack
    - Usaremos LocalStack para simular AWS SQS em um sistema local. Usaremos um contêiner docker para executar o LocalStack. Estaremos usando docker-compose para executar o LocalStack.

    - docker-compose.yml

```
versão: '3.7' 
serviços: 
  aws: 
    imagem: ' localstack/localstack ' 
    container_name: 'localstack' 
    ambiente: 
      - SERVICES=sqs 
      - DEFAULT_REGION=us-east-1 
      - AWS_DEFAULT_REGION=us-east-1 
      - DEBUG=1 
      - DATA_DIR= /tmp/localstack/ 
    portas de dados: 
      - '4566:4566'
```

- Vamos iniciar o serviço LocalStack.

```
docker-compose up -d

```

- Agora, vamos criar uma instância SQS no LocalStack para trabalharmos.

- Primeiro, exporemos o shell script do contêiner do docker para que possamos ter acesso ao LocalStack.

```
docker exec -it localstack sh

```

- O comando acima irá expor o script shell do contêiner docker.

- Assim que tivermos acesso ao shell script do contêiner docker, podemos executar o comando abaixo para criar uma fila.

```
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name sample-queue.fifo --attributes FifoQueue=true

```

Você pode receber um erro ao executar o comando acima. Se você receber algum erro, execute primeiro o comando abaixo para configurar as credenciais do AWS. Isso precisa ser executado apenas uma vez pela primeira vez.
#aws configure

- Agora temos uma fila SQS de amostra pronta para usar.

# Microsserviço Spring Boot

- Veremos dois microsserviços diferentes: editor e assinante.

# Editor

- Precisaremos da dependência abaixo em nosso projeto para obter as bibliotecas necessárias para trabalhar com SQS.

```
<dependency> 
    <groupId>org.springframework.cloud</groupId> 
    <artifactId>spring-cloud-starter-aws-messaging</artifactId> 
    <version>2.2.1.RELEASE</version> 
</dependency>

```

- Usaremos as propriedades abaixo em nosso arquivo application.yml.

```
cloud:
  aws:
    stack:
      auto: false
    region:
      static: us-east-1
    credentials:
      access-key: ANUJDEKAVADIYAEXAMPLE
      secret-key: 2QvM4/Tdmf38SkcD/qalvXO4EXAMPLEKEY
    end-point:
      uri: http://localhost:4566

```

- Vamos configurar o cliente Amazon SQS para se conectar ao LocalStack.

```
@Configuration 
public class SQSConfig { 

    @Value("${cloud.aws.region.static}") 
    private String região; 

    @Value("${cloud.aws.credentials.access-key}") 
    private String accessKeyId; 

    @Value("${cloud.aws.credentials.secret-key}") 
    private String secretAccessKey; 

    @Value("${cloud.aws.end-point.uri}") 
    private String sqsUrl; 

    @Bean 
    @Primary 
    public AmazonSQSAsync amazonSQSAsync() { 
        return AmazonSQSAsyncClientBuilder. padrão () 
                .withEndpointConfiguration(new AwsClientBuilder.EndpointConfiguration(sqsUrl, região)) 
                .withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials(accessKeyId, secretAccessKey))) 
                .build(); 
    } 

    @Bean 
    public QueueMessagingTemplate queueMessagingTemplate() { 
        return new QueueMessagingTemplate(amazonSQSAsync()); 
    } 

    @Bean 
    public ObjectMapper objectMapper() { 
        return new ObjectMapper(); 
    } 
}

```

 - Adicione um editor para publicar mensagens no SQS.

 ```
@Service 
public class SQSEventPublisher { 

    private static final Logger LOGGER = LoggerFactory. getLogger (SQSEventPublisher.class); 

    @Autowired 
    AmazonSQS privado amazonSQS; 

    @Autowired 
    private ObjectMapper objectMapper; 

    public voidpublishEvent(mensagem JsonNode) { 
        LOGGER .info("Gerando evento: {}", mensagem); 
        SendMessageRequest sendMessageRequest = null; 
        tente { 
            sendMessageRequest = new SendMessageRequest().withQueueUrl("http://localhost:4566/000000000000/sample-queue.fifo") 
                    .withMessageBody(objectMapper.writeValueAsString(message)) 
                    .withMessageGroupId("Amostra de mensagem") 
                    .withMessageDeduplicationId( UUID.randomUUID () . toString()); 
            amazonSQS.sendMessage(sendMessageRequest); 
            LOGGER .info("O evento foi publicado no SQS."); 
        } catch (JsonProcessingException e) { 
            LOGGER .error("JsonProcessingException e: {} e stacktrace: {}", e.getMessage(), e); 
        } catch (Exception e) { 
            LOGGER .error("Ocorreu uma exceção ao enviar o evento para sqs: {} e stacktrace; {}", e.getMessage(), e); 
        } } 

    }

```

- O URL da fila mudará com base na sua versão do LocalStack. Portanto, pegue o URL mostrado nos logs do LocalStack.

- Adicione um controlador para expor a API da web para aceitar dados a serem publicados no SQS.

```
@RestController 
public class PublisherController { 

    @Autowired 
    private SQSEventPublisher sqsEventPublisher; 

    @PostMapping("/sendMessage") 
    public ResponseEntity sendMessage(@RequestBody JsonNode jsonNode) { 
        sqsEventPublisher.publishEvent(jsonNode); 
        retornar ResponseEntity. ok ().build(); 
    } 
}

```

# Ouvinte

- Precisaremos da dependência abaixo em nosso projeto para obter as bibliotecas necessárias para trabalhar com SQS.

```
<dependency> 
    <groupId>org.springframework.cloud</groupId> 
    <artifactId>spring-cloud-starter-aws-messaging</artifactId> 
    <version>2.2.1.RELEASE</version> 
</dependency>

```

- Usaremos as propriedades abaixo em nosso arquivo application.yml.

```
cloud:
  aws:
    stack:
      auto: false
    region:
      static: us-east-1
      auto: false
    credentials:
      access-key: ANUJDEKAVADIYAEXAMPLE
      secret-key: 2QvM4/Tdmf38SkcD/qalvXO4EXAMPLEKEY
    queue:
      uri: http://localhost:4566
      name: sample-queue.fifo

server:
  port: 8081

```

- Vamos configurar o cliente Amazon SQS para se conectar ao LocalStack.

```
@Configuration 
public class SQSConfig { 

    @Value("${cloud.aws.region.static}") 
    private String região; 

    @Value("${cloud.aws.credentials.access-key}") 
    private String accessKeyId; 

    @Value("${cloud.aws.credentials.secret-key}") 
    private String secretAccessKey; 

    @Value("${cloud.aws.queue.uri}") 
    private String sqsUrl; 
    @Bean 
    @Primary 
    public AmazonSQSAsync amazonSQSAsync() { 
        return AmazonSQSAsyncClientBuilder. padrão () 
                .withEndpointConfiguration(new AwsClientBuilder.EndpointConfiguration(sqsUrl, região)) 
                .withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials(accessKeyId, secretAccessKey))) 
                .build(); 
    } 

    @Bean 
    public QueueMessagingTemplate queueMessagingTemplate() { 
        return new QueueMessagingTemplate(amazonSQSAsync()); 
    } 

    @Bean 
    protected MessageConverter messageConverter(ObjectMapper objectMapper) { 

        Conversor MappingJackson2MessageConverter = new MappingJackson2MessageConverter(); 
        conversor.setObjectMapper(objectMapper); 
        conversor.setSerializedPayloadClass(String.class); 
        conversor.setStrictContentTypeMatch(falso); 
        conversor de retorno; 
    } 
}

```

- Vamos adicionar um ouvinte.

```
@Component 
public class SQSConsumer { 

    private static final Logger LOGGER = LoggerFactory. getLogger (SQSConsumer.class); 

    @SqsListener("${cloud.aws.queue.name}") 
    public void recebeMessage(Map<String, Object> mensagem) { 
        LOGGER .info("Mensagem SQS recebida: {}", mensagem); 
    } 
}

```