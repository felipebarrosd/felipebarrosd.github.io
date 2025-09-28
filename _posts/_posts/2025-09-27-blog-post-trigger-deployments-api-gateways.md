
# API Gateway Deployments Não Sendo Atualizados com Terraform

## Introdução

Quem trabalha com **API Gateway** na **AWS** usando **Terraform** provavelmente já passou por uma situação estranha: você aplica mudanças nas rotas, integrações ou métodos da sua API, o `terraform apply` executa sem erros, mas… quando testa a API, nada mudou.  
Isso acontece porque os **deployments** do API Gateway não são automaticamente recriados pelo Terraform em toda alteração, e isso pode gerar confusão, bugs em produção e até rollback inesperado.

Neste artigo, vamos entender:
- Por que esse problema acontece.
- Como o Terraform gerencia `aws_api_gateway_deployment`.
- Estratégias para evitar que seu deployment “volte atrás” ou não aplique as mudanças.

---

## O Problema

No API Gateway REST, cada vez que você altera rotas, métodos ou integrações, é necessário criar um **novo deployment** para que as alterações entrem em vigor em um **stage**.

Porém, o recurso `aws_api_gateway_deployment` do Terraform tem um comportamento peculiar:

- Ele só cria um novo deployment quando seu **hash interno** muda.
- Isso significa que mesmo com alterações na API, o deployment **não é atualizado** automaticamente.

Resultado: sua infraestrutura no Terraform parece correta, mas sua API continua servindo uma versão antiga.

---

## Exemplo de Código

```hcl
resource "aws_api_gateway_rest_api" "example" {
  name = "example-api"
}

resource "aws_api_gateway_resource" "example" {
  rest_api_id = aws_api_gateway_rest_api.example.id
  parent_id   = aws_api_gateway_rest_api.example.root_resource_id
  path_part   = "items"
}

resource "aws_api_gateway_method" "example" {
  rest_api_id   = aws_api_gateway_rest_api.example.id
  resource_id   = aws_api_gateway_resource.items.id
  http_method   = "GET"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "example" {
  rest_api_id             = aws_api_gateway_rest_api.example.id
  resource_id             = aws_api_gateway_resource.items.id
  http_method             = aws_api_gateway_method.get_items.http_method
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.items.invoke_arn
}

resource "aws_api_gateway_deployment" "deployment" {
  rest_api_id = aws_api_gateway_rest_api.example.id
  stage_name  = "dev"

  # O problema: se nada aqui muda, o Terraform não recria o deployment
}
```

Nesse cenário, se alterarmos a integração ou adicionarmos uma rota, o Terraform pode aplicar a mudança na configuração da API, mas **não fará o redeploy** da API automaticamente.

---

## Solução: Forçar o Redeploy

A prática recomendada é adicionar um **`triggers`** ao recurso `aws_api_gateway_deployment`, para que ele seja recriado sempre que houver mudança na API:

```hcl
resource "aws_api_gateway_deployment" "deployment" {
  rest_api_id = aws_api_gateway_rest_api.example.id
  stage_name  = "dev"

  triggers = {
    redeployment = sha1(jsonencode([
      aws_api_gateway_resource.example.id,
      aws_api_gateway_method.example.id,
      aws_api_gateway_integration.example.id,
    ]))
  }
}
```

Outra abordagem é usar **`timestamp()`** ou **`md5`** de recursos críticos como dependência, garantindo que um novo deployment seja criado:

```hcl
resource "aws_api_gateway_deployment" "deployment" {
  rest_api_id = aws_api_gateway_rest_api.example.id
  stage_name  = "dev"

  triggers = {
    timestamp = timestamp()
  }
}
```

Essa última opção força um redeploy a cada `apply`, útil em desenvolvimento, mas pode ser perigoso em produção (gera sempre um novo deployment, mesmo sem mudanças).

---

## Boas Práticas

1. **Use `triggers` com `sha1(jsonencode(...))`** das rotas, métodos ou integrações. Assim só redeploya quando houver mudança real.
2. **Evite `timestamp()` em produção**, pois cria deployments desnecessários e polui o histórico.
3. **Automatize a limpeza de deployments antigos** se sua API for atualizada com frequência.
