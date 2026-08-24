# Pipeline de Implantação em Massa via CSV (Python + PostgreSQL)

Pipeline que demonstra a substituição de um processo de implantação manual
(cliente → dispositivo → marca de componente → componente) por uma carga
automatizada com validação em camadas, staging table, idempotência e log
de auditoria.

> **Nota:** todos os dados usados neste repositório (clientes, dispositivos,
> componentes) são fictícios, gerados para fins de demonstração. O domínio
> escolhido (gestão de ativos de TI) é ilustrativo e não representa nenhum
> sistema de produção real.

## Problema

Implantar dados em massa manualmente, registro a registro, é lento e sujeito
a erro humano: inconsistência de cadastro (mesmo dado digitado de formas
diferentes), duplicidade, e nenhuma garantia automática de que o dado
respeita as regras do negócio antes de chegar no banco.

Este projeto demonstra uma solução onde **toda a verificação passa a ser
automática antes da carga** — o processo deixa de depender de atenção manual
para pegar erro de digitação, duplicidade ou dado fora do padrão.

Fluxo detalhado:

```
CSV bruto
   │
   ▼
[Python] validação estrutural + padronização (pandas)
   │  - colunas obrigatórias
   │  - linhas vazias / nulos em campos obrigatórios
   │  - duplicidade por chave natural (customer_id + asset_tag + serial_code)
   │  - tipos de dispositivo válidos (consulta dinâmica em device_types)
   │  - upper/strip em campos texto
   ▼
[PostgreSQL] staging table `staging_import` (por lote_id)
   ▼
procedure `processar_lote(lote_id)`
   ▼
procedure `implanta_registro(...)` — por linha, idempotente (check-then-insert)
   ▼
tabelas finais: customers / devices / component_brands / components
   ▼
trigger `trg_insert_components` → tabela `logs` (auditoria: quem, quando, o quê)
```

## Stack

- Python (pandas, psycopg)
- PostgreSQL (PL/pgSQL: procedures, trigger)

## Decisões técnicas

- **Staging table com `lote_id`**: cada execução grava com um UUID de lote,
  permitindo rastrear e reprocessar cargas específicas sem misturar execuções.
- **Idempotência por design**: a procedure `implanta_registro` verifica
  existência antes de inserir em cada tabela — rodar o mesmo CSV duas vezes
  não duplica registros.
- **Isolamento de erro por linha**: `processar_lote` roda cada chamada dentro
  de um bloco `BEGIN/EXCEPTION` próprio — uma linha inválida é registrada em
  `implantacao_errors` e o processamento continua para as demais, em vez de
  reverter o lote inteiro.
- **Log de auditoria via trigger**: toda inserção de componente gera registro
  em `logs` automaticamente, sem depender de o processo de origem lembrar de logar.

### Validação em ação

<img width="431" height="680" alt="image" src="https://github.com/user-attachments/assets/521a4e42-d93d-4477-8df0-f008a6887310" />

*(print do terminal com o script rodando as checagens — colunas obrigatórias,
nulos, tipo de dispositivo — antes de qualquer dado tocar o banco)*

### Auditoria gerada automaticamente (opcional)

<img width="1720" height="815" alt="image" src="https://github.com/user-attachments/assets/360a8c30-ed51-4dcd-b98c-03b3d2e48697" />

*(print da tabela `logs` mostrando o registro automático gerado pelo trigger —
pode remover esta seção se preferir um README mais enxuto)*

## Resultado

<img width="790" height="385" alt="image" src="https://github.com/user-attachments/assets/62750e31-edb5-4b29-85dd-6ca25c8e8ed4" />

- Processo manual de referência: ~1 semana para um lote de 250 registros.
- Processo automatizado: 2,02s para um lote de 500 registros (o dobro do
  volume), medido com `Measure-Command`.
- Os dois números não são do mesmo volume — o automatizado foi testado
  propositalmente com o dobro para demonstrar que o ganho se mantém mesmo
  aumentando a carga.
- Toda inconsistência (dado nulo, duplicidade, fora do padrão) é barrada
  **antes** de chegar ao banco — nenhuma carga incompleta é possível.

## Próximos passos

- Orquestração hoje é execução manual do script; próxima etapa é migrar para
  Airflow (extract/validate → load staging → process, com retry e alerta por task).

Variáveis de ambiente necessárias: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`.

---

## 👤 Autor

**Gustavo Silva Reis**  
Engenheiro de Dados Júnior  
[www.linkedin.com/in/gsreisit](https://www.linkedin.com/in/gsreisit/)

