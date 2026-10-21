# Hello World: AWS Lambda & API Gateway en Java

Taller de introducción a microservicios con Java sobre **AWS Lambda** y **Amazon API Gateway**. Se implementa un servicio REST que recibe un entero y retorna su cuadrado.

**Autor:** Carlos Avellaneda
**Región AWS:** us-east-1 (Norte de Virginia)
**URL del servicio:** `https://yjqqj7jlkc.execute-api.us-east-1.amazonaws.com/Beta?value=14`

---

## Arquitectura

```
Cliente (navegador)
   │  GET /Beta?value=14
   ▼
API Gateway (mathServices, etapa Beta)
   │  Mapping template: $input.params("value")  →  14
   ▼
AWS Lambda (square, Java 21)
   │  MathServices.square(14)
   ▼
Respuesta: 196
```

## Estructura del proyecto

```
taller-lambda/
├── pom.xml
├── src/main/java/co/edu/escuelaing/services/MathServices.java
├── img/          ← evidencias del taller
└── README.md
```

## 1. Clase que implementa el servicio

```java
package co.edu.escuelaing.services;

public class MathServices {

    public static Integer square(Integer i) {
        return i * i;
    }
}
```

Se compila y empaqueta con Maven:

```bash
mvn package
```

Esto genera `target/square-lambda-1.0.jar`, que es el archivo que se carga en Lambda.

> **Nota:** el tutorial original usa Java 8. En este taller se usó el runtime **Java 21**, porque Java 8 es un runtime antiguo. El proyecto se compila para Java 17, que es compatible con Java 21.

---

## 2. Creación de la función Lambda

Se accede al servicio Lambda. En la consola actual las funciones se llaman **Event Functions**.

![Consola de Lambda](img/01-lambda-consola.png)

Se crea la función con la opción **Crear desde cero**.

![Crear función](img/02-crear-funcion.png)

Configuración usada:
- **Nombre:** `square`
- **Runtime:** Java 21
- **Rol de ejecución:** rol existente `LabRole` (AWS Academy)

![Configuración de la función](img/03-crear-funcion-java21-labrole.png)

Función creada correctamente:

![Función creada](img/04-funcion-creada.png)

### Carga del código

Se carga el jar desde **Actualizar → Archivo .zip o .jar**.

![Jar cargado](img/05-jar-cargado.png)

En **Configuración de la versión ejecutable** se define el **Handler** con el formato `paquete.Clase::método`:

```
co.edu.escuelaing.services.MathServices::square
```

![Handler configurado](img/06-handler.png)

### Prueba de la función

Se crea el evento de prueba `testSquare`. Primero se dejó el JSON que AWS trae por defecto:

![Evento por defecto](img/07-evento-por-defecto.png)

Esto produce un error, porque el método espera un `Integer` y Lambda recibe un objeto JSON que no puede convertir (`Cannot deserialize value of type java.lang.Integer from Object value`):

![Error de deserialización](img/08-error-json.png)

Al reemplazar el evento por un único valor, `5`, la función responde correctamente **25**:

![Prueba exitosa: 25](img/09-test-lambda-25.png)

**Conclusión:** la función recibe **un solo valor, no un JSON**. Esto determina cómo se configura el mapeo en API Gateway.

---

## 3. Configuración de API Gateway

Se crea una **API REST nueva** llamada `mathServices`, con punto de enlace **Regional**.

![Crear API](img/10-crear-api.png)

### Método GET

Se crea un método **GET** sobre el recurso `/`:
- **Tipo de integración:** Función Lambda (`square`)
- **Integración de proxy de Lambda:** desactivada
- **Parámetro de cadena de consulta:** `value`

![Crear método GET](img/11-crear-metodo-get.png)

![Método GET creado](img/12-metodo-get-creado.png)

### Solicitud de integración

En **Solicitud de integración** se mapea el parámetro `value` desde `method.request.querystring.value`. Se agrega además una **plantilla de mapeo** para `application/json`, que inicialmente es una constante (`7`):

![Integración con constante](img/13-integracion-constante.png)

![Plantilla con constante](img/14-plantilla-constante.png)

Al probar, el resultado siempre es **49**, porque se envía una constante:

![Prueba con constante: 49](img/15-test-constante-49.png)

### Plantilla con parámetro

Se cambia la plantilla para que tome el valor del parámetro de la URL. La variable `$input` representa la entrada de la solicitud:

```
$input.params("value")
```

![Plantilla con parámetro](img/16-plantilla-parametro.png)

Prueba con `value=14` → **196**:

![Prueba con value=14](img/17-test-value-14.png)

Prueba **sin** el parámetro: se reporta un error, como se esperaba, porque la función recibe un cuerpo vacío.

![Prueba sin value](img/18-test-sin-value.png)

---

## 4. Despliegue

Se despliega la API desde **Implementar API** en una nueva etapa llamada `Beta`.

![Deploy Beta](img/19-deploy-beta.png)

La etapa queda activa con su **URL de invocación**:

![Etapa Beta](img/20-etapa-beta-url.png)

Al invocar el servicio desde el navegador:

```
https://yjqqj7jlkc.execute-api.us-east-1.amazonaws.com/Beta?value=14
```

se obtiene **196**:

![Servicio desde el navegador](img/21-navegador-196.png)

---

## Conclusiones

- AWS Lambda permite exponer un método Java como servicio sin administrar servidores. Basta con subir el jar y definir el handler `Clase::método`.
- Lambda deserializa automáticamente la entrada al tipo del parámetro del método. Por eso el evento debe ser un número (`5`) y no un objeto JSON.
- Las **plantillas de mapeo** de API Gateway transforman la solicitud HTTP en la entrada que espera la función. En este caso `$input.params("value")` extrae el parámetro de la URL.
- La **integración de proxy de Lambda** debe estar desactivada para usar plantillas de mapeo. Si está activa, Lambda recibe el evento HTTP completo como JSON.
- En casos reales se usan POJOs o los tipos de `aws-lambda-java-events`, que Lambda serializa y deserializa automáticamente desde y hacia JSON.


