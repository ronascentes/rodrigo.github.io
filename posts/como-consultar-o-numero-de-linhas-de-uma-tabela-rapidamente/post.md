<!--:::{
  "post_title": "Como consultar o número de linhas de uma tabela rapidamente no SQL Server",
  "post_description": "Uma tarefa comum tanto na administração ou no desenvolvimento de banco de dados é consultar o número de linhas de uma tabela. Nesse artigo mostrarei quatro maneiras distintas de como consultar o número de linhas de uma tabela eficientemente.",
  "post_created_at": "Wed Aug 16 2023 17:19:12 GMT-0300 (Brasilia Standard Time)"
}:::-->

Uma tarefa comum tanto na administração ou no desenvolvimento de banco de dados é consultar o número de linhas de uma tabela. Nesse artigo mostrarei quatro maneiras distintas de como consultar o número de linhas de uma tabela eficientemente.

#### Método 1 — O Intuitivo
Talvez a forma mais intuitiva e fácil de consultar o número de linhas de uma tabela é utilizar a função `COUNT()`, como, por exemplo, `SELECT COUNT(<nome_do_campo>) FROM <nome_da_tabela>`. Apesar desse método retornar um número preciso de linhas, ele torna-se lento proporcionalmente ao número de linhas da tabela. Isso se deve ao fato que motor de consulta irá, inevitavelmente, ler TODAS as linhas da tabela, e não importa se o campo utilizado seja um index clusterizado ou não clusterizado pois não há benefício de index. Além disso, se for executado em uma tabela com uma alta carga de trabalho, esse tipo de consulto pode causar excessivos bloqueios (*locks*). Por isso, o Método 1 é somente recomendado para tabelas com poucas linhas e/ou em ambientes de testes ou de não-produção.

```SQL
SELECT COUNT(id) from Employees;
```
<br/>

#### Método 2 — Rápido mas não preciso
Selecionar o campo rows da tabela *sys.sysindexes* é o método mais rápido para consultar o número de linhas. No entanto, tal rapidez sacrifica a precisão do valor do número de linhas, uma vez que, depende se as estatísticas estão atualizadas. Além disso, a tabela *sys.sysindexes* está em processo de descomissionamento, ou seja, será removida em uma versão futura do Microsoft SQL Server.

```SQL
SELECT rows FROM sys.sysindexes
WHERE id = OBJECT_ID ('<table_name>')
AND indid < 2;
```
<br/>

#### Método 3 — Por debaixo dos panos
O método 3 é como o SQL Server Management Studio consulta o número de linhas de uma tabela por debaixo dos panos. Veja em *Table Properties > Storage > Row count*. Usa três tabelas diferentes para mostrar um número mais preciso de linhas ao custo de adicionar JOINs na consulta. São necessários dois paramêtros: (1) o nome da tabela e (2) o nome do *schema*.

```SQL
SELECT CAST(p.rows AS float)
FROM sys.tables AS tbl
INNER JOIN sys.indexes AS idx ON idx.object_id = tbl.object_id and idx.index_id < 2
INNER JOIN sys.partitions AS p ON p.object_id=CAST(tbl.object_id AS int)
AND p.index_id=idx.index_id
WHERE ((tbl.name=N'<table_name>'
AND SCHEMA_NAME(tbl.schema_id)='<schema_name>'));
```
<br/>

#### Método 4 — O Moderninho 
Talvez o método com o melhor custo-benefício pois retorna um valor aproximado número de linhas usando uma consulta simples à uma [Dynamic Management Views](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/database-related-dynamic-management-views-transact-sql?view=azuresqldb-current&viewFallbackFrom=sql-server-ver16), nesse caso à *sys.dm_db_partition_stats*. As DMVs permitem ao usuário um mecanismo para ver informações do *Database Engine*, o chamado de SQLOS (*SQL Operating System*), que foi introduzido no SQL Server 2005, de forma rápida, simplificada e leve. 

```SQL
SELECT SUM (row_count)
FROM sys.dm_db_partition_stats
WHERE object_id=OBJECT_ID('<table_name>')   
AND (index_id = 0 or index_id = 1);
```
<br/>

#### BÔNUS:

#### Método 5 - O Velhinho que ainda funciona
Por que não comentar sobre a `sp_spaceused`? Esse procedimento de sistema foi herdado do código do Sybase (quem não sabe, o SQL Server é derivado do código base do Sybase). Além do número de linhas retorna, também, espaço em disco reservado e espaço em disco usado por uma tabela. Particularmente eu não gosto de usar os procedimentos legados do SQL Server (mas não estou dizendo que são ruins ou que não funcionam) mas sim as DMVs, como no Método 4, que são mais modernas e oferecem uma acesso leve e simplificado.

```SQL
EXEC sp_spaceused N'dbo.Employees'; 
```
<br/>

I know this is only SQL but I like it.