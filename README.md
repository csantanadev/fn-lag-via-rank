# Como fazer a função lag() do postgresql de um jeito diferente 

O objetivo desde post é mostrar como é possível implementar a função lag() do postgreSQL utilizando outro comando de janela (rank).

## Conceito

A função PostgreSQL LAG fornece acesso a uma linha que vem antes da linha atual em um deslocamento físico especificado. Em outras palavras, a partir da linha atual, a função LAG () pode acessar os dados da linha anterior, ou da linha anterior à linha anterior, e assim por diante.


![1](https://user-images.githubusercontent.com/20893840/134688216-c90c684c-ce07-46c3-b5d8-f3e7c0052aa2.png)


## Função lag() na prática

Para o nosso exemplo utilizamos dados mokados simulando o total de vendas ao longo dos anos/meses: 

```sql
WITH dados AS
  (SELECT '01' mes, '2021' ano, 1520 total_vendas
   UNION 
   SELECT '02' mes, '2021' ano, 1800 total_vendas
   UNION 
   SELECT '03' mes, '2021' ano, 2360 total_vendas
   UNION 
   SELECT '04' mes, '2021' ano, 1409 total_vendas
   UNION 
   SELECT '05' mes, '2021' ano, 1566 total_vendas
   UNION 
   SELECT '06' mes, '2021' ano, 4121 total_vendas)

select * from dados
order by 2, 1
```
![2](https://user-images.githubusercontent.com/20893840/134689243-a1f2ae61-6088-4c89-86e9-2c05c0e78687.png)

### Com o uso do lag() foi possível acessar a linha anterior.

```sql
WITH dados AS
  (SELECT '01' mes, '2021' ano, 1520 total_vendas
   UNION 
   SELECT '02' mes, '2021' ano, 1800 total_vendas
   UNION 
   SELECT '03' mes, '2021' ano, 2360 total_vendas
   UNION 
   SELECT '04' mes, '2021' ano, 1409 total_vendas
   UNION 
   SELECT '05' mes, '2021' ano, 1566 total_vendas
   UNION 
   SELECT '06' mes, '2021' ano, 4121 total_vendas)
SELECT v.mes,
          v.ano,
          v.total_vendas::decimal,
          lag(total_vendas, 1) OVER () mes_anterior
   FROM
     (SELECT mes,
             ano,
             total_vendas
      FROM dados
      ORDER BY 1) v
```
![3](https://user-images.githubusercontent.com/20893840/134689935-7e671436-2f4b-4ff4-a5b5-475cb54b48dd.png)

### Calculando o percentual (%) de crescimento

```sql
WITH dados AS
  (SELECT '01' mes, '2021' ano, 1520 total_vendas
   UNION 
   SELECT '02' mes, '2021' ano, 1800 total_vendas
   UNION 
   SELECT '03' mes, '2021' ano, 2360 total_vendas
   UNION 
   SELECT '04' mes, '2021' ano, 1409 total_vendas
   UNION 
   SELECT '05' mes, '2021' ano, 1566 total_vendas
   UNION 
   SELECT '06' mes, '2021' ano, 4121 total_vendas)

SELECT v.mes,
       v.ano,
       v.total_vendas,
       v.mes_anterior,
       round((v.total_vendas - v.mes_anterior) / v.mes_anterior, 2) * 100 AS percentual_crescimento
FROM
  (SELECT v.mes,
          v.ano,
          v.total_vendas::decimal,
          lag(total_vendas, 1) OVER () mes_anterior
   FROM
     (SELECT mes,
             ano,
             total_vendas
      FROM dados
      ORDER BY 1) v) v
```

![image](https://user-images.githubusercontent.com/20893840/134691067-f1936aa0-cfef-48f6-a512-916c42ef8905.png)

## Utilizando a Função rank() para simular o lag()

1º Ranqueamos os dados por ano, mês

```sql
WITH dados AS
  (...)

SELECT mes,
          ano,
          total_vendas,
          rank() over( ORDER BY ano, mes) rnk
   FROM dados
   ORDER BY 1
```

![image](https://user-images.githubusercontent.com/20893840/134691969-ba607901-f907-41fe-8f79-707b003185a6.png)

2º Fazemos um left outer join com a mesma resultset, porém com  <b>rnk = rnk + 1 </b>  

```sql
WITH dados AS
  (...)
   
SELECT v.mes,
       v.ano,
       v.total_vendas::decimal,
       v2.total_vendas::decimal mes_anterior
FROM
  (SELECT mes,
          ano,
          total_vendas,
          rank() over(
                      ORDER BY ano, mes) rnk
   FROM dados
   ORDER BY 1) v
LEFT OUTER JOIN
  (SELECT v.mes,
          v.ano,
          v.total_vendas::decimal,
          v.rnk
   FROM
     (SELECT mes,
             ano,
             total_vendas,
             rank() over(
                         ORDER BY ano, mes) rnk
      FROM dados
      ORDER BY 1) v) v2 ON (v.rnk = v2.rnk + 1)
```

3º Efetuamos o cálculo

```sql
WITH dados AS
  (...)
SELECT v.mes,
       v.ano,
       v.total_vendas::decimal,
       v2.total_vendas::decimal mes_anterior,
       round((v.total_vendas - v2.total_vendas::decimal) / v2.total_vendas::decimal, 2) * 100 AS percentual_crescimento
FROM
  (SELECT mes,
          ano,
          total_vendas,
          rank() over(
                      ORDER BY ano, mes) rnk
   FROM dados
   ORDER BY 1) v
LEFT OUTER JOIN
  (SELECT v.mes,
          v.ano,
          v.total_vendas::decimal,
          v.rnk
   FROM
     (SELECT mes,
             ano,
             total_vendas,
             rank() over(
                         ORDER BY ano, mes) rnk
      FROM dados
      ORDER BY 1) v) v2 ON (v.rnk = v2.rnk + 1)
```

![image](https://user-images.githubusercontent.com/20893840/134693528-ce4e5e70-0508-4ba5-acb5-bcd04c3e0b2e.png)

## Considerações finais

Foi utilizado a função rank() para que o exemplo ficasse mais didático, porém é possível utilizar a função agregada count() desde que os dados de mês e ano estivessem devidamente ordenados.

```sql
SELECT mes,
          ano,
          total_vendas,
          count(*) over(ORDER BY ano, mes) rnk
   FROM dados
   ORDER BY ano, mes
```

![image](https://user-images.githubusercontent.com/20893840/134694425-aba34a92-a95a-4127-973c-e752d940d58d.png)


