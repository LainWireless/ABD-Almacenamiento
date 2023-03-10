# Alumno 3 (Alfonso Roldán Amador):
## ORACLE:

### 1. Muestra los objetos a los que pertenecen las extensiones del tablespace TS2 (creado por Alumno 2) y el tamaño de cada una de ellas.

Replicaremos el proceso de creación del tablespace TS2 tal y como hizo el alumno 2 (2 ficheros en rutas diferentes de 1M cada uno no autoextensibles). Posteriormente insertaremos los registros hasta llenar el tablespace.

Con la siguiente consulta obtendremos los segmentos, id de extensión y tamaño del tablespace TS2:

```sql
SELECT SEGMENT_NAME, EXTENT_ID, BYTES FROM DBA_EXTENTS WHERE TABLESPACE_NAME = 'TS2';
```

![Ejercicio1_Consulta](capturas/1.1.consulta.png)



### 2. Borra la tabla que está llenando TS2 consiguiendo que vuelvan a existir extensiones libres. Añade después otro fichero de datos a TS2.

Para borrar la tabla que está llenando TS2 podemos usar **"Truncate table"**. Con esto se eliminará la tabla y además se liberará espacio en el disco:

```sql
TRUNCATE TABLE nombre_tabla;
```

![Ejercicio2_TruncateTable](capturas/2.1.tabla_truncada.png)

Posteriormente añadimos otro fichero de datos a TS2 mediante la siguiente sentencia (En este con un tamaño de 1M):

```sql
ALTER TABLESPACE TS2 ADD DATAFILE '/opt/oracle/oradata/ORCLCDB/extra.dbf' SIZE 1M;
```

![Ejercicio2_TruncateTable](capturas/2.2.add_datafile.png)


### 3. Crea el tablespace TS3 gestionado localmente con un tamaño de extension uniforme de 128K y un fichero de datos asociado. Cambia la ubicación del fichero de datos y modifica la base de datos para que pueda acceder al mismo. Crea en TS3 dos tablas e inserta registros en las mismas. Comprueba que segmentos tiene TS3, qué extensiones tiene cada uno de ellos y en qué ficheros se encuentran.

**Creamos el tablespace TS3**, en este caso de 100M, con un tamaño de extensión uniforme de gestión local establecido en 138 KB.

```sql
CREATE TABLESPACE TS3 DATAFILE '/opt/oracle/oradata/ORCLCDB/ts3.dbf' SIZE 100M EXTENT MANAGEMENT LOCAL UNIFORM SIZE 128K;
```

![Ejercicio3_CrearTS3](capturas/3.1.crear_ts3.png)


**Cambiar la ubicación del fichero de datos**

Antes de cambiar de ubicación el fichero de datos debemos marcar el tablespace como inactivo (OFFLINE), mediante la siguiente sentencia:

```sql
ALTER TABLESPACE TS3 OFFLINE;
```

Posteriormente podemos mostrar los tablespaces junto con su estádo con:

```sql
SELECT TABLESPACE_NAME, STATUS FROM DBA_TABLESPACES;
```

![Ejercicio3_StatusTablespaces](capturas/3.3.status_tablespaces.png)



O podemos filtrar únicamente el estado especificando el tablespace:

```sql
SELECT STATUS FROM DBA_TABLESPACES WHERE TABLESPACE_NAME='TS3';
```

En este caso crearé un directorio (**ts3**) dentro de la ruta donde se encuentra el fichero de datos, y moveré el fichero ts3.dbf al nuevo directorio.


```shell
cd /opt/oracle/oradata/ORCLCDB && mkdir ts3
mv ts3.dbf ts3
```

![Ejercicio3_cambiar_ubicacion_fichero_de_datos](capturas/3.2.cambiar_ubicacion_fichero_de_datos.png)

Ahora vamos a **modificar la base de datos** para que pueda acceder al fichero de datos (ts3.dbf).

Una vez tenemos el tablespace en modo **"offline"** y hemos cambiado la ubicación del fichero de datos, modificamos la entrada del fichero de datos en el control file: 

```sql
ALTER DATABASE RENAME FILE '/opt/oracle/oradata/ORCLCDB/ts3.dbf' TO '/opt/oracle/oradata/ORCLCDB/ts3/ts3.dbf';
```

![Ejercicio3_base_de_datos_modificada](capturas/3.4.base_de_datos_modificada.png)


Finalmente pasamos el tablespace a estado online:

```sql
ALTER TABLESPACE TS3 ONLINE;
```

![Ejercicio3_tablespace_online](capturas/3.5.tablespace_online.png)


A continuación vamos a **crear en el tablespace 2 tablas e insertar registros en las mismas**.

**Tabla1**

```sql
CREATE TABLE tabla1 (
    id number, 
    nombre varchar2(30), 
    apellido VARCHAR2(30)
) 
TABLESPACE TS3;
```

Insercción de datos:

```sql
INSERT INTO tabla1 VALUES (1,'Alfonso','Roldan');
INSERT INTO tabla1 VALUES (2,'Pedro','Rodriguez');
INSERT INTO tabla1 VALUES (3,'Marta','Pozo');
INSERT INTO tabla1 VALUES (4,'Alba','Garcia');
```

![Ejercicio3_tabla1](capturas/3.6.tabla1.png)


**Tabla2**

```sql
CREATE TABLE tabla2 (
    id number, 
    nombre varchar2(30), 
    apellido VARCHAR2(30)
) 
TABLESPACE TS3;
```

Insercción de datos:

```sql
INSERT INTO tabla2 VALUES (1,'Lucas','Perez');
INSERT INTO tabla2 VALUES (2,'Marcos','Molina');
INSERT INTO tabla2 VALUES (3,'Maria','Romero');
INSERT INTO tabla2 VALUES (4,'Jaime','Ortega');
```

![Ejercicio3_tabla2](capturas/3.7.tabla2.png)


**Comprobamos que segmentos tiene TS3, que extensiones tienen cada uno, y el identificador del fichero al que pertenece:**

```sql
select segment_name from dba_extents where tablespace_name='TS3';
```

![Ejercicio3_segmentos_ts3](capturas/3.8.segmentos_ts3-extensiones-ficheros.png)


Si queremos saber el fichero al que pertenece el identificador que hemos obtenido, usamos la siguiente consulta:

```sql
select file_name from dba_data_files where file_id = 16;
```

![Ejercicio3_fichero](capturas/3.9.fichero.png)


Como vemos el **extent_id** tiene el valor 0. En Oracle, un **extent_id** con **valor 0** significa que se trata de un sistema extent y que está siendo utilizado para almacenar objetos internos de la base de datos.

### 4. Redimensiona los ficheros asociados a los tres tablespaces que has creado de forma que ocupen el mínimo espacio posible para alojar sus objetos.

Antes que nada comprobaremos si los tablespace son **SYSTEM** o **NON-SYSTEM**, consultando la tabla **DBA_TABLESPACES**, concretamente la columna **CONTENTS**. Lo haremos mediante la siguiente consulta:

```sql
SELECT TABLESPACE_NAME, CONTENTS FROM DBA_TABLESPACES;
```

![Ejercicio4_fichero](capturas/4.1.contents.png)


Si la columna **CONTENTS** contiene el valor **PERMANENT**, entonces el tablespace es **SYSTEM**. Si contiene el valor **TEMPORARY**, entonces es un tablespace **NON-SYSTEM**.


Mediante el comando **SHRINK SPACE** podemos reducir el tamaño de un archivo de un tablespace, liberando el espacio no utilizado y haciendo que el archivo ocupe el mínimo espacio posible para alojar sus objetos.

```sql
ALTER TABLESPACE nombre_tablespace SHRINK SPACE;
```

Debemos de tener en cuenta que el comando **SHRINK SPACE** solo puede ejecutarse en un tablespace **NON-SYSTEM**, por lo que en este caso no podemos reducir el espacio para que ocupe el mínimo posible.

Sin embargo, podemos reducir el tablespace especificando un valor manualmente (En este caso el mínimo es 2MB) con la siguiente sentencia:

```sql
ALTER DATABASE DATAFILE '/opt/oracle/oradata/ORCLCDB/ts3/ts3.dbf' RESIZE 2M
```

![Ejercicio4_resize](capturas/4.2.resize.png)


### 5. Realiza un procedimiento llamado InformeRestricciones que reciba el nombre de una tabla y muestre los nombres de las restricciones que tiene, a qué columna o columnas afectan y en qué consisten exactamente.

**Procedimiento Principal**

```sql
CREATE OR REPLACE PROCEDURE InformeRestricciones(p_tabla DBA_TABLES.TABLE_NAME%TYPE)
IS
	cursor c_constraints is
	select distinct CONSTRAINT_NAME from dba_constraints where table_name=p_tabla;
BEGIN
	for v_constraint in c_constraints loop
	dbms_output.put_line(chr(10)||'Restriccion: '||v_constraint.CONSTRAINT_NAME||chr(10)||'Columnas afectadas:');
	Comprobar_columnas_afectadas(v_constraint.CONSTRAINT_NAME);
	end loop;
END;
/
```


**Procedimientos dependientes** (Compilar antes del procedimiento principal, de abajo a arriba).


```sql
CREATE OR REPLACE PROCEDURE Comprobar_columnas_afectadas(p_constraint
DBA_CONS_COLUMNS.CONSTRAINT_NAME%TYPE)
IS
    cursor c_columnas is
    select distinct COLUMN_NAME, TABLE_NAME from DBA_CONS_COLUMNS where CONSTRAINT_NAME=p_constraint;
BEGIN
    for v_columna in c_columnas loop
        dbms_output.put_line(chr(9)||'Columna: '||v_columna.COLUMN_NAME);
        Mostrar_en_que_consiste_restriccion(p_constraint, v_columna.TABLE_NAME);
    end loop;
END;
/
```

```sql
CREATE OR REPLACE PROCEDURE Mostrar_en_que_consiste_restriccion(p_constraint
DBA_CONSTRAINTS.CONSTRAINT_NAME%TYPE, p_tabla DBA_CONSTRAINTS.TABLE_NAME%TYPE)
IS
    v_consiste VARCHAR2(4000);
BEGIN
    select distinct search_condition_vc into v_consiste from DBA_CONSTRAINTS where CONSTRAINT_NAME=p_constraint and TABLE_NAME=p_tabla;
    if v_consiste is null or length(v_consiste) = 0 then
        dbms_output.put_line(chr(9)||'Consiste en: No existe descripcion.');
    else
        dbms_output.put_line(chr(9)||'Consiste en: '||v_consiste);
    end if;
END;
/
```

**Nota:** Para la definición de la restricción, usaremos la obtendremos datos de la columna
**search_condition_vc** que a diferencia de **search_condition**, esta es **VARCHAR2**.

#### ***Compilación de procedimientos***

![Ejercicio5_CompilacionProcedimientos](capturas/5.1.compilacion.png)

#### ***Comprobación del funcionamiento***

![Ejercicio5_Comprobacion](capturas/5.2.comprobacion.png)


### 6. Realiza un procedimiento llamado MostrarAlmacenamientoUsuario que reciba el nombre de un usuario y devuelva el espacio que ocupan sus objetos agrupando por dispositivos y archivos:

```sql
Usuario: NombreUsuario
Dispositivo:xxxx
Archivo: xxxxxxx.xxx
Tabla1......nnn
…
TablaN......nnn
Indice1.....nnn
…
IndiceN.....nnn
K
K
K
K
Total Espacio en Archivo xxxxxxx.xxx: nnnnn K
Archivo:...
…
Total Espacio en Dispositivo xxxx: nnnnnn K
Dispositivo: yyyy
…
Total Espacio Usuario en la BD: nnnnnnn K
```

**Procedimiento Principal**
```sql
CREATE OR REPLACE PROCEDURE MostrarAlmacenamientoUsuario(p_usuario VARCHAR2)
IS
    cursor c_objetos is
    select distinct REGEXP_SUBSTR(f.file_name, '^/[^/]+') as dispositivo, f.file_name as ruta_archivo,REGEXP_SUBSTR(f.file_name, '[^/]+$') as archivo, f.file_id as id_fichero from dba_extents e, dba_data_files f where f.file_id=e.file_id and owner=upper(p_usuario);
BEGIN
    dbms_output.put_line('Usuario: '||p_usuario);
    for objeto in c_objetos loop
    dbms_output.put_line(chr(10)||chr(9)||'Dispositivo: '||objeto.dispositivo||chr(10)||chr(9)||chr(9)||'Archivo: '||objeto.ruta_archivo||chr(10));
    Mostrar_tablas_archivo_por_usuario(p_usuario,objeto.id_fichero);
    Mostrar_indices_archivo_por_usuario(p_usuario,objeto.id_fichero);
    dbms_output.put_line(chr(10)||chr(9)||chr(9)||chr(9)||'Total Espacio en Archivo '||objeto.archivo||': '||Devolver_espacio_fichero(p_usuario, objeto.id_fichero)||' K');
    dbms_output.put_line(chr(9)||chr(9)||chr(9)||'Total Espacio en Dispositivo '||objeto.dispositivo||': '||Devolver_espacio_dispositivo(p_usuario, objeto.id_fichero)||' K');
    end loop;
    dbms_output.put_line(chr(10)||'Total Espacio Usuario en la BD: '||Devolver_espacio_usuario(p_usuario)||' K');
END;
/


CREATE OR REPLACE PROCEDURE Mostrar_tablas_archivo_por_usuario(p_usuario VARCHAR2, p_id_fichero NUMBER)
is
    cursor c_tablas is
    select distinct t.table_name as tab, e.bytes/1024 as kilobytes
    from dba_tables t, dba_extents e 
    where t.owner=upper(p_usuario) and t.owner=e.owner and e.file_id=p_id_fichero;
    v_cont number := 1;
    v_punto varchar(1) := '.';
begin
    for tabla in c_tablas loop
        dbms_output.put_line(chr(9)||chr(9)||chr(9)||'Tabla'||v_cont||RPAD(v_punto,10,'.')||tabla.tab||' '||tabla.kilobytes);
        v_cont:=v_cont+1;
    end loop;
    dbms_output.put_line(chr(9));
end;
/

CREATE OR REPLACE PROCEDURE Mostrar_indices_archivo_por_usuario(p_usuario VARCHAR2, p_id_fichero NUMBER)
is
    cursor c_indices is
    select distinct i.index_name as ind, e.bytes/1024 as kilobytes
    from dba_indexes i, dba_extents e 
    where i.owner=upper(p_usuario) and i.owner=e.owner and e.file_id=p_id_fichero;
    v_cont number := 1;
    v_punto varchar(1) := '.';
begin
    for indice in c_indices loop
        dbms_output.put_line(chr(9)||chr(9)||chr(9)||'Indice'||v_cont||RPAD(v_punto,10,'.')||indice.ind||' '||indice.kilobytes);
        v_cont:=v_cont+1;
    end loop;
    dbms_output.put_line(chr(9));
end;
/

CREATE OR REPLACE FUNCTION Devolver_espacio_fichero(p_usuario VARCHAR2, p_id_fichero NUMBER)
return number
is
    v_espacio number := 0;
begin
    select distinct d.user_bytes/1024 into v_espacio
    from dba_data_files d, dba_extents e 
    where e.owner=upper(p_usuario) and d.file_id=e.file_id and d.file_id=p_id_fichero;
return v_espacio;
end;
/

CREATE OR REPLACE FUNCTION Devolver_espacio_usuario(p_usuario VARCHAR2)
return number
is
    v_espacio number := 0;
begin
    select SUM(bytes)/1024 into v_espacio
    from dba_segments
    where owner = upper(p_usuario);
return v_espacio;
end;
/

-- Suma de espacio de cada bloque

CREATE OR REPLACE FUNCTION Devolver_espacio_dispositivo(p_usuario VARCHAR2, p_id_fichero NUMBER)
return number
is
    v_espacio number := 0;
begin
    select sum(d.user_bytes/1024) into v_espacio
    from dba_data_files d, dba_extents e 
    where e.owner=upper(p_usuario) and d.file_id=e.file_id and d.file_id=p_id_fichero;
return v_espacio;
end;
/
```

### Compilación del procedimiento principal

![Ejercicio6_Comprobacion](capturas/6.1.Compilacion_del_procedimiento_principal.png)

### Prueba de funcionamiento

![Ejercicio6_Comprobacion](capturas/6.2.Ejecucion_del_procedimiento.png)

## Postgres:

### 7. Averigua si es posible establecer cuotas de uso sobre los tablespaces en Postgres.

Tras investigar bastante, he llegado a la conclusión de que sí podemos establecer cuotas de uso sobre los tablespaces en Postgres, aunque esta característica no está incorporada directamente en la base de datos. Por lo que tendremos que recurrir a otros métodos para limitar la cantidad de espacio que consumirán un usuario o un conjunto de objetos en el sistema de archivos.

### **Desde el intérprete del sistema (Bash)**

Podemos optar por el método del uso de **quotas** del sistema. Para ello seguiremos los siguientes pasos:

Instalamos el paquete quota:

```shell
sudo apt install quota
```

Editamos el fichero **/etc/fstab** y añadimos los siguiente parámetros, en la línea donde se encuentra la partición o dispositivo al que vamos a aplicar la quota.

![Ejercicio7_fstab](capturas/7.1.fstab.png)

Montamos de nuevo la partición correspondiente:

```shell
sudo mount -o remount /
```

Verificamos que se hayan aplicado las nuevas opciones a la hora de montar el sistema de archivos, en este caso, mediante el comando:

```shell
cat /proc/mounts | grep ' / '
```

![Ejercicio7_VerificarMontaje](capturas/7.2.verificar_montaje.png)

Habilitamos las quotas:

```shell
sudo quotacheck -ugm /
```

Activamos el sistema de quotas:

```shell
sudo quotaon -v /
```

![Ejercicio7_ActivarSistemaDeQuotas](capturas/7.3.activar_sistema_de_quotas.png)

Ahora podremos modificar las cuotas de los usuarios mediante el comando:

```shell
sudo edquota -u usuario
```

![Ejercicio7_quota_postgres](capturas/7.4.quota_postgres.png)

Con esto le habríamos asignado una cuota máxima de tamaño a un usuario específico en la partición donde se aloja el tablespace.

### **Desde el intérprete de Postgres**

Desde el intérprete de Postgres, también contamos con algunos métodos para gestionar el almacenamiento del usuario, por ejemplo:

Creamos un nuevo tablespace llamado **"tablespace_limitado"**, en una ubicación donde tengamos un tamaño máximo definido en el sistema (Como puede ser un volumen lógico):

```sql
CREATE TABLESPACE tablespace_limitado LOCATION '/path/to/tablespace_limitado';
```

Creamos un nuevo esquema llamado **"schema_limitado"** dentro del tablespace **"tablespace_limitado"**:

```sql
CREATE SCHEMA schema_limitado IN tablespace_limitado;
```

Asignamos permisos de uso al usuario **"usuario"** sobre el esquema "schema_limitado"

```sql
GRANT USAGE ON SCHEMA schema_limitado TO usuario;
```


## MySQL:

### 8. Averigua si existe el concepto de extensión en MySQL y si coincide con el existente en ORACLE.

Sí, existe el concepto de extent en MySQL, y además coincide con el existente en Oracle.

La principal diferencia es que en MySQL su nomenclatura define que un **extent**, es una agrupación de **pages** a diferencia de los **data blocks** en Oracle, que son un número específico de bloques contiguos usados para almacenar un tipo específico de información.

**En MySQL:**

Un archivo de datos **InnoDB** o archivo ibd es una secuencia de páginas del mismo tamaño. Estas páginas se agrupan en extensiones y segmentos. 

Una extensión o extent es una **secuencia contigua de páginas** (porción contigua de espacio en el disco) en un archivo ibd. El número de páginas que pertenecen a una extensión dependerá del tamaño de la página.

La siguiente tabla proporciona la relación entre el tamaño de página y el tamaño de extensión (en páginas y megabytes):

| **Tamaño de página** | **Tamaño de extensión en páginas** | **Tamaño de extensión en MB** |
|----------------------|------------------------------------|-------------------------------|
|          4 KB        |                 256                |              1 MB             |
|          8 KB        |                 128                |              1 MB             |
|         16 KB        |                 64                 |              1 MB             |
|         32 KB        |                 64                 |              2 MB             |
|         64 KB        |                 64                 |              4 MB             |

Los extents se utilizan en los motores de almacenamiento que implementan el sistema de gestión de tablas InnoDB, y su objetivo es mejorar la eficiencia y la gestión del espacio en disco.

Cada extent contiene un número fijo de páginas de disco, y los datos de una tabla se dividen en varios extents para aumentar la eficiencia y mejorar el rendimiento. Por ejemplo, cuando una tabla crece y necesita más espacio, InnoDB puede asignar un nuevo extent para almacenar los datos adicionales.

**En Oracle:**

Un extent en Oracle es un bloque de almacenamiento contiguo en el disco que se utiliza para almacenar datos de una tabla o índice. Cada tabla o índice en Oracle se divide en una serie de extents, y cada extent se compone de un número específico de bloques de datos.

El tamaño de un extent es determinado por el parámetro de inicialización del tamaño del bloque del sistema y por el número de bloques que componen el extent. Por ejemplo, si el tamaño de un bloque de datos es de 8 kilobytes y un extent está compuesto por 4 bloques, el tamaño total del extent será de 32 kilobytes.

El uso de extents permite a Oracle administrar de manera eficiente el espacio en disco, permitiendo el crecimiento dinámico de las tablas y índices en pequeños bloques contiguos en lugar de una sola gran extensión. Además, los extents permiten una mayor concurrencia de lectura y escritura, ya que cada extent es una unidad de almacenamiento independiente.

## MongoDB:

### 9. Averigua si en MongoDB puede saberse el espacio disponible para almacenar nuevos documentos.

Sí, en MongoDB podemos conocer el espacio disponible para almacenar nuevos documentos.

#### Comprobar el espacio disponible en la base de datos:

Con el comando **"db.stats()"** podremos ver información detallada sobre el tamaño de la base de datos (actual/en uso), incluyendo el tamaño en disco, el número de documentos, la cantidad de espacio utilizado y la cantidad de espacio disponible.

También podemos ejecutarlo de la siguiente forma:

```mongodb
db.runCommand( { dbStats: 1, scale: 1024, freeStorage: 1 } )
```

Parámetros aplicados:

**dbStats:** El comando devuelve estadísticas de almacenamiento para una base de datos dada.

**Scale:** (Opcional). El factor de escala para los distintos datos de tamaño. El valor "scale" predeterminado es 1 para devolver datos de tamaño en bytes. Para mostrar kilobytes en lugar de bytes, especificamos un scale valor de 1024.

**freeStorage** (Opcional). Para devolver detalles sobre el espacio libre asignado a las colecciones, establezca freeStorageen 1.


![Ejercicio8_fs_size](capturas/8.1.fs_size.png)


#### Comprobar el espacio disponible en una colección específica:

Con el siguiente comando obtendremos información detallada sobre la colección (En este caso **"profesores"**), incluyendo el tamaño de la colección, el número de documentos, el espacio utilizado y el espacio disponible.

```mongodb
db.profesores.stats()
```

o

```mongodb
db.runCommand({"collStats": "profesores"})
```


También podemos utilizar el siguiente comando para filtrar y obtener solo el freeStorageSize de una colección:

```mongodb
db.runCommand({collStats: "profesores", scale: 1024}).freeStorageSize
```

Al utilizar el parámetro (**"scale: 1024"**), especificamos que la información se devuelva en KiloBytes.

![Ejercicio8_collection_size](capturas/8.2.collection_size.png)


En resumen, este comando obtiene el espacio de almacenamiento disponible en kilobytes de la colección **"profesores"** en MongoDB.