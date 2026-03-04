
# Escenario Técnico: CineGest 2.0
Tenemos el siguiente esquema relacional para recoger la información de una cadena de cines.

CINES (id_cine, nombre, ciudad)

PELICULAS (id_pelicula, titulo, director, genero)

PROYECCIONES(id_proyeccion, id_cine, id_pelicula, fecha, hora, recaudacion)

(Nota: id_cine e id pelicula son claves ajenas)

# 1: Consultas Básicas y Composiciones

## 1. Consulta que devuelve el titulo y el género de todas las películas cuyo género sea 'Ciencia Ficción', ordenadas alfabéticamente por titulo.

```sql
SELECT titulo, genero
FROM PELICULAS
WHERE genero = 'Ciencia Ficción'
ORDER BY titulo ASC;
```

## 2. Consulta que muestra el nombre del cine y el título de la película para todas las proyecciones realizadas utilizando la sintaxis de composición interna (JOIN).

```sql
SELECT c.nombre, p.titulo
FROM PROYECCIONES pr
JOIN CINES C ON pr.id_cine = c.id_cine
JOIN PELICULAS P ON pr.id_pelicula = p.id_pelicula;
```

## 3. Consulta de composición externa que muestra el nombre de todos los cines y el id_pelicula de sus proyecciones (incluyendo los nulos).

```sql
SELECT c.nombre, pr.id_pelicula
FROM CINES C
LEFT JOIN PROYECCIONES pr ON c.id_cine = pr.id_cine;
```

# 2: Consultas de Resumen y Agrupación

## 1. Consulta para calcular la recaudación total acumulada por cada cine.

```sql
SELECT c.nombre, SUM(pr.recaudacion) AS recaudacion_total FROM CINES C
JOIN PROYECCIONES pr ON c.id_cine = pr.id_cine
GROUP BY c.nombre;
```


## 2. Consulta para que solo aparezcan los cines cuya recaudación total sea superior a 5.000€. Explición de por qué se usa HAVING en lugar de WHERE.

```sql
SELECT c.nombre, SUM(pr.recaudacion) AS recaudacion_total
FROM CINES c
JOIN PROYECCIONES pr ON c.id_cine = pr.id_cine
GROUP BY c.nombre
HAVING SUM(pr.recaudacion) > 5000;
```

La cláusula WHERE se utiliza para filtrar filas individuales antes de que se agrupen los datos. 

Sin embargo, cuando necesitamos filtrar resultados basados en una función de agregación (como la SUMA total de la recaudación), la agrupación ya debe haberse realizado. 

Para evaluar condiciones sobre esos grupos ya calculados, es obligatorio utilizar la cláusula HAVING, que se ejecuta después del GROUP BY.

# 3: Subconsultas y Herramientas

## 1. Subconsulta para obtener los títulos de las peliculas que han tenido una recaudación en una sesión individual superior a la recaudación media de todas las sesiones registradas en la tabla PROYECCIONES.

```sql
SELECT DISTINCT p.titulo
FROM PELICULAS P
JOIN PROYECCIONES pr ON p.id_pelicula = pr.id_pelicula
WHERE pr.recaudacion > (SELECT AVG(recaudacion) FROM PROYECCIONES);
```

## 2. Si ejecutamos la siguiente consulta en MySQL y en la consola(CLI) observamos que:
```sql
SELECT DISTINCT p.titulo
FROM PELICULAS P
JOIN PROYECCIONES pr ON p.id_pelicula = pr.id_pelicula
WHERE pr.recaudacion > (SELECT AVG(recaudacion) FROM PROYECCIONES);
```


### Comando DML:
El comando del Lenguaje de Manipulación de Datos (DML) que se ejecuta por debajo en ambos casos es SELECT.

### Diferencia en el resultado:
No existe ninguna diferencia en los datos devueltos (los valores reales, el número de filas o de columnas). 

Ambas herramientas se conectan al mismo motor de base de datos y le envían exactamente la misma instrucción. 

La única diferencia es puramente estética: MySQL Workbench te presentará los datos en una interfaz gráfica (una tabla visual con celdas), mientras que la Consola (CLI) dibujará la tabla en formato de texto plano directamente en el terminal.

# 4: Optimización de Consultas

## 1. Indices y Estructura: Sentencia SQL para crear el objeto que acelera las consultas de la tabla PROYECCIONES tuviera millones de registros. Justificion de por qué este objeto evita que el motor tenga que leer la tabla completa.

```sql
CREATE INDEX idx_proyecciones_fecha ON PROYECCIONES(fecha);
```

Un indice funciona exactamente igual que el indice alfabético al final de un libro grueso. 

Sin el índice, si quieres buscar las proyecciones de una fecha concreta, el motor de la base de datos tiene que leer fila por fila desde la primera hasta la última (lo que se conoce como Full Table Scan o escaneo completo de tabla). 

Al crear un indice sobre la columna fecha, el motor crea una estructura de datos interna (normalmente un árbol B) ordenada por esas fechas. 

Cuando haces una consulta, el motor busca en ese árbol y salta directamente a la posición de memoria exacta donde están los datos, reduciendo drásticamente las operaciones de lectura en el disco duro.

## 2. Comparativa de Eficiencia: Dos versiones para obtener el título de las películas proyectadas en el año 2023:

### Versión 1: Una consulta técnicamente correcta pero ineficiente (basada en malas prácticas de selección y filtrado).

```sql
SELECT * FROM PELICULAS P, PROYECCIONES pr
WHERE p.id_pelicula = pr.id_pelicula
AND YEAR(pr.fecha) = 2023;
```

### Versión 2: Una consulta optimizada (que facilite el trabajo al motor de la BD).

```sql
SELECT DISTINCT p.titulo
FROM PELICULAS P
JOIN PROYECCIONES pr ON p.id_pelicula = pr.id_pelicula
WHERE pr.fecha >= '2023-01-01' AND pr.fecha <= '2023-12-31';
```

## Descripción de la importancia de la optimización comparando ambas versiones en términos de tráfico de red, uso de memoria y aprovechamiento de índices.


### Tráfico de red y uso de memoria:
La Versión 1 utiliza SELECT *, lo que obliga a la base de datos a leer, procesar en RAM y enviar por la red todas las columnas de ambas tablas (incluyendo IDs, recaudación, ciudad del cine, etc.), saturando la memoria y el ancho de banda inútilmente. 

La Versión 2 solo solicita la columna necesaria (p.titulo), minimizando el consumo de recursos. 

Además, el uso de DISTINCT evita devolver el mismo título cientos de veces si la película se proyectó muchos días.

### Aprovechamiento de indices:
En la versión 1, aplicar una función a la columna en el filtrado (YEAR(pr.fecha)) es una pésima práctica porque "ciega" a la base de datos. 

Impide que el motor utilice el indice que creamos en el paso anterior, forzándolo a calcular el año registro por registro (escaneo completo). 

La Versión 2, al usar un rango puro de fechas (>= y <=), permite que el motor aproveche el indice al 100% para acotar la búsqueda de forma inmediata.

### HECHO POR JOSE AGUILERA ORTIZ