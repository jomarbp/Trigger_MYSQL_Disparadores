# 🚀 Tutorial: Triggers en MySQL con XAMPP

## Sistema de Inventario con Disparadores Automáticos

[![MySQL](https://img.shields.io/badge/MySQL-8.0+-blue.svg)](https://www.mysql.com/)
[![XAMPP](https://img.shields.io/badge/XAMPP-8.0+-orange.svg)](https://www.apachefriends.org/)
[![PHP](https://img.shields.io/badge/PHP-8.0+-purple.svg)](https://www.php.net/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

### 📋 Descripción

Tutorial práctico paso a paso para aprender a crear y gestionar **Triggers (Disparadores)** en MySQL usando XAMPP. Implementaremos un sistema de inventario básico donde los triggers automáticamente gestionan el stock, validan datos y mantienen auditorías.

### 🎯 Objetivos de Aprendizaje

Al completar este tutorial, serás capaz de:
- ✅ Crear triggers BEFORE y AFTER
- ✅ Implementar validaciones automáticas
- ✅ Mantener auditorías de cambios
- ✅ Generar alertas automáticas
- ✅ Gestionar triggers existentes

---

## 📋 Tabla de Contenidos

- [Requisitos Previos](#-requisitos-previos)
- [Instalación y Configuración](#-instalación-y-configuración)
- [Estructura de la Base de Datos](#-estructura-de-la-base-de-datos)
- [Implementación de Triggers](#-implementación-de-triggers)
- [Pruebas y Validación](#-pruebas-y-validación)
- [Consultas de Análisis](#-consultas-de-análisis)
- [Gestión de Triggers](#-gestión-de-triggers)
- [Troubleshooting](#-troubleshooting)

---

## 🔧 Requisitos Previos

### Software Necesario
- **XAMPP** (Apache + MySQL + PHP)
- Navegador web moderno
- Editor de texto (VS Code recomendado)

### Conocimientos Previos
- SQL básico (SELECT, INSERT, UPDATE, DELETE)
- Conceptos de bases de datos relacionales
- Uso básico de phpMyAdmin

---

## 🚀 Instalación y Configuración

### Paso 1: Configurar XAMPP

1. **Iniciar XAMPP Control Panel**
2. **Activar servicios necesarios:**
   ```
   ✅ Apache
   ✅ MySQL
   ```
3. **Verificar estado:** Ambos servicios deben aparecer en verde

### Paso 2: Acceder a phpMyAdmin

1. Abrir navegador web
2. Navegar a: `http://localhost/phpmyadmin`
3. **Credenciales por defecto:**
   - Usuario: `root`
   - Contraseña: *(vacía)*

---

## 🗄️ Estructura de la Base de Datos

### Crear Base de Datos

```sql
-- Crear la base de datos
CREATE DATABASE inventario_tienda;

-- Seleccionar la base de datos
USE inventario_tienda;
```

### Tabla: Productos

```sql
CREATE TABLE productos (
    id_producto INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    precio DECIMAL(10,2) NOT NULL,
    stock_actual INT NOT NULL DEFAULT 0,
    stock_minimo INT NOT NULL DEFAULT 5,
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Tabla: Ventas

```sql
CREATE TABLE ventas (
    id_venta INT AUTO_INCREMENT PRIMARY KEY,
    id_producto INT NOT NULL,
    cantidad INT NOT NULL,
    precio_unitario DECIMAL(10,2) NOT NULL,
    total DECIMAL(10,2) NOT NULL,
    fecha_venta TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

### Tabla: Historial de Stock

```sql
CREATE TABLE historial_stock (
    id_historial INT AUTO_INCREMENT PRIMARY KEY,
    id_producto INT NOT NULL,
    stock_anterior INT NOT NULL,
    stock_nuevo INT NOT NULL,
    motivo VARCHAR(100) NOT NULL,
    fecha_cambio TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

### Tabla: Alertas de Stock

```sql
CREATE TABLE alertas_stock (
    id_alerta INT AUTO_INCREMENT PRIMARY KEY,
    id_producto INT NOT NULL,
    mensaje VARCHAR(200) NOT NULL,
    fecha_alerta TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    solucionada BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto)
);
```

### Datos de Prueba

```sql
-- Insertar productos de ejemplo
INSERT INTO productos (nombre, precio, stock_actual, stock_minimo) VALUES
('Laptop Dell', 1500.00, 10, 3),
('Mouse Logitech', 25.50, 50, 10),
('Teclado Mecánico', 89.99, 8, 5),
('Monitor 24"', 299.99, 15, 4),
('Auriculares Sony', 199.50, 2, 5);
```

---

## ⚡ Implementación de Triggers

### 1️⃣ Trigger: Actualizar Stock Después de Venta

**Función:** Reduce automáticamente el stock cuando se registra una venta y mantiene historial.

```sql
DELIMITER //

CREATE TRIGGER actualizar_stock_venta
    AFTER INSERT ON ventas
    FOR EACH ROW
BEGIN
    -- Actualizar el stock del producto
    UPDATE productos 
    SET stock_actual = stock_actual - NEW.cantidad
    WHERE id_producto = NEW.id_producto;
    
    -- Registrar en historial
    INSERT INTO historial_stock (id_producto, stock_anterior, stock_nuevo, motivo)
    VALUES (
        NEW.id_producto,
        (SELECT stock_actual + NEW.cantidad FROM productos WHERE id_producto = NEW.id_producto),
        (SELECT stock_actual FROM productos WHERE id_producto = NEW.id_producto),
        CONCAT('Venta de ', NEW.cantidad, ' unidades')
    );
END//

DELIMITER ;
```

### 2️⃣ Trigger: Validar Stock Antes de Venta

**Función:** Impide registrar ventas si no hay stock suficiente.

```sql
DELIMITER //

CREATE TRIGGER validar_stock_venta
    BEFORE INSERT ON ventas
    FOR EACH ROW
BEGIN
    DECLARE stock_disponible INT;
    
    -- Obtener stock actual
    SELECT stock_actual INTO stock_disponible
    FROM productos 
    WHERE id_producto = NEW.id_producto;
    
    -- Validar si hay suficiente stock
    IF stock_disponible < NEW.cantidad THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Stock insuficiente para realizar la venta';
    END IF;
END//

DELIMITER ;
```

### 3️⃣ Trigger: Alertas de Stock Bajo

**Función:** Genera alertas automáticas cuando el stock está por debajo del mínimo.

```sql
DELIMITER //

CREATE TRIGGER alerta_stock_bajo
    AFTER UPDATE ON productos
    FOR EACH ROW
BEGIN
    -- Solo si cambió el stock y está por debajo del mínimo
    IF NEW.stock_actual != OLD.stock_actual AND NEW.stock_actual <= NEW.stock_minimo THEN
        INSERT INTO alertas_stock (id_producto, mensaje)
        VALUES (
            NEW.id_producto,
            CONCAT('ALERTA: El producto "', NEW.nombre, '" tiene stock bajo (',
                   NEW.stock_actual, ' unidades). Mínimo requerido: ', NEW.stock_minimo)
        );
    END IF;
END//

DELIMITER ;
```

### 4️⃣ Trigger: Cálculo Automático de Total

**Función:** Calcula automáticamente el total de la venta.

```sql
DELIMITER //

CREATE TRIGGER calcular_total_venta
    BEFORE INSERT ON ventas
    FOR EACH ROW
BEGIN
    -- Obtener precio actual del producto
    SELECT precio INTO NEW.precio_unitario
    FROM productos 
    WHERE id_producto = NEW.id_producto;
    
    -- Calcular total
    SET NEW.total = NEW.cantidad * NEW.precio_unitario;
END//

DELIMITER ;
```

---

## 🧪 Pruebas y Validación

### Verificar Estado Inicial

```sql
-- Ver productos actuales
SELECT * FROM productos;

-- Verificar tablas vacías
SELECT * FROM ventas;
SELECT * FROM historial_stock;
SELECT * FROM alertas_stock;
```

### Prueba 1: Venta Exitosa

```sql
-- Registrar una venta normal
INSERT INTO ventas (id_producto, cantidad) VALUES (1, 2);

-- Verificar cambios automáticos
SELECT * FROM productos WHERE id_producto = 1;
SELECT * FROM ventas;
SELECT * FROM historial_stock;
```

**Resultado esperado:**
- ✅ Stock de Laptop Dell reducido en 2 unidades
- ✅ Registro en historial_stock
- ✅ Total calculado automáticamente

### Prueba 2: Validación de Stock Insuficiente

```sql
-- Intentar vender más de lo disponible (debería fallar)
INSERT INTO ventas (id_producto, cantidad) VALUES (5, 10);
```

**Resultado esperado:**
- ❌ Error: "Stock insuficiente para realizar la venta"

### Prueba 3: Alerta de Stock Bajo

```sql
-- Vender auriculares para generar alerta
INSERT INTO ventas (id_producto, cantidad) VALUES (5, 1);

-- Ver la alerta generada
SELECT * FROM alertas_stock;
```

**Resultado esperado:**
- ✅ Alerta automática de stock bajo
- ✅ Stock de auriculares en 1 (menor que mínimo de 5)

---

## 📊 Consultas de Análisis

### Reporte de Ventas por Producto

```sql
SELECT 
    p.nombre,
    COUNT(v.id_venta) as total_ventas,
    SUM(v.cantidad) as unidades_vendidas,
    SUM(v.total) as ingresos_totales
FROM productos p
LEFT JOIN ventas v ON p.id_producto = v.id_producto
GROUP BY p.id_producto, p.nombre;
```

### Productos con Stock Bajo

```sql
SELECT 
    nombre,
    stock_actual,
    stock_minimo,
    (stock_minimo - stock_actual) as unidades_faltantes
FROM productos 
WHERE stock_actual <= stock_minimo;
```

### Historial de Cambios de Stock

```sql
SELECT 
    p.nombre,
    h.stock_anterior,
    h.stock_nuevo,
    h.motivo,
    h.fecha_cambio
FROM historial_stock h
JOIN productos p ON h.id_producto = p.id_producto
ORDER BY h.fecha_cambio DESC;
```

---

## 🔧 Gestión de Triggers

### Listar Triggers Existentes

```sql
-- Ver todos los triggers de la base de datos
SHOW TRIGGERS;

-- Ver triggers específicos de una tabla
SHOW TRIGGERS LIKE 'ventas';
```

### Eliminar un Trigger

```sql
-- Sintaxis general
DROP TRIGGER IF EXISTS nombre_trigger;

-- Ejemplo
DROP TRIGGER IF EXISTS actualizar_stock_venta;
```

### Modificar un Trigger

```sql
-- Para modificar un trigger:
-- 1. Eliminar el trigger existente
DROP TRIGGER IF EXISTS actualizar_stock_venta;

-- 2. Crear la nueva versión
DELIMITER //
CREATE TRIGGER actualizar_stock_venta
    AFTER INSERT ON ventas
    FOR EACH ROW
BEGIN
    -- Nueva lógica aquí
END//
DELIMITER ;
```

---

## 🔍 Troubleshooting

### Errores Comunes y Soluciones

| Error | Causa | Solución |
|-------|--------|----------|
| `Table 'xxx' doesn't exist` | Tabla no creada | Verificar creación de todas las tablas |
| `Trigger 'xxx' already exists` | Trigger duplicado | Usar `DROP TRIGGER IF EXISTS` antes de crear |
| `Can't update table in stored function/trigger` | Trigger recursivo | Evitar modificar la misma tabla que activó el trigger |
| `Unknown column 'NEW.xxx'` | Columna inexistente | Verificar nombres de columnas en la tabla |

### Debugging de Triggers

```sql
-- Ver información detallada de triggers
SELECT 
    TRIGGER_NAME,
    EVENT_MANIPULATION,
    EVENT_OBJECT_TABLE,
    ACTION_TIMING,
    ACTION_STATEMENT
FROM INFORMATION_SCHEMA.TRIGGERS 
WHERE TRIGGER_SCHEMA = 'inventario_tienda';
```

---

## 📚 Conceptos Clave

### Tipos de Triggers
- **BEFORE**: Se ejecuta antes de la operación
- **AFTER**: Se ejecuta después de la operación
- **INSERT/UPDATE/DELETE**: Eventos que activan el trigger

### Referencias Especiales
- **NEW**: Valores nuevos (INSERT, UPDATE)
- **OLD**: Valores anteriores (UPDATE, DELETE)

### Mejores Prácticas
- ✅ Mantener triggers simples y eficientes
- ✅ Documentar la lógica de cada trigger
- ✅ Usar nombres descriptivos
- ✅ Probar exhaustivamente
- ⚠️ Evitar triggers recursivos
- ⚠️ No abusar para lógica compleja

---

## 🤝 Contribuir

¿Encontraste un error o tienes una mejora? ¡Las contribuciones son bienvenidas!

1. Fork el proyecto
2. Crea una rama para tu feature (`git checkout -b feature/AmazingFeature`)
3. Commit tus cambios (`git commit -m 'Add some AmazingFeature'`)
4. Push a la rama (`git push origin feature/AmazingFeature`)
5. Abre un Pull Request

---

## 📝 Licencia

Este proyecto está bajo la Licencia MIT. Ver `LICENSE` para más detalles.

---

## 👨‍💻 Autor

**Tu Nombre**
- GitHub: [@tu-usuario](https://github.com/tu-usuario)
- Email: tu-email@ejemplo.com

---

## 🙏 Agradecimientos

- [MySQL Documentation](https://dev.mysql.com/doc/)
- [XAMPP Community](https://www.apachefriends.org/)
- Estudiantes que probaron este tutorial

---

## 📈 Versiones

- **v1.0.0** - Versión inicial con triggers básicos
- **v1.1.0** - Agregadas validaciones y alertas
- **v1.2.0** - Mejoras en documentación y troubleshooting

---

<div align="center">

**⭐ Si este tutorial te fue útil, no olvides darle una estrella ⭐**

</div>
