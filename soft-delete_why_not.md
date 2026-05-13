## Exclusão Lógica (*Soft Delete*) em Bancos de Dados Relacionais - Por Que **NÃO** usar?

### 1. Objetivo

O presente parecer técnico tem por objetivo apresentar fundamentos contrários à adoção do padrão de exclusão lógica (*logical deletion* ou *soft delete*) em bancos de dados relacionais, especialmente em sistemas que dependem fortemente das garantias de integridade e consistência providas pelo modelo relacional.

A análise considera aspectos arquiteturais, operacionais, semânticos e de integridade referencial, demonstrando que a exclusão lógica frequentemente viola princípios fundamentais do paradigma relacional e introduz riscos relevantes à confiabilidade dos dados.

---

## 2. Conceituação

O padrão conhecido como exclusão lógica consiste em não remover fisicamente um registro da base de dados, marcando-o apenas como “excluído” por meio de atributos como:

* `deleted`
* `is_deleted`
* `deleted_at`
* `status = 'DELETED'`

Nesse modelo, os registros continuam existindo fisicamente na tabela, cabendo à aplicação filtrar os dados considerados “ativos”.

Exemplo:

```sql
UPDATE cliente
SET deleted_at = NOW()
WHERE id = 10;
```

Em vez de:

```sql
DELETE FROM cliente
WHERE id = 10;
```

---

# 3. Violação prática da integridade referencial

## 3.1. A principal garantia do modelo relacional

Uma das características mais importantes de bancos relacionais é a integridade referencial, garantida nativamente por:

* chaves estrangeiras (*foreign keys*)
* restrições (*constraints*)
* regras de cascata
* atomicidade transacional

Esses mecanismos impedem, por exemplo:

* exclusão de registros pai ainda referenciados;
* existência de registros órfãos;
* inconsistências entre entidades relacionadas.

Exemplo clássico:

```sql
pedido.cliente_id → cliente.id
```

Nesse cenário, o banco garante que:

* um pedido não pode apontar para um cliente inexistente;
* um cliente não pode ser removido sem tratamento adequado de seus pedidos.

---

## 3.2. A exclusão lógica contorna artificialmente essa garantia

Ao utilizar *soft delete*, a aplicação passa a “simular” uma exclusão sem que o banco de dados reconheça efetivamente a remoção da entidade.

Na prática:

* o registro pai continua existindo fisicamente;
* as *foreign keys* permanecem válidas;
* o banco entende que a integridade está preservada;
* porém, semanticamente, o registro foi removido do domínio de negócio.

Isso cria uma inconsistência conceitual grave:

> O banco considera o registro existente, enquanto a aplicação o considera inexistente.

---

## 3.3. Possibilidade de “excluir” entidades pai sem tratar entidades filhas

Esse é um dos problemas mais críticos do padrão.

Considere:

```text
cliente
 └── pedido
```

Com exclusão física:

```sql
DELETE FROM cliente WHERE id = 10;
```

O banco poderá:

* impedir a operação;
* exigir exclusão dos filhos;
* aplicar `ON DELETE CASCADE`;
* aplicar `ON DELETE SET NULL`.

Ou seja: existe governança relacional.

Já com exclusão lógica:

```sql
UPDATE cliente
SET deleted_at = NOW()
WHERE id = 10;
```

Nada impede que:

* pedidos continuem ativos;
* tabelas filhas permaneçam operacionais;
* relacionamentos semanticamente inválidos persistam.

Na prática, isso equivale a abrir mão da integridade referencial nativamente fornecida pelo SGBD.

O banco deixa de ser o guardião da consistência e transfere integralmente essa responsabilidade para código de aplicação — exatamente o tipo de problema que bancos relacionais surgiram para resolver.

---

# 4. Aumento exponencial de complexidade

## 4.1. Toda consulta passa a exigir filtros adicionais

Após adoção de *soft delete*, praticamente toda consulta precisa incluir:

```sql
WHERE deleted_at IS NULL
```

ou equivalente.

Isso implica:

* aumento de complexidade cognitiva;
* repetição sistemática;
* maior probabilidade de erro humano;
* inconsistências entre consultas.

---

## 4.2. Vazamento acidental de dados “excluídos”

A omissão de um simples filtro pode causar:

* exibição de registros supostamente excluídos;
* reprocessamentos indevidos;
* cálculos incorretos;
* falhas em relatórios;
* reativação involuntária de entidades.

Esse problema é especialmente grave em:

* ETLs;
* integrações;
* APIs;
* relatórios analíticos;
* consultas ad-hoc;
* ferramentas BI.

---

## 4.3. Multiplicação de regras de negócio ocultas

O conceito de “registro válido” deixa de ser garantido pelo banco e passa a depender de convenções implícitas.

Exemplo:

```sql
SELECT *
FROM   pedido p
JOIN   cliente c ON c.id = p.cliente_id
WHERE  c.deleted_at IS NULL
```

Toda consulta relacional passa a depender de conhecimento contextual adicional.

Isso reduz:

* previsibilidade;
* legibilidade;
* manutenibilidade;
* auditabilidade.

---

# 5. Impactos negativos em performance e armazenamento

A exclusão lógica também produz efeitos estruturais relevantes.

## 5.1. Crescimento indefinido das tabelas

Como registros nunca são removidos:

* índices crescem continuamente;
* *table scans* tornam-se mais custosos;
* *vacuum* e manutenção aumentam;
* estatísticas degradam;
* particionamento torna-se mais complexo.

---

## 5.2. Piora da seletividade dos índices

Índices passam a conter:

* registros ativos;
* registros “mortos” semanticamente.

Isso reduz eficiência de:

* buscas;
* *joins*;
* agregações;
* ordenações.

---

# 6. Quebra de semântica relacional

No modelo relacional, uma entidade inexistente simplesmente não deve existir.

Exclusão lógica cria um estado híbrido:

| Estado       | Existe fisicamente | Existe semanticamente |
| ------------ | ------------------ | --------------------- |
| Ativo        | Sim                | Sim                   |
| Soft deleted | Sim                | Não                   |

Esse terceiro estado não pertence naturalmente ao modelo relacional.

Consequentemente:

* regras tornam-se ambíguas;
* entidades passam a ter “meia existência”;
* consultas tornam-se semanticamente frágeis.

---

# 7. Fragilidade operacional e risco arquitetural

## 7.1. Dependência excessiva da disciplina da equipe

O sucesso do *soft delete* depende de:

* todos lembrarem dos filtros;
* todos os serviços respeitarem a convenção;
* todas as integrações aplicarem as regras corretamente.

Ou seja:

> a integridade deixa de ser garantida pelo banco e passa a depender de comportamento humano.

Isso representa regressão arquitetural.

---

## 7.2. Quebra de encapsulamento da integridade

Em bancos relacionais bem modelados:

* o banco protege os dados;
* aplicações podem falhar sem comprometer integridade estrutural.

Com exclusão lógica:

* regras críticas migram para múltiplas aplicações;
* diferentes serviços podem interpretar exclusão de formas distintas;
* inconsistências tornam-se inevitáveis em sistemas distribuídos.

---

# 8. Casos em que a exclusão lógica costuma ser usada indevidamente

Frequentemente o *soft delete* é adotado como solução genérica para:

* auditoria;
* rastreabilidade;
* recuperação de dados;
* conformidade regulatória;
* medo de perda acidental.

Entretanto, existem soluções tecnicamente superiores:

| Necessidade     | Solução mais adequada            |
| --------------- | -------------------------------- |
| Auditoria       | tabelas de histórico             |
| Versionamento   | event sourcing / temporal tables |
| Recuperação     | backup e PITR                    |
| Retenção legal  | arquivamento                     |
| Rastreabilidade | trilhas de auditoria             |

---

# 9. Alternativas tecnicamente superiores

## 9.1. Exclusão física com integridade referencial

A abordagem mais consistente continua sendo:

* `DELETE` real;
* constraints adequadas;
* cascatas explícitas;
* regras transacionais.

---

## 9.2. Tabelas de arquivamento

Exemplo:

```text
cliente
cliente_archive
```

Benefícios:

* separação semântica clara;
* preservação de performance operacional;
* manutenção da integridade relacional no domínio ativo.

---

## 9.3. Temporal tables / auditoria nativa

Muitos SGBDs modernos oferecem:

* temporal tables;
* CDC (*Change Data Capture*);
* audit logs;
* WAL recovery.

Essas abordagens preservam histórico sem corromper a semântica relacional principal.

---

# 10. Conclusão

A utilização indiscriminada do padrão de exclusão lógica em bancos relacionais representa, sob diversos aspectos, uma deterioração das garantias fundamentais do modelo relacional.

Embora frequentemente apresentada como mecanismo de segurança ou rastreabilidade, a exclusão lógica:

* enfraquece a integridade referencial;
* transfere responsabilidades do SGBD para a aplicação;
* aumenta complexidade sistêmica;
* introduz inconsistências semânticas;
* degrada performance;
* amplia riscos operacionais.

Particularmente grave é o fato de que ela permite que registros de entidades pai sejam semanticamente removidos sem qualquer obrigação estrutural de tratamento das entidades filhas, neutralizando, na prática, uma das principais proteções fornecidas por bancos relacionais: a garantia automática de consistência entre relações.

Assim, sob a ótica arquitetural, relacional e operacional, recomenda-se evitar o uso de exclusão lógica como estratégia padrão de modelagem, reservando sua adoção apenas para cenários excepcionalmente justificados e cuidadosamente controlados.
