# 🗂️ Neo4j + Yelp Dataset — Guia de Configuração
 
Pipeline completo para importar dados do Yelp Academic Dataset em um banco de grafos Neo4j local.
 
---
 
## 🛠️ Passo 1: Preparação do Ambiente
 
1. Instale o **Neo4j Desktop** e crie um banco de dados local.
2. Na aba **Plugins** do banco, instale o **APOC** e clique em **Start** uma vez para garantir que ele inicializou. Depois, clique em **Stop** — o banco precisa estar parado para mover os arquivos.
3. Mova os 3 arquivos originais do Yelp para a pasta `import`:
   - `yelp_academic_dataset_business.json`
   - `yelp_academic_dataset_user.json`
   - `yelp_academic_dataset_review.json`
> 💡 **Atalho para encontrar a pasta:** Clique no banco → **Open** → **Instance folder** → entre na pasta `import`.
 
---
 
## 🌐 Passo 2: Abrir o Servidor Local (Localhost)
 
Para contornar as restrições de segurança de arquivo do Neo4j sem quebrar o sistema:
 
1. Abra o **Prompt de Comando (CMD)** ou **PowerShell** do Windows.
2. Navegue até a pasta `import` (ajuste o nome do usuário se necessário):
```bash
cd C:\Users\lucas\.Neo4jDesktop2\Data\dbmss\dbms-e10e68ab-6853-43e9-8f16-da6bdcfe69b0\import
```
 
3. Inicie o servidor HTTP do Python:
```bash
python -m http.server 8000
```
 
> ⚠️ **Deixe essa janela aberta durante todo o processo de importação.**
 
---
 
## ⚡ Passo 3: Scripts Cypher (Neo4j Browser)
 
Execute os blocos abaixo **exatamente nesta ordem** no Neo4j Browser.
 
### 1. Limpeza Inicial — Garantia de Banco Zerado
 
```cypher
MATCH (n)
DETACH DELETE n
```
 
### 2. Importação de Usuários (Amostra de 500)
 
```cypher
CALL apoc.load.json("http://localhost:8000/yelp_academic_dataset_user.json") YIELD value
WITH value LIMIT 50000
WITH value WHERE value.user_id IS NOT NULL
WITH value LIMIT 500
CREATE (u:User {
    id: value.user_id,
    name: value.name,
    review_count: toInteger(value.review_count),
    average_stars: toFloat(value.average_stars)
})
```
 
### 3. Importação de Empresas (Amostra de 500)
 
```cypher
CALL apoc.load.json("http://localhost:8000/yelp_academic_dataset_business.json") YIELD value
WITH value LIMIT 500
CREATE (b:Business {
    id: value.business_id,
    name: value.name,
    address: value.address,
    city: value.city,
    state: value.state,
    stars: toFloat(value.stars),
    review_count: toInteger(value.review_count)
})
```
 
### 4. Criação das Conexões via Reviews (varrendo 500k linhas)
 
```cypher
CALL apoc.load.json("http://localhost:8000/yelp_academic_dataset_review.json") YIELD value
WITH value LIMIT 500000
MATCH (u:User {id: value.user_id})
MATCH (b:Business {id: value.business_id})
CREATE (u)-[r:REVIEWED {
    id: value.review_id,
    stars: toInteger(value.stars),
    date: value.date,
    text: value.text
}]->(b)
```
 
---
 
## 📊 Passo 4: Validação e Visualização
 
### Contagem de Nós — deve retornar exatamente 1000
 
```cypher
MATCH (n)
RETURN count(n) AS total_de_nos
```
 
### Grafo Interativo — visualização completa
 
```cypher
MATCH (u:User)-[r:REVIEWED]->(b:Business)
RETURN u, r, b
LIMIT 50
```

# 🔧 CRUD com Neo4j — Operações Básicas em Cypher

Referência rápida das operações **Create, Read, Update e Delete** aplicadas ao modelo de grafos com `User`, `Business` e `REVIEWED`.

---

## 1. 🟢 Create (Criar)

### Criar um novo Usuário (`User`)

```cypher
CREATE (u:User {
    id: "user_lucas_01",
    name: "Lucas",
    review_count: 0,
    average_stars: 0.0
})
RETURN u
```

### Criar uma nova Empresa (`Business`)

```cypher
CREATE (b:Business {
    id: "business_tech_01",
    name: "Tech Solutions",
    address: "Av. Principal, 100",
    city: "San Francisco",
    state: "CA",
    stars: 5.0,
    review_count: 0
})
RETURN b
```

### Criar um Relacionamento (`REVIEWED`) entre um Usuário e uma Empresa existentes

```cypher
MATCH (u:User {id: "user_lucas_01"})
MATCH (b:Business {id: "business_tech_01"})
CREATE (u)-[r:REVIEWED {
    id: "review_001",
    stars: 5,
    date: "2026-05-30",
    text: "Excelente atendimento e infraestrutura!"
}]->(b)
RETURN u, r, b
```

---

## 2. 🔵 Read (Ler / Consultar)

### Consultar um Usuário específico pelo ID ou Nome

```cypher
MATCH (u:User {id: "user_lucas_01"})
RETURN u
```

### Consultar Empresas filtrando por propriedade (ex: Cidade)

```cypher
MATCH (b:Business {city: "San Francisco"})
RETURN b
LIMIT 10
```

### Consultar todas as Avaliações feitas por um Usuário específico

```cypher
MATCH (u:User {id: "user_lucas_01"})-[r:REVIEWED]->(b:Business)
RETURN u.name AS Usuario, r.stars AS Nota, r.text AS Comentario, b.name AS Empresa
```

---

## 3. 🟡 Update (Atualizar)

### Atualizar propriedades de um Nó (ex: corrigir nome e média de estrelas de um Usuário)

```cypher
MATCH (u:User {id: "user_lucas_01"})
SET u.name = "Lucas Silva",
    u.average_stars = 5.0
RETURN u
```

---

## 4. 🔴 Delete (Deletar)

### Deletar apenas um Relacionamento (apagar a Review, mantendo Usuário e Empresa intactos)

```cypher
MATCH (u:User {id: "user_lucas_01"})-[r:REVIEWED {id: "review_001"}]->(b:Business)
DELETE r
```
