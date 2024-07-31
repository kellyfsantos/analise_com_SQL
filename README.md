# Exercício de SQL e Python para criação de tabelas de clientes e vendedores e posteriores consultas realizadas durante a rotina

**Criação da tabela de clientes**
```DROP TABLE IF EXISTS tb_customers
CREATE TABLE tb_customers (
	 customerId INT IDENTITY(1,1) PRIMARY KEY
	,customerDoc CHAR(14)
	,firstName VARCHAR(32)
	,lastName VARCHAR(32)
	,birthDate DATE
)
```
**Criação da tabela de produtos**

```DROP TABLE IF EXISTS tb_products;
CREATE TABLE tb_products (
    productId INT IDENTITY(1,1) PRIMARY KEY,
    productName VARCHAR(50),
    price DECIMAL(10, 2)
);
C
**Criação da tabela de pedidos**

```DROP TABLE IF EXISTS tb_orders;
CREATE TABLE tb_orders (
    orderId INT IDENTITY(1,1) PRIMARY KEY,
    customerId INT FOREIGN KEY REFERENCES tb_customers(customerId),
    orderDate DATE
);
```
**Criação da tabela de itens de pedidos**

```DROP TABLE IF EXISTS tb_order_items;
CREATE TABLE tb_order_items (
    orderItemId INT IDENTITY(1,1) PRIMARY KEY,
    orderId INT FOREIGN KEY REFERENCES tb_orders(orderId),
    productId INT FOREIGN KEY REFERENCES tb_products(productId),
    quantity INT
);
```
**Índices para otimização de consultas**

```CREATE NONCLUSTERED INDEX ix_customerDoc on tb_customers (customerDoc) INCLUDE (customerId)
CREATE NONCLUSTERED INDEX ix_customerId ON tb_orders (customerId);
CREATE NONCLUSTERED INDEX ix_orderId ON tb_order_items (orderId);
CREATE NONCLUSTERED INDEX ix_productId ON tb_order_items (productId);
```

**População de dados na tabela de tb_customers**

```INSERT INTO tb_customers (customerDoc, firstName, lastName, birthDate)
SELECT
    RIGHT('00000000000' + CAST(ABS(CHECKSUM(NEWID())) % 100000000000000 AS VARCHAR), 11) AS customerDoc,
    LEFT(NEWID(), 32) AS firstName,
    LEFT(NEWID(), 32) AS lastName,
    DATEADD(DAY, ABS(CHECKSUM(NEWID())) % 36525, '1906-01-01') AS birthDate
FROM
    (SELECT TOP 1000 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS r FROM sys.columns) AS Numbers;
```

**População de dados na tabela de produtos**

```INSERT INTO tb_products (productName, price)
VALUES
    ('Notebook', 2100.00),
    ('Smartphone', 1200.00),
    ('Tablet', 800.00),
    ('Headphones', 150.00),
    ('Monitor', 500.00),
    ('Keyboard', 80.00),
    ('Mouse', 40.00),
    ('Printer', 250.00),
    ('Webcam', 100.00),
    ('Speakers', 75.00),
    ('External Hard Drive', 130.00),
    ('USB Flash Drive', 20.00),
    ('Router', 90.00),
    ('Smartwatch', 250.00),
    ('Fitness Tracker', 100.00);
```

**População de dados na tabela de pedidos**

```DECLARE @COUNT INT = 0, @LIM INT = 5
WHILE @COUNT <= @LIM BEGIN
	INSERT INTO tb_orders (customerId, orderDate)
	SELECT
		customerId,
		DATEADD(DAY, ABS(CHECKSUM(NEWID())) % 365, GETDATE()) AS orderDate
	FROM
		tb_customers
	WHERE
		customerId <= (SELECT ABS(CHECKSUM(NEWID())) % 1000)

	SET @COUNT += 1

END
```
**População de dados na tabela de itens de pedidos**

```DECLARE @COUNT1 INT = 0, @LIM1 INT = 10
WHILE @COUNT1 <= @LIM1 BEGIN
INSERT INTO tb_order_items (orderId, productId, quantity)
SELECT
    o.orderId,
    p.productId,
    ABS(CHECKSUM(NEWID())) % 5 + 1 AS quantity 
FROM
    tb_orders o
    CROSS JOIN tb_products p
WHERE
    ABS(CHECKSUM(NEWID())) % 10 < 3; 

	SET @COUNT1 += 1
END
```

## 1. Criando uma consulta que retorne apenas o item mais pedido e a quantidade total de pedidos.

```SELECT 
    TOP 1
    p.productName,
    SUM(oi.quantity) AS totalQuantity
FROM 
    tb_order_items oi
JOIN 
    tb_products p ON oi.productId = p.productId
GROUP BY 
    p.productName
ORDER BY 
    totalQuantity DESC;
```
   
 ## 2.Criando uma consulta que retorne todos os clientes que realizaram mais de 4 pedidos no último ano em ordem decrescente.
   
 ```SELECT 
    c.customerId,
    c.customerDoc,
    c.firstName,
    c.lastName,
    c.birthDate
    COUNT(o.orderId) AS totalOrders
FROM 
    tb_customers c
JOIN 
    tb_orders o ON c.customerId = o.customerId
WHERE 
    o.orderDate >= DATEADD(YEAR, -1, GETDATE())
GROUP BY 
    c.customerId, c.customerDoc, c.firstName, c.lastName, c.birthDate
HAVING 
    COUNT(o.orderId) > 4
ORDER BY 
    totalOrders DESC;
```
   
 ## 3. Criando uma consulta de quantos pedidos foram realizados em cada mês do último ano.

```SELECT 
    YEAR(orderDate) AS orderYear,
    MONTH(orderDate) AS orderMonth,
    COUNT(orderId) AS totalOrders
FROM 
    tb_orders
WHERE 
    orderDate >= DATEADD(YEAR, -1, GETDATE())
GROUP BY 
    YEAR(orderDate),
    MONTH(orderDate)
ORDER BY 
    orderYear ASC,
    orderMonth ASC;
```
   
## 4. Criando uma consulta que retorne APENAS os campos "productName" e "totalAmount" dos 5 produtos mais pedidos.

```SELECT 
    TOP 5
    p.productName,
    SUM(oi.quantity) AS totalAmount
FROM 
    tb_order_items oi
JOIN 
    tb_products p ON oi.productId = p.productId
GROUP BY 
    p.productName
ORDER BY 
    totalAmount DESC;
```
   
## 5. Criando uma consulta liste todos os clientes que não realizaram nenhum pedido.

```SELECT 
    c.customerId,
    c.customerDoc,
    c.firstName,
    c.lastName,
    c.birthDate
FROM 
    tb_customers c
LEFT JOIN 
    tb_orders o ON c.customerId = o.customerId
WHERE 
    o.orderId IS NULL;
```
   
## 6. Criando uma consulta que retorne a data e o nome do produto do último pedido realizado pelos clientes onde o customerId são 94, 130, 300 e 1000.

```SELECT 
    o.customerId,
    o.orderDate,
    p.productName
FROM 
    tb_orders o
JOIN 
    tb_order_items oi ON o.orderId = oi.orderId
JOIN 
    tb_products p ON oi.productId = p.productId
WHERE 
    o.customerId IN (94, 130, 300, 1000)
    AND o.orderDate = (
        SELECT MAX(o2.orderDate)
        FROM tb_orders o2
        WHERE o2.customerId = o.customerId
    )
ORDER BY 
    o.customerId, 
    o.orderDate DESC;
```
   
## 7. Criando uma nova tabela para armazenar informações sobre vendedores com base na estrutura das tabelas fornecidas (tb_order_items, tb_orders, tb_products, tb_customers).
## A tabela segue os conceitos básicos de modelo relacional. 
   
```DROP TABLE IF EXISTS tb_sellers;
CREATE TABLE tb_sellers (
    sellerId INT IDENTITY(1,1) PRIMARY KEY,
    sellerDoc CHAR(14),
    sellerName VARCHAR(50) NOT NULL,
    email VARCHAR(100),
    phoneNumber VARCHAR(15)
```

**Para correlacionar os vendedores aos pedidos realizados, foi adicionada mais uma colun na Tabela de Pedidos:**
 
```ALTER TABLE tb_orders
ADD sellerId INT;
```

## 8. Crie uma procedure que insira dados na tabela de vendedores criada anteriormente.

```CREATE PROCEDURE InsertSeller
    @sellerName VARCHAR(50),
    @sellerDoc CHAR(14),
    @email VARCHAR(100),
    @phoneNumber VARCHAR(15)
AS
BEGIN

	DECLARE @existingSellerId INT;
```

## a. Validar se o vendedor já existe na tabela.

```SELECT @existingSellerId = sellerId
    FROM tb_sellers
    WHERE sellerName = @sellerName;
IF @existingSellerId IS NOT NULL
    BEGIN
        PRINT 'O vendedor está cadastrado sob o ID: ' + CAST(@existingSellerId AS VARCHAR);
    END
    ELSE
    BEGIN
```
   
## b. Se o vendedor não existir, inserir um novo registro com os dados fornecidos.
	    
		```PRINT 'O vendedor com o ID ' + @sellerId + ' não está cadastrado.';
		PRINT 'Por favor, execute a procedure InsertSeller para adicionar um novo vendedor com os dados fornecidos.';
    END
END;	
```
```INSERT INTO tb_sellers (sellerName, email, phoneNumber, hireDate)
VALUES (@sellerName, @sellerDoc, @email, @phoneNumber);
```

## c. Retornar uma mensagem indicando se a inserção foi bem-sucedida ou se o vendedor já está na tabela.

    ```PRINT 'Inserção bem-sucedida: novo vendedor adicionado com o ID ' + @sellerId + '.';
    END
END; ```

## Escreva a implementação completa da procedure, incluindo a validação e a mensagem de retorno.

```EXEC InsertSeller 
    @sellerName = 'Evandro dos Santos Rosa', 
    @sellerDoc = '08930098903'
    @email = 'rosa.evandro@empresa.com', 
    @phoneNumber = '19984050066';
```

## 9. Código em Python que se conecte a um banco de dados SQL Server e chame a procedure criada anteriormente para inserir um novo vendedor na tabela criada:

## Importação da biblioteca pyodbc:
   
```import pyodbc```

## Criação variáveis para as credenciais de conexão:

```server = '<server-address>'  
database = '<database_name>'  
username = '<username>'  
password = '<password>'
```

## Criação de uma variável de cadeia de conexão usando interpolação de cadeia de caracteres:

```connectionString = f'DRIVER={{ODBC Driver 17 for SQL Server}};SERVER={server};DATABASE={database};UID={username};PWD={password}'```

## Uso da função pyodbc.connect para conectar a um banco de dados SQL:

```conn = pyodbc.connect(connectionString) conn = pyodbc.connect(connectionString)```

## Apresentando os parâmetros para a procedure e chamando a procedure:

```seller_name = 'Carlos Santos'
seler_doc = '08930098903'
email = 'rosa.evandro@empresa.com'
phone_number = '19984050066'
```

```cursor.execute("""
	EXEC InsertSeller 
		@sellerName = ?, 
		@sellerDoc = ?,
		@email = ?, 
		@phoneNumber = ?
	""", (seller_name, seller_doc, email, phone_number))
```

	
## 10. Criando um código em Python que carregue em um “data frame” a tabela pedidos e a partir dele retorne os 10 produtos mais pedidos com as colunas "productName" e "numberOfOrders" em ordem decrescente:
	
**Importação das bibliotecas pyodbc e pandas:**

```import pyodbc
import pandas as pd```

```server = 'server_name'  
database = 'database_name'  
username = 'username' 
password = 'password'
``` 

```connectionString = f'DRIVER={{ODBC Driver 17 for SQL Server}};SERVER={server};DATABASE={database};UID={username};PWD={password}'```

**Consulta dos 10 produtos mais pedidos:**

```query =
SELECT 
    p.productName,
    COUNT(oi.orderItemId) AS numberOfOrders
FROM 
    tb_order_items oi
JOIN 
    tb_products p ON oi.productId = p.productId
GROUP BY 
    p.productName
ORDER BY 
    numberOfOrders DESC
LIMIT 10;
```


```with pyodbc.connect(conn_str) as conn:
df = pd.read_sql(query, conn)
	print(df)
df_sorted = df.sort_values(by='numberOfOrders', ascending=False)
print(df_sorted.head(10))
```
