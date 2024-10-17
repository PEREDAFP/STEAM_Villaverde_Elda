# STEAM_Villaverde_Elda

OBJETIVOS: Creación de una aplicación Web para poner en contacto emrpesas lideradas por mujeres y estudiantes de FP y Universidad interesadas en trabajar en STEAM

El proyecto se desarrollará con Vue3 y se utilizará, en un primer momento, la base de datos SUPABASE

## Tablas de la base de datos
Estructura básica para este sistema.

### Tablas de la base de datos

1. **Empresas**: Aquí se almacenarán los detalles de las empresas que publican las ofertas.
2. **Ofertas de Empleo**: Aquí estarán las ofertas que las empresas crean.
3. **Alumnas**: Contendrá la información de las alumnas que buscan trabajo.
4. **Preferencias de Alumnas**: Las preferencias de búsqueda de trabajo de las alumnas, para el filtrado automático.
5. **Categorías STEAM**: Para definir qué sector STEAM pertenece una oferta.
6. **Postulaciones**: Registro de las postulaciones de las alumnas a las ofertas.
7. **Ubicaciones**: Detalles de las ubicaciones de las empresas y ofertas.

### Diagrama Estructural

#### 1. Tabla `empresas`
```sql
CREATE TABLE empresas (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL,
    descripcion TEXT,
    email_contacto VARCHAR(255),
    telefono_contacto VARCHAR(20),
    sector VARCHAR(255),  -- STEAM o área en la que trabaja la empresa
    ubicacion_id INT REFERENCES ubicaciones(id)
);
```

#### 2. Tabla `ofertas_empleo`
```sql
CREATE TABLE ofertas_empleo (
    id SERIAL PRIMARY KEY,
    titulo VARCHAR(255) NOT NULL,
    descripcion TEXT NOT NULL,
    fecha_publicacion DATE DEFAULT CURRENT_DATE,
    empresa_id INT REFERENCES empresas(id),
    ubicacion_id INT REFERENCES ubicaciones(id),
    categoria_id INT REFERENCES categorias_steam(id),
    tipo_contrato VARCHAR(50),  -- Ejemplo: 'Tiempo Completo', 'Prácticas'
    remuneracion DECIMAL(10, 2),
    requisitos TEXT
);
```

#### 3. Tabla `alumnas`
```sql
CREATE TABLE alumnas (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    telefono VARCHAR(20),
    universidad_centro_fp VARCHAR(255),  -- FP o universidad
    carrera_estudios VARCHAR(255),  -- Nombre de la carrera o FP
    descripcion_perfil TEXT
);
```

#### 4. Tabla `preferencias_alumnas`
```sql
CREATE TABLE preferencias_alumnas (
    id SERIAL PRIMARY KEY,
    alumna_id INT REFERENCES alumnas(id),
    categoria_id INT REFERENCES categorias_steam(id),
    tipo_contrato VARCHAR(50),  -- Preferencia de contrato (tiempo completo, prácticas)
    ubicacion_id INT REFERENCES ubicaciones(id)
);
```

#### 5. Tabla `categorias_steam`
```sql
CREATE TABLE categorias_steam (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL  -- Ciencia, Tecnología, Ingeniería, Artes, Matemáticas, etc.
);
```

#### 6. Tabla `postulaciones`
```sql
CREATE TABLE postulaciones (
    id SERIAL PRIMARY KEY,
    alumna_id INT REFERENCES alumnas(id),
    oferta_id INT REFERENCES ofertas_empleo(id),
    fecha_postulacion DATE DEFAULT CURRENT_DATE,
    estado VARCHAR(50)  -- Ej: 'Pendiente', 'Aceptada', 'Rechazada'
);
```

#### 7. Tabla `ubicaciones`
```sql
CREATE TABLE ubicaciones (
    id SERIAL PRIMARY KEY,
    ciudad VARCHAR(255),
    pais VARCHAR(255)
);
```

### Funcionalidad de Filtrado Automático
Para implementar el filtrado de ofertas basado en las preferencias de las alumnas, puedes realizar una consulta SQL que compare las ofertas de empleo con las preferencias de cada alumna.

Ejemplo de consulta SQL para encontrar ofertas que coincidan con las preferencias de una alumna:

```sql
SELECT oe.*
FROM ofertas_empleo oe
JOIN preferencias_alumnas pa ON oe.categoria_id = pa.categoria_id
WHERE pa.alumna_id = 1  -- ID de la alumna actual
  AND (pa.tipo_contrato IS NULL OR oe.tipo_contrato = pa.tipo_contrato)
  AND (pa.ubicacion_id IS NULL OR oe.ubicacion_id = pa.ubicacion_id);
```

### 1. Tabla `notificaciones`

Esta tabla almacenará las notificaciones que se van generando para las alumnas sobre nuevas ofertas.

```sql
CREATE TABLE notificaciones (
    id SERIAL PRIMARY KEY,
    alumna_id INT REFERENCES alumnas(id),
    oferta_id INT REFERENCES ofertas_empleo(id),
    fecha_notificacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    enviada BOOLEAN DEFAULT FALSE  -- Para controlar si ya se ha enviado la notificación
);
```

### 2. Generar notificaciones

#### 1. Función para generar notificaciones con un trigger

Creación de  **trigger** en la tabla de `ofertas_empleo` para que se genere una notificación cuando se inserta una nueva oferta:

```sql
-- Trigger para generar notificaciones cuando se inserta una nueva oferta
CREATE OR REPLACE FUNCTION generar_notificaciones_trigger() RETURNS trigger AS $$
BEGIN
    INSERT INTO notificaciones (alumna_id, oferta_id)
    SELECT pa.alumna_id, NEW.id
    FROM preferencias_alumnas pa
    WHERE pa.categoria_id = NEW.categoria_id
      AND (pa.tipo_contrato IS NULL OR pa.tipo_contrato = NEW.tipo_contrato)
      AND (pa.ubicacion_id IS NULL OR pa.ubicacion_id = NEW.ubicacion_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Crear el trigger para ejecutarse después de la inserción de una nueva oferta
CREATE TRIGGER generar_notificaciones_after_insert
AFTER INSERT ON ofertas_empleo
FOR EACH ROW
EXECUTE FUNCTION generar_notificaciones_trigger();
```


### 4. Lógica para el envío de notificaciones
Se  podría implementar un envío de las notificaciones por correo electrónico/SMS/whatsapp/Tlegram mediante un script que consuma los datos de la tabla `notificaciones` y envíe el mensaje correspondiente. Un ejemplo en python sería:

```python
import smtplib
from email.mime.text import MIMEText
from psycopg2 import connect

def enviar_notificaciones():
    # Conexión a la base de datos
    conn = connect("dbname=mi_bd user=mi_usuario password=mi_password")
    cur = conn.cursor()

    # Obtener notificaciones pendientes de enviar
    cur.execute("""
        SELECT n.id, a.email, o.titulo, o.descripcion
        FROM notificaciones n
        JOIN alumnas a ON n.alumna_id = a.id
        JOIN ofertas_empleo o ON n.oferta_id = o.id
        WHERE n.enviada = FALSE;
    """)
    
    notificaciones = cur.fetchall()

    for notif in notificaciones:
        id_notif, email_alumna, titulo_oferta, descripcion_oferta = notif
        # Crear el mensaje de correo
        msg = MIMEText(f"Se ha publicado una nueva oferta que puede interesarte: {titulo_oferta}\n\n{descripcion_oferta}")
        msg['Subject'] = 'Nueva oferta de trabajo disponible'
        msg['From'] = 'noreply@miapp.com'
        msg['To'] = email_alumna

        # Enviar el correo electrónico
        with smtplib.SMTP('smtp.miapp.com') as server:
            server.login("usuario", "password")
            server.sendmail(msg['From'], [msg['To']], msg.as_string())

        # Marcar la notificación como enviada
        cur.execute("UPDATE notificaciones SET enviada = TRUE WHERE id = %s", (id_notif,))
    
    conn.commit()
    cur.close()
    conn.close()
```

### 5. Integración

La lógica de notificaciones podría integrarse en el backend de la aplicación web, donde una rutina diaria verifica la tabla de `notificaciones` para enviar mensajes a las alumnas.
