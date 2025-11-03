/******************************************************
 Script T-SQL de ejemplo ampliado
 Contiene: tablas, datos de prueba, funciones, proc,
 variables, condicionales, cursores, SQL dinámico,
 transacciones, TRY...CATCH, tablas temporales.
 Autor: Ejemplo educativo
 Fecha: (usar según convenga)
******************************************************/

-- 0. Asegurarnos de usar la base correcta (opcional)
-- USE TuBaseDeDatos;
-- GO

/**************
 1. Tablas de ejemplo
**************/
IF OBJECT_ID('dbo.Clientes', 'U') IS NOT NULL DROP TABLE dbo.Clientes;
IF OBJECT_ID('dbo.Productos', 'U') IS NOT NULL DROP TABLE dbo.Productos;
IF OBJECT_ID('dbo.Ventas', 'U') IS NOT NULL DROP TABLE dbo.Ventas;
IF OBJECT_ID('dbo.DetalleVenta', 'U') IS NOT NULL DROP TABLE dbo.DetalleVenta;

CREATE TABLE dbo.Clientes (
    ClienteID INT IDENTITY(1,1) PRIMARY KEY,
    Nombre NVARCHAR(150) NOT NULL,
    Email NVARCHAR(200) NULL,
    FechaRegistro DATE NOT NULL DEFAULT GETDATE(),
    EsActivo BIT NOT NULL DEFAULT 1,
    Saldo DECIMAL(12,2) NOT NULL DEFAULT 0.00
);
GO

CREATE TABLE dbo.Productos (
    ProductoID INT IDENTITY(1,1) PRIMARY KEY,
    Codigo NVARCHAR(50) NOT NULL UNIQUE,
    Nombre NVARCHAR(200) NOT NULL,
    Precio DECIMAL(12,2) NOT NULL,
    Stock INT NOT NULL DEFAULT 0,
    EsActivo BIT NOT NULL DEFAULT 1
);
GO

CREATE TABLE dbo.Ventas (
    VentaID INT IDENTITY(1,1) PRIMARY KEY,
    ClienteID INT NOT NULL FOREIGN KEY REFERENCES dbo.Clientes(ClienteID),
    FechaVenta DATETIME NOT NULL DEFAULT GETDATE(),
    Total DECIMAL(14,2) NOT NULL DEFAULT 0.00,
    Estado NVARCHAR(20) NOT NULL DEFAULT 'Pendiente'
);
GO

CREATE TABLE dbo.DetalleVenta (
    DetalleID INT IDENTITY(1,1) PRIMARY KEY,
    VentaID INT NOT NULL FOREIGN KEY REFERENCES dbo.Ventas(VentaID),
    ProductoID INT NOT NULL FOREIGN KEY REFERENCES dbo.Productos(ProductoID),
    Cantidad INT NOT NULL,
    PrecioUnitario DECIMAL(12,2) NOT NULL,
    Subtotal AS (Cantidad * PrecioUnitario) PERSISTED
);
GO

/**************
 2. Insertar datos de prueba
**************/
INSERT INTO dbo.Clientes (Nombre, Email, Saldo)
VALUES
('Ana Pérez', 'ana.perez@example.com', 0.00),
('Luis Gómez', 'luis.gomez@example.com', 20.00),
('María Ruiz', 'maria.ruiz@example.com', 100.00);

INSERT INTO dbo.Productos (Codigo, Nombre, Precio, Stock)
VALUES
('P-001', 'Camiseta Algodón', 25.50, 100),
('P-002', 'Pantalón Denim', 45.00, 30),
('P-003', 'Zapatos Deportivos', 80.00, 20),
('P-004', 'Gorra Unisex', 15.00, 200);
GO

/**************
 3. Función escalar: calcular total con impuesto y descuento
    - se muestra por qué usar función: reusable en SELECT/WHERE
**************/
IF OBJECT_ID('dbo.fn_CalcularTotalConImpuesto', 'FN') IS NOT NULL
    DROP FUNCTION dbo.fn_CalcularTotalConImpuesto;
GO

CREATE FUNCTION dbo.fn_CalcularTotalConImpuesto
(
    @subtotal DECIMAL(14,2),
    @impuestoPct DECIMAL(5,2),       -- por ejemplo 18.00 para 18%
    @descuento DECIMAL(14,2) = 0.00  -- descuento fijo
)
RETURNS DECIMAL(14,2)
AS
BEGIN
    DECLARE @total DECIMAL(14,2);
    -- cálculo paso a paso
    DECLARE @impuesto DECIMAL(14,2);
    SET @impuesto = ROUND(@subtotal * (@impuestoPct / 100.0), 2);
    SET @total = (@subtotal + @impuesto) - @descuento;

    IF @total < 0 SET @total = 0.00; -- seguridad

    RETURN @total;
END;
GO

/**************
 4. Función con valor de tabla (Inline Table-Valued Function)
    - útil para reutilizar conjuntos de filas en FROM
**************/
IF OBJECT_ID('dbo.fn_ProductosActivos', 'IF') IS NOT NULL
    DROP FUNCTION dbo.fn_ProductosActivos;
GO

CREATE FUNCTION dbo.fn_ProductosActivos(@minStock INT)
RETURNS TABLE
AS
RETURN
(
    SELECT ProductoID, Codigo, Nombre, Precio, Stock
    FROM dbo.Productos
    WHERE EsActivo = 1 AND Stock >= @minStock
);
GO

/**************
 5. Procedimiento almacenado: Registrar venta segura con transacción
    - uso de variables, control de errores, actualización de stock
**************/
IF OBJECT_ID('dbo.sp_RegistrarVenta', 'P') IS NOT NULL
    DROP PROCEDURE dbo.sp_RegistrarVenta;
GO

CREATE PROCEDURE dbo.sp_RegistrarVenta
(
    @ClienteID INT,
    @Items dbo.DetalleVenta READONLY = NULL -- alternativa: usar TVP si definido
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @VentaID INT;
    DECLARE @Subtotal DECIMAL(14,2) = 0.00;
    DECLARE @Total DECIMAL(14,2);
    DECLARE @ImpuestoPct DECIMAL(5,2) = 18.00; -- por ejemplo IVA 18%
    DECLARE @Descuento DECIMAL(14,2) = 0.00;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Crear la cabecera de venta (total se calculará luego)
        INSERT INTO dbo.Ventas (ClienteID, Total, Estado)
        VALUES (@ClienteID, 0.00, 'Procesando');

        SET @VentaID = SCOPE_IDENTITY();

        -- Para este ejemplo vamos a simular el proceso usando un cursor local
        -- sobre los detalles contenidos en una tabla temporal "##tmpItems" si existen.
        -- En un escenario real se podría usar un TVP (tipo tabla paramétrico).

        IF OBJECT_ID('tempdb..#ItemsTemp') IS NOT NULL DROP TABLE #ItemsTemp;
        CREATE TABLE #ItemsTemp (
            ProductoID INT,
            Cantidad INT,
            PrecioUnitario DECIMAL(12,2)
        );

        -- Si existe una tabla con nombre "##TVP_Items" (por ejemplo) la podríamos insertar
        -- Aquí insertamos datos de ejemplo si Items no es provisto
        INSERT INTO #ItemsTemp (ProductoID, Cantidad, PrecioUnitario)
        SELECT ProductoID, 2, Precio FROM dbo.Productos WHERE ProductoID IN (1,2);

        -- Calcular subtotal e insertar detalle
        DECLARE @pID INT, @cant INT, @pu DECIMAL(12,2);
        DECLARE cur CURSOR LOCAL FAST_FORWARD FOR
            SELECT ProductoID, Cantidad, PrecioUnitario FROM #ItemsTemp;

        OPEN cur;
        FETCH NEXT FROM cur INTO @pID, @cant, @pu;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            -- Verificar stock
            IF EXISTS (SELECT 1 FROM dbo.Productos WHERE ProductoID = @pID AND Stock >= @cant)
            BEGIN
                -- Insertar detalle
                INSERT INTO dbo.DetalleVenta (VentaID, ProductoID, Cantidad, PrecioUnitario)
                VALUES (@VentaID, @pID, @cant, @pu);

                -- Actualizar stock
                UPDATE dbo.Productos
                SET Stock = Stock - @cant
                WHERE ProductoID = @pID;

                -- Sumar al subtotal
                SET @Subtotal = @Subtotal + (@cant * @pu);
            END
            ELSE
            BEGIN
                -- Si no hay stock suficiente, lanzar error controlado
                RAISERROR('Stock insuficiente para ProductoID = %d', 16, 1, @pID);
            END

            FETCH NEXT FROM cur INTO @pID, @cant, @pu;
        END

        CLOSE cur;
        DEALLOCATE cur;

        -- Aplicar función para calcular total con impuesto
        SET @Total = dbo.fn_CalcularTotalConImpuesto(@Subtotal, @ImpuestoPct, @Descuento);

        -- Actualizar total en cabecera
        UPDATE dbo.Ventas
        SET Total = @Total, Estado = 'Completado'
        WHERE VentaID = @VentaID;

        COMMIT TRANSACTION;

        PRINT 'Venta registrada con ID: ' + CAST(@VentaID AS VARCHAR(20));
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;

        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrNum INT = ERROR_NUMBER();
        PRINT 'Ocurrió un error: ' + @ErrMsg;
        -- Re-lanzar el error
        THROW;
    END CATCH
END;
GO

/**************
 6. Procedimiento: Aplicar descuentos masivos usando SQL dinámico
    - Ejemplo de manejo de variables y SQL dinámico seguro
**************/
IF OBJECT_ID('dbo.sp_AplicarDescuentoPorCategoria', 'P') IS NOT NULL
    DROP PROCEDURE dbo.sp_AplicarDescuentoPorCategoria;
GO

CREATE PROCEDURE dbo.sp_AplicarDescuentoPorCategoria
(
    @PorcentajeDescuento DECIMAL(5,2),
    @Condicion NVARCHAR(400) = NULL  -- condición para filtrar productos (ej: "Precio > 30")
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @sql NVARCHAR(MAX);
    DECLARE @params NVARCHAR(500);

    SET @sql = N'UPDATE dbo.Productos SET Precio = ROUND(Precio * (1 - @pct/100.0), 2) WHERE EsActivo = 1';
    IF @Condicion IS NOT NULL AND LEN(RTRIM(@Condicion)) > 0
    BEGIN
        SET @sql = @sql + N' AND (' + @Condicion + N')';
    END

    SET @params = N'@pct DECIMAL(5,2)';

    BEGIN TRY
        EXEC sp