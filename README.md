
# Implantação em Massa de Frota via CSV (Python + PostgreSQL)

Pipeline que demonstra a substituição de um processo de implantação manual
(cliente → veículo → item → subitem) por uma carga automatizada com validação
em camadas, staging table, idempotência e log de auditoria.

> **Nota:** todos os dados usados neste repositório (clientes, placas, itens)
> são fictícios, gerados para fins de demonstração. Nenhuma informação real
> de cliente ou de sistema de produção está presente.

## Problema

Implantar dados em massa manualmente, registro a registro, é lento e sujeito
a erro humano: inconsistência de cadastro (mesmo dado digitado de formas
diferentes), duplicidade, e nenhuma garantia automática de que o dado
respeita as regras do negócio antes de chegar no banco.

Este projeto demonstra uma solução onde **toda a verificação passa a ser
automática antes da carga** — o processo deixa de depender de atenção manual
para pegar erro de digitação, duplicidade ou dado fora do padrão.

## Solução
Fluxo detalhado:

```
CSV bruto
   │
   ▼
[Python] validação estrutural + padronização (pandas)
   │  - colunas obrigatórias
   │  - linhas vazias / nulos em campos obrigatórios
   │  - duplicidade por chave natural (customer_id + plate + number)
   │  - tipos de veículo válidos (consulta dinâmica em car_types)
   │  - upper/strip em campos texto
   ▼
[PostgreSQL] staging table `insercao_massa` (por lote_id)
   ▼
procedure `processar_insercao_massa(lote_id)`
   ▼
procedure `implantacao_carros(...)` — por linha, idempotente (check-then-insert)
   ▼
tabelas finais: clientes / carros / marcas_pneu / pneus
   ▼
trigger `trg_insert_tires` → tabela `logs` (auditoria: quem, quando, o quê)
```

## Stack

- Python (pandas, psycopg)
- PostgreSQL (PL/pgSQL: procedures, trigger)

## Decisões técnicas

- **Staging table com `lote_id`**: cada execução grava com um UUID de lote,
  permitindo rastrear e reprocessar cargas específicas sem misturar execuções.
- **Idempotência por design**: a procedure `implantacao_carros` verifica
  existência antes de inserir em cada tabela — rodar o mesmo CSV duas vezes
  não duplica registros.
- **Transação única por lote**: staging + processamento rodam na mesma
  transação; qualquer erro reverte o lote inteiro (nada fica em estado parcial).
- **Log de auditoria via trigger**: toda inserção de pneu gera registro em
  `logs` automaticamente, sem depender de o processo de origem lembrar de logar.

### Validação em ação

<img width="1243" height="528" alt="image" src="https://github.com/user-attachments/assets/ec2d1b28-87af-4802-9753-29e4dbe63b6a" />

*(print do terminal com o script rodando as checagens — colunas obrigatórias,
nulos, tipo de veículo — antes de qualquer dado tocar o banco)*

### Auditoria gerada automaticamente 

<img width="1329" height="855" alt="WhatsApp Image 2026-08-21 at 16 57 50" src="https://github.com/user-attachments/assets/207a9c34-bb78-4572-815e-63fcc3da68fc" />


*(print da tabela `logs` mostrando o registro automático gerado pelo trigger.*

## Resultado

<img width="633" height="309" alt="image" src="https://github.com/user-attachments/assets/f4c9d184-0026-4e03-b244-185edfc52d02" />

- Lote de demonstração com 500 registros fictícios processado em 2,02 segundos.
- O mesmo processo, feito manualmente registro a registro, levaria próximo de 1 semana entre validações e padronizações dos dados e análise para não violação de Constraints(regras do banco).
- Toda inconsistência (dado nulo, duplicado, fora do padrão) é barrada
  **antes** de chegar ao banco — nenhuma carga incompleta é possível.

## Próximos passos

- Orquestração hoje é execução manual do script; próxima etapa é migrar para
  Airflow (extract/validate → load staging → process, com retry e alerta).

Variáveis de ambiente necessárias: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`.

  ## 👤 Autor

**Gustavo Silva Reis**  
Engenheiro de Dados Júnior  
[LinkedIn](www.linkedin.com/in/gsreisit)

