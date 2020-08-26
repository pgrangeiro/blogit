---
description: >-
  Testar aplicações serverless localmente pode ser um desafio - principalmente
  se você não deseja utilizar o serverless-framework e pretende ter controle
  sobre o que será deployado.
---

# Serverless localmente com LocalStack

[LocalStack](https://github.com/localstack/localstack) é uma ferramenta, escrita em python, que se utiliza de containers Docker para simular o ambiente serverless da AWS. Em conjunto com o [awscli-local](https://github.com/localstack/awscli-local), seremos capazes de reproduzir todo o ecossistema da AWS para testar o desenvolvimento das nossas aplicações serverless localmente :\)

## Instalação

Antes de instalar o LocalStack, você precisará instalar os seguintes requisitos:

* Python 3+ \(este tutorial utilizou a versão 3.8\)
* Pip\*
* Docker
* AWS Cli

```
$ pip install localstack
```

Por ser uma ferramenta python, você pode instalá-la utilizando o _pip_.

No caso deste tutorial, iremos utilizar também o _awscli-local_, que irá funcionar como um wrapper do _aws cli_. A partir dele, seremos capazes de criar os recursos localmente, nos containers do LocalStack.

```bash
$ pip install awscli-local
```

Com isso, já temos tudo pronto rodar o LocalStack!

```bash
$ localstack start
```

{% hint style="warning" %}
Por padrão, o localstack sobe **um container docker para cada serviço AWS que ele possui disponível**. Portanto, se você quiser limitar a quantidade de recursos a serem levantados, confira a [documentação da ferramenta](https://github.com/localstack/localstack) para configurar as variáveis de ambiente necessárias para isto.
{% endhint %}

## Criando uma lambda localmente

Agora, com o localstack rodando, podemos criar a nossa primeira lambda localmente. 

{% hint style="warning" %}
A partir daqui, os comandos serão muito similares aos comandos do _aws cli_, que utilizamos para criar recursos na AWS, com a diferença de estarmos sempre utilizando o _aws local_.
{% endhint %}

```bash
$ awslocal lambda create-function --function-name myfunction --handler handler.myhandler --runtime python3.7 --role myrole --zip-file fileb://function.zip
```

* **--function-name** define o nome da função lambda que está sendo criada
* **--handler** define quem será o handler da aplicação
* **--runtime** define o ambiente de execução do código
  * Deve ser um dos valores pré-definidos na [documentação da AWS](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html).
* **--role** define as roles de execução da lambda
  * Pode receber qualquer valor default
  * Caso esteja criando uma lambda com integrações a outros serviços AWS rodando no localstack, veja também a [documentação de roles de execução da AWS](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html).
* **--zip-file** define o pacote zip, o código, que será deployado na lambda

Se tudo der certo, teremos a nossa lambda criada no container responsável por lambdas do LocalStack. Podemos inclusive testá-la, através do invoke :\)

```bash
$ awslocal lambda invoke --function-name myfunction --payload "{}" out --log-type Tail
```

* **--function-name** define o nome da função lambda que está sendo invocada
* **--payload** define os dados \(formato JSON\) que a lambda espera
  * Utilizado quando a lambda espera dados na sua invocação
* **--log-type** Tail para acompanhar o log da execução na instância do localstack que foi levantada

## Expondo uma lambda utilizando o API Gateway localmente

```bash
$ awslocal apigateway create-rest-api --name myapi
```

Certifique-se de copiar o **id** da API criada, que é exibido na saída do comando acima.

A API criada possui apenas um base-path, precisamos agora então criar os resources que irão compor nossa API. Os resources compreendem os subdomínios da nossa API, cada qual possui também um id único, que pode ser utilizado para linkar um resource a outro, como subdomínio de um subdomínio \(algo como /item e /item/delete\).

Primeiro, vamos listar os recursos da nossa API. Apesar de vazia, ela já possui um base-path e esse base-path será o parent dos nossos resources.

```bash
$ awslocal apigateway get-resources --rest-api-id lmy32f5pc0
> {
    "items": [
        {
            "id": "8u603wqxb4",
            "path": "/"
        }
    ]
}
```

* **--rest-api-id** espera o id da API criada

Com o id do base-path em mãos, conseguimos criar o nosso primeiro resource:

```bash
$ awslocal apigateway create-resource --rest-api-id lmy32f5pc0 --path-part path --parent-id 8u603wqxb4
> {
    "id": "md3ay0ebnw",
    "parentId": "8u603wqxb4",
    "pathPart": "path",
    "path": "/path"
} 
```

* **--rest-api-id** espera o id da API criada
* **--path-part** espera o path do resource a ser criado
* **--parent-id** espera o id do path pai do resource criado
  * Neste exemplo, nosso resource é um subdomínio da raiz da API

Com o resource criado, podemos adicionar os metódos HTTP desejados. Neste exemplo, iremos adicionar o método POST.

```bash
$ awslocal apigateway put-method --rest-api-id lmy32f5pc0 --resource-id md3ay0ebnw --http-method POST --authorization-type NONE
> {
    "httpMethod": "POST",
    "authorizationType": "NONE",
    "apiKeyRequired": false
}
```

* **--rest-api-id** espera o id da API criada
* **--resource-id** espera o id do resource onde o método será adicionado
* **--http-method** espera o método HTTP a ser adicionado no resource
* **--authorization-type** espera o método de autenticação para o endpoint a ser criado

Agora, finalmente, podemos conectar o nosso endpoint criado no API Gateway à uma lambda.

```bash
$ awslocal apigateway put-integration --rest-api-id lmy32f5pc0 --resource-id md3ay0ebnw --http-method POST --type AWS --integration-http-method POST --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:myfunction/invocations
> {
    "type": "AWS",
    "httpMethod": "POST",
    "uri": "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:sshkeymanager/invocations",
    "passthroughBehavior": "WHEN_NO_MATCH",
    "cacheNamespace": "51df6e0e",
    "cacheKeyParameters": [],
    "integrationResponses": {
        "200": {
            "statusCode": 200,
            "responseTemplates": {
                "application/json": null
            }
        }
    }
}
```

* **--rest-api-id** espera o id da API criada
* **--resource-id** espera o id do resource onde o método será adicionado
* **--http-method** espera o método HTTP a ser adicionado no resource
*  **--type** AWS
*  **--integration-http-method** espera o método HTTP no qual o API Gateway irá se comunicar com a lambda
* **--uri** espera um arn identificador único do endpoint para qual o API Gateway irá rotear as requisições
  * Note que no meio da uri, existe a informação da arn da lambda criada previamente. Substitua pelo arn da sua lambda

Agora, precisamos deployar o nosso API Gateway, localmente:

```bash
$ awslocal apigateway create-deployment --rest-api-id lmy32f5pc0 --stage-name local --endpoint-url=http://localhost:4567
> {
    "id": "xngnmbe24t",
    "description": "",
    "createdDate": 1597864231
}
```

* **--rest-api-id** espera o id da API criada
* **--stage-name** espera o identificador do stage a ser deployado
  * Pode-se informar qualquer string, já que o deploy é local
* **--endpoint-url** espera o endpoint no qual o API gateway será deployado

Se o deploy ocorrer com sucesso, você pode listar os resources de sua API deployada acessando: [http://localhost:4567/restapis](http://localhost:4567/restapis)

Agora, você pode testar a chamada de sua lambda através do API Gateway. A url de chamada é composta por http://&lt;endpoint-url&gt;/restapis/&lt;rest-api-id&gt;/&lt;stage-name&gt;/\_user\_request\_/&lt;path-part&gt;. Podemos então testar a chamada do nosso exemplo da seguinte maneira:

```bash
$ curl -X POST http://localhost:4567/restapis/lmy32f5pc0/local/_user_request_/path -d '{}'
```

{% hint style="info" %}
Não deixe de conferir a documentação completa do [aws cli](https://aws.amazon.com/cli/), para ter acesso aos recursos da AWS, sempre substituindo pelo wrapper _awslocal_ para utilizá-los localmente.
{% endhint %}

Dúvidas? Sugestões? Deixe aí nos comentários :\)

