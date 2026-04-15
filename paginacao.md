# Especificação: Rota para listar itoken's de determinado device

## Objetivo

Criar um endpoint GET que retorna registros de um device. Por ser um numero teoricamente infinito, devemos devolver uma quantidade limitada de registros e informar ao consumidor que existem mais registros a serem consumidos, dessa forma, fica a cargo do usuario decidir se quer buscar mais dados ou não.

---

## Endpoint

```
GET /{dispositivoId}/dispositivos-autorizados
```

### Path parameter

| Parâmetro        | Tipo   | Obrigatório | Descrição                        |
|------------------|--------|-------------|----------------------------------|
| dispositivoId    | string | Sim         | Identificador do dispositivos    |

### Query parameter

| Parâmetro | Tipo   | Obrigatório | Padrão | Descrição                                                                     |
|-----------|--------|-------------|--------|-------------------------------------------------------------------------------|
| status    | string | Não         | *      | Status dos itokens                                                            |
| cursor    | string | Não         | nulo   | Proximo serial que deve ser listado, iremos retornar mais 200 a partir desse  |
| limite    | int    | Não         | 300    | Quantidade de registros por página (máximo: 300)                              |

---

## Estratégia de Paginação: Baseada em Cursor

Utilizar **paginação por cursor**, pois:
- A base pode retornar até N registros
- Garante consistência mesmo se novos registros forem inseridos entre chamadas

### Como funciona

1. O consumidor chama o endpoint sem `cursor` na primeira requisição
2. A API retorna até 300 registros + um `proximoCursor` (se houver mais)
3. O consumidor usa o `proximoCursor` como `cursor` na próxima chamada
4. Quando não houver mais registros, `proximoCursor` vem `nulo`

### Códigos de status

| Status | Situação |
|--------|----------|
| `206 Partial Content` | Há mais registros além dos retornados (`temMais = true`) |
| `200 OK` | Última página, não há mais registros (`temMais = false`) |

---

## Resposta

### 206 Partial Content — há mais registros

```json
{
  "dados": [
    { "id": 1, "campo": "valor" },
    { "...": "..." }
  ],
  "paginacao": {
    "limite": 300,
    "proximoCursor": "70012547896",
    "temMais": true,
    "total": 500
  }
}
```

### 200 OK — última página

```json
{
  "dados": [ { "...": "..." } ],
  "paginacao": {
    "limite": 300,
    "proximoCursor": null,
    "temMais": false,
    "total": 500
  }
}
```

### Campos da resposta

| Campo                      | Tipo    | Descrição                                                                   |
|----------------------------|---------|-----------------------------------------------------------------------------|
| dados                      | array   | Lista de registros da página atual                                          |
| paginacao.limite           | int     | Quantidade de registros retornados nesta página                             |
| paginacao.proximoCursor    | string  | Token para buscar a próxima página. `null` quando não há mais registros     |
| paginacao.temMais          | boolean | Indica se existem mais registros além dos retornados                        |
| paginacao.total            | int     | Total de registros                                                          |

---

## Fluxo Completo do Consumidor

```
1ª chamada:  GET /{dispositivoId}/dispositivos-autorizados
             → 206, retorna registros 1–300, proximoCursor = "70012345678", temMais = true



2ª chamada:  GET /{dispositivoId}/dispositivos-autorizados?cursor=70012345678
             → 206, retorna registros 301–600, proximoCursor = "70012345600", temMais = true



Nª chamada:  GET /{dispositivoId}/dispositivos-autorizados?cursor=70012345600
             → 200, retorna registros finais, proximoCursor = null, temMais = false
```


## Erros

| Status | Código | Descrição |
|--------|--------|-----------|
| 400 | CURSOR_INVALIDO | Cursor inválido ou corrompido |
| 400 | LIMITE_INVALIDO | Limite maior que 300 ou menor que 1 |
| 404 | DISPOSITIVO_NAO_ENCONTRADO | Dispositivo não encontrado |
| 500 | ERRO_INTERNO | Erro interno ao buscar os registros |