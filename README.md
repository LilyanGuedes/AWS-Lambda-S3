## 📖 Sobre Este Repositório

Este repositório é o resultado da minha jornada de aprendizado no desafio da DIO sobre **automatização de tarefas com AWS Lambda e S3**. Aqui documentei tudo o que aprendi, os desafios que enfrentei e os insights que obtive durante a prática.

---

## 🎓 O Que Eu Aprendi

### 1. **Conceito de Serverless Computing**

Antes deste desafio, eu tinha uma ideia vaga sobre o que era "serverless". Agora entendo que:

**Aprendi que** serverless não significa "sem servidor", mas sim que **eu não preciso me preocupar com a infraestrutura**. A AWS cuida de toda a parte de provisionamento, escala e manutenção dos servidores.

**O mais interessante** foi descobrir que pago apenas pelo tempo de execução do código (em milissegundos!), o que torna a solução muito econômica para automações pontuais.

### 2. **AWS Lambda - O Coração da Automação**

**Descobri que** Lambda é como ter um robô que fica esperando um evento acontecer para executar uma tarefa específica.

**Principais aprendizados:**
- Lambda executa código em resposta a **eventos** (triggers)
- Suporta várias linguagens (Python, Node.js, Java, Go, etc.)
- Tem limite de **15 minutos** de execução por invocação
- Posso configurar memória de 128MB até 10GB
- **Cold start**: a primeira execução pode ser mais lenta (aprendi isso na prática!)

**Insight importante**: Quanto mais memória eu aloco, mais CPU a função recebe proporcionalmente, o que pode compensar o custo em execuções intensivas.

### 3. **Amazon S3 - Mais do Que Armazenamento**

Eu achava que S3 era apenas um "HD na nuvem", mas **aprendi que é muito mais**:

**S3 como Gatilho de Eventos:**
- Posso configurar o S3 para **notificar** outros serviços quando algo acontece
- Eventos disponíveis: criação de objeto (PUT), remoção (DELETE), restauração, etc.
- Posso filtrar eventos por **prefixo** (pastas) ou **sufixo** (extensão de arquivo)

**Exemplo prático que testei:**
```
Upload de arquivo .jpg no S3
    ↓
Trigger dispara Lambda
    ↓
Lambda processa a imagem
    ↓
Resultado salvo em outro bucket
```

**Aprendi também sobre:**
- **Buckets**: os "containers" onde os arquivos ficam
- **Objects**: os arquivos propriamente ditos
- **Keys**: o "caminho" completo do arquivo (como `pasta/subpasta/arquivo.txt`)
- **Versionamento**: posso manter histórico de versões dos arquivos

### 4. **IAM - Segurança é Fundamental**

**Este foi um dos aprendizados mais importantes**: sem as permissões corretas, NADA funciona na AWS!

**Aprendi que:**
- Lambda precisa de uma **IAM Role** para acessar outros serviços
- Princípio do **menor privilégio**: dar apenas as permissões necessárias
- Três componentes principais:
  - **Política (Policy)**: documento JSON que define permissões
  - **Role**: "papel" que pode ser assumido por serviços
  - **Recurso**: o que está sendo acessado (ex: bucket específico)

**Erro comum que cometi:**
Esqueci de dar permissão `s3:GetObject` e a Lambda não conseguia ler os arquivos. O erro só apareceu nos logs do CloudWatch!

### 5. **Estrutura de Eventos do S3**

**Aprendi que** quando o S3 dispara uma Lambda, ele envia um objeto JSON com todas as informações do evento.

**Estrutura que entendi:**
```python
event = {
    'Records': [
        {
            's3': {
                'bucket': {
                    'name': 'meu-bucket'  # Nome do bucket
                },
                'object': {
                    'key': 'pasta/arquivo.txt',  # Caminho do arquivo
                    'size': 1024  # Tamanho em bytes
                }
            }
        }
    ]
}
```

**Insight**: Posso processar múltiplos arquivos em uma única invocação, pois `Records` é um array!

---

## 💡 Insights Práticos

### 1. **CloudWatch é Meu Melhor Amigo para Debug**

Aprendi que **SEMPRE** devo olhar os logs no CloudWatch quando algo não funciona. É lá que ficam os `print()` do Python e os erros detalhados.

**Dica que descobri**: Adicionar logs descritivos facilita MUITO o troubleshooting:
```python
print(f"Processando arquivo: {file_key}")
print(f"Tamanho: {file_size} bytes")
```

### 2. **Boto3 - SDK da AWS para Python**

**Descobri que** `boto3` é a biblioteca Python para interagir com serviços AWS. É muito poderosa!

**Principais operações que aprendi:**
```python
import boto3

# Cliente S3
s3 = boto3.client('s3')

# Ler arquivo
response = s3.get_object(Bucket='meu-bucket', Key='arquivo.txt')
conteudo = response['Body'].read()

# Escrever arquivo
s3.put_object(
    Bucket='meu-bucket',
    Key='saida/novo-arquivo.txt',
    Body='Conteúdo processado'
)
```

### 3. **Cold Start vs Warm Start**

**Aprendi na prática** a diferença:
- **Cold Start**: primeira execução ou após inatividade - demora mais (pode levar segundos)
- **Warm Start**: execuções subsequentes - muito mais rápidas (milissegundos)

**Solução que descobri**: Para funções críticas, posso usar "Provisioned Concurrency" para manter instâncias aquecidas.

### 4. **Limites e Quotas**

**Descobri que** existem limites importantes:
- Tamanho do código: 50MB (zipado) ou 250MB (descompactado)
- Timeout máximo: 15 minutos
- Variáveis de ambiente: até 4KB
- Concurrent executions: 1000 por região (pode aumentar via suporte)

**Quando atingi o limite**: Minha primeira função tentou processar um arquivo de 1GB e deu timeout. Aprendi a dividir o processamento em chunks!

---

## 🛠️ Minha Implementação Prática

### Projeto 1: Conversor de Texto Automático

**O que fiz**: Criei uma Lambda que converte arquivos .txt para maiúsculas automaticamente.

**Fluxo:**
1. Upload de arquivo `.txt` no bucket `input-bucket`
2. Lambda é disparada automaticamente
3. Função lê o arquivo, converte para maiúsculas
4. Salva resultado em `output-bucket/processed/`

**Código que desenvolvi:**

```python
import json
import boto3
from datetime import datetime

s3 = boto3.client('s3')

def lambda_handler(event, context):
    print("🚀 Função Lambda iniciada!")
    
    for record in event['Records']:
        # Extrair informações do evento
        bucket_origem = record['s3']['bucket']['name']
        arquivo_key = record['s3']['object']['key']
        tamanho = record['s3']['object']['size']
        
        print(f"📄 Arquivo detectado: {arquivo_key}")
        print(f"📦 Bucket: {bucket_origem}")
        print(f"💾 Tamanho: {tamanho} bytes")
        
        try:
            # Baixar arquivo do S3
            print("⬇️ Baixando arquivo...")
            resposta = s3.get_object(Bucket=bucket_origem, Key=arquivo_key)
            conteudo = resposta['Body'].read().decode('utf-8')
            
            print(f"✅ Arquivo baixado com sucesso!")
            print(f"📝 Conteúdo original (primeiras 100 chars): {conteudo[:100]}")
            
            # Processar: converter para maiúsculas
            print("🔄 Convertendo para maiúsculas...")
            conteudo_processado = conteudo.upper()
            
            # Definir bucket e key de destino
            bucket_destino = 'meu-output-bucket'
            arquivo_saida = f"processed/{arquivo_key}"
            
            # Upload do arquivo processado
            print(f"⬆️ Fazendo upload para: {arquivo_saida}")
            s3.put_object(
                Bucket=bucket_destino,
                Key=arquivo_saida,
                Body=conteudo_processado.encode('utf-8'),
                ContentType='text/plain'
            )
            
            print("🎉 Processamento concluído!")
            
            return {
                'statusCode': 200,
                'body': json.dumps({
                    'mensagem': 'Sucesso!',
                    'arquivo_original': arquivo_key,
                    'arquivo_processado': arquivo_saida,
                    'timestamp': datetime.now().isoformat()
                })
            }
            
        except Exception as erro:
            print(f"❌ ERRO: {str(erro)}")
            return {
                'statusCode': 500,
                'body': json.dumps({
                    'erro': str(erro)
                })
            }
```

**O que aprendi desenvolvendo isso:**
- Como extrair informações do evento S3
- A importância de logs descritivos (os emojis ajudam a visualizar!)
- Tratamento de erros é ESSENCIAL
- Sempre converter encoding (`.decode()` e `.encode()`)

---

## 🐛 Problemas Que Enfrentei e Como Resolvi

### Problema 1: "Access Denied" ao Ler do S3

**O que aconteceu**: Minha Lambda não conseguia ler arquivos do S3.

**Erro nos logs**:
```
An error occurred (AccessDenied) when calling the GetObject operation
```

**Como resolvi**:
1. Fui no IAM e verifiquei a Role da Lambda
2. Adicionei a permissão `s3:GetObject` para o bucket específico
3. **Lição aprendida**: SEMPRE verificar as permissões primeiro!

### Problema 2: Função Dando Timeout

**O que aconteceu**: Lambda parava de executar após 3 segundos (timeout padrão).

**Como resolvi**:
1. Aumentei o timeout nas configurações da Lambda para 30 segundos
2. **Lição aprendida**: Para processamento de arquivos maiores, preciso ajustar o timeout!

### Problema 3: Erro "Module Not Found"

**O que aconteceu**: Tentei usar a biblioteca `Pillow` mas recebi erro de módulo não encontrado.

**Como resolvi**:
1. Aprendi sobre **Lambda Layers** - onde posso adicionar dependências
2. Criei um layer com Pillow e anexei à minha função
3. **Alternativa**: usar uma imagem Docker customizada
4. **Lição aprendida**: Lambda não vem com todas as bibliotecas Python por padrão!

### Problema 4: Trigger Sendo Disparado em Loop

**O que aconteceu**: Minha Lambda salvava no mesmo bucket de entrada, causando trigger infinito! 😱

**Como resolvi**:
1. Criei um bucket separado para saída (`output-bucket`)
2. Configurei o trigger apenas no bucket de entrada
3. **Lição aprendida**: NUNCA processar e salvar no mesmo bucket que dispara o trigger!

---

## 💰 O Que Aprendi Sobre Custos

### Modelo de Cobrança do Lambda

**Free Tier (sempre grátis):**
- 1 milhão de requests por mês
- 400.000 GB-segundos de compute time por mês

**Depois do Free Tier:**
- $0.20 por 1 milhão de requests
- $0.0000166667 por GB-segundo

**Exemplo de cálculo que fiz:**
```
Cenário: 10.000 execuções/mês, 512MB RAM, 2 segundos cada

Compute: 10.000 × 0.5GB × 2s = 10.000 GB-segundos
Custo Compute: (10.000 - 400.000 free tier) = $0 (dentro do free tier!)

Requests: 10.000 requests
Custo Requests: (10.000 - 1.000.000 free tier) = $0 (dentro do free tier!)

Total: GRÁTIS! 🎉
```

**Insight importante**: Para a maioria dos casos de automação simples, fica no free tier!

### Custos do S3

**Storage:**
- $0.023 per GB/mês (Standard)
- $0.0125 per GB/mês (Infrequent Access)

**Requests:**
- PUT/POST: $0.005 per 1.000 requests
- GET: $0.0004 per 1.000 requests

**Aprendi que**: Para arquivos acessados raramente, usar S3 Glacier pode economizar muito!

---

## 🎯 Casos de Uso Reais Que Pensei

Após aprender tudo isso, identifiquei cenários reais onde posso aplicar:

### 1. **Processamento de Notas Fiscais**
- Upload de XML → Lambda extrai dados → Salva em banco de dados
- **Economia**: Não preciso manter servidor rodando 24/7

### 2. **Backup Automático de Logs**
- App gera log diário → Lambda compacta → Move para S3 Glacier
- **Benefício**: Redução de 75% no custo de armazenamento

### 3. **Geração de Relatórios**
- Upload de CSV → Lambda processa → Gera PDF → Envia por email
- **Vantagem**: Processamento sob demanda

### 4. **Otimização de Imagens para Web**
- Upload de foto → Lambda redimensiona → Cria múltiplos tamanhos
- **Aplicação**: Site com carregamento mais rápido

### 5. **Validação de Arquivos CSV**
- Upload → Lambda valida formato → Move para pasta correta ou rejeita
- **Uso**: Pipeline de dados automatizado

---

## 📊 Monitoramento e Observabilidade

### CloudWatch - Meu Painel de Controle

**Aprendi a usar:**

1. **CloudWatch Logs**: Onde ficam os `print()` do código
   - Cada execução cria um "log stream"
   - Posso filtrar logs por palavra-chave
   - Retenção configurável (1 dia até ∞)

2. **CloudWatch Metrics**: Métricas automáticas
   - Invocations (número de execuções)
   - Duration (tempo de execução)
   - Errors (erros ocorridos)
   - Throttles (execuções limitadas)

3. **CloudWatch Alarms**: Alertas automáticos
   - Posso criar alarmes se erros > 5 em 5 minutos
   - Integra com SNS para enviar email/SMS

**Prática que adotei:**
```python
import logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info(f"Evento recebido: {event}")
    # ... resto do código
```

---

## 🚀 Próximos Passos no Meu Aprendizado

Agora que domino o básico, quero explorar:

✅ **Lambda Layers**: Para reutilizar código entre funções  
✅ **Step Functions**: Orquestrar múltiplas Lambdas  
✅ **DynamoDB + Lambda**: Construir APIs serverless  
✅ **Lambda@Edge**: Processar requests no CloudFront  
✅ **EventBridge**: Triggers mais avançados  
✅ **SAM (Serverless Application Model)**: Infraestrutura como código  

---

## 📚 Recursos Que Me Ajudaram

### Documentação Oficial
- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [Amazon S3 Documentation](https://docs.aws.amazon.com/s3/)
- [Boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)

### Tutoriais que Segui
- [AWS Lambda Tutorial](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html)
- [S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html)

### Comunidade
- [AWS re:Post](https://repost.aws/) - Forum oficial da AWS
- [Stack Overflow - aws-lambda tag](https://stackoverflow.com/questions/tagged/aws-lambda)

---

## 🏆 Conclusão e Reflexões

Este desafio foi **muito além de apenas aprender ferramentas técnicas**. Eu desenvolvi:

**Habilidades Técnicas:**
- ✅ Arquitetura serverless
- ✅ Automação de processos
- ✅ Cloud computing (AWS)
- ✅ Gerenciamento de permissões (IAM)
- ✅ Debugging e troubleshooting

**Habilidades Comportamentais:**
- ✅ Resolução de problemas
- ✅ Persistência (muitos erros no caminho!)
- ✅ Documentação técnica
- ✅ Aprendizado autodidata

**Principais Takeaways:**

1. **Serverless é poderoso**: Posso criar automações robustas sem gerenciar servidores
2. **Segurança é fundamental**: IAM pode ser complicado, mas é essencial entender
3. **Logs salvam vidas**: CloudWatch é imprescindível para debug
4. **Começar simples**: Comecei com um "Hello World", depois evolui para casos reais
5. **Documentar é essencial**: Este README me ajudará no futuro!

**O que mais me surpreendeu:**
A facilidade de integração entre os serviços AWS. Literalmente alguns cliques e meu S3 já estava conversando com a Lambda!

**Desafio que pretendo fazer agora:**
Construir um sistema completo de processamento de imagens com múltiplas Lambdas orquestradas!


## 🤝 Contribuições

Este é um repositório de aprendizado pessoal, mas feedbacks são sempre bem-vindos!

Se você também está aprendendo AWS, sinta-se à vontade para:
- ⭐ Dar uma estrela no repositório
- 🐛 Reportar erros ou sugestões
- 💬 Compartilhar suas próprias experiências
