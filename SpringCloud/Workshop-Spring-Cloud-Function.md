### Spring Cloud Function

## Conceptos Clave

1. Programación funcional en Java

    - Function<T, R> → recibe un input y devuelve un output.

    - Consumer<T> → recibe un input pero no devuelve nada.

    - Supplier<T> → no recibe input pero devuelve algo.

    Spring Cloud Function trabaja con estas interfaces estándar de Java.

2. Adaptadores de entrada/salida
    
    Spring abstrae cómo llegan los datos a tu función:

    - HTTP Request/Response

    - Mensajes de Kafka o RabbitMQ

    - Eventos de AWS Lambda

3. Desacoplamiento
    
    La lógica de negocio no depende del protocolo de comunicación.

    - Ejemplo: tu función para calcular el IVA puede usarse desde un endpoint REST o un evento en Kafka sin modificar el código.


## Ejemplo Básico

Supongamos que queremos una función que convierta texto a mayúsculas.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;
import java.util.function.Function;

@Component
public class MyFunctions {

    @Bean
    public Function<String, String> uppercase() {
        return value -> value.toUpperCase();
    }
}
```

Qué pasa aquí?

Definimos una función uppercase que implementa ```Function<String, String>.```

Spring Cloud Function la reconoce como bean.

¿Cómo la usamos?

Si corres tu aplicación con Spring Boot + Web, puedes acceder vía HTTP:

```bash
POST http://localhost:8080/uppercase
Body: "hola mundo"
```

```bash
"HOLA MUNDO"
```

## Taller Práctico: “De REST a Funciones con Spring Cloud Function”
### 🎯 Objetivo

Que los estudiantes comprendan cómo usar Spring Cloud Function para:

    1. Crear funciones (```Function, Consumer, Supplier```) en Java.
    2. Ejecutarlas en una aplicación Spring Boot.
    3. Invocarlas fácilmente desde HTTP sin necesidad de un ```@RestController.```


1. Crear un nuevo proyecto

    Generar un proyecto desde [Spring Initializr](https://start.spring.io/)

    - Project: Maven o Gradle
    - Language: Java
    - Spring Boot: 3.5.6

    Dependencies:

    - Spring Cloud Function Web
    - Spring Boot Starter Web


2. Escribir la primera función

    Crear una clase ```MyFunctions.java:```

    ```java
    import org.springframework.context.annotation.Bean;
    import org.springframework.stereotype.Component;

    import java.util.function.Function;
    import java.util.function.Supplier;
    import java.util.function.Consumer;

    @Component
    public class MyFunctions {

        // Function: transforma un String a mayúsculas
        @Bean
        public Function<String, String> uppercase() {
            return value -> value.toUpperCase();
        }

        // Supplier: genera un saludo fijo
        @Bean
        public Supplier<String> helloSupplier() {
            return () -> "Hola desde Spring Cloud Function!";
        }

        // Consumer: imprime un mensaje recibido
        @Bean
        public Consumer<String> printMessage() {
            return message -> System.out.println("Mensaje recibido: " + message);
        }
    }
    ```

3. Invocar funciones con HTTP

Levantar la app ``(mvn spring-boot:run)`` o ```./gradlew bootRun`` y probar:


- **Function (con entrada/salida)**

 

    ```bash
        "SPRING CLOUD FUNCTION"
    ```

- **Supplier (sin entrada, solo salida)**
    ``` bash
    GET http://localhost:8080/helloSupplier
    ```

    ```bash
    "Hola desde Spring Cloud Function!"
    ```

- **Consumer (solo entrada, sin respuesta)**
    ```bash
    POST http://localhost:8080/printMessage
    Body: "Prueba desde el taller"
    ```
    ```bash
    Mensaje recibido: Prueba desde el taller
    ```

4. Reflexión guiada (preguntas a estudiantes)

    - ¿Qué diferencias encontraron entre ```Function, Supplier y Consumer```?

    - ¿Qué ventaja ven en no usar ```@RestController``` ni ```@RequestMapping```?

    - ¿Qué tan fácil sería reutilizar estas funciones en otro entorno (ejemplo: colas o serverless)?

5. Ejercicio propuesto

    Crear una función calculateTax que:

    - Reciba un número decimal (precio de un producto).
    - Devuelva el precio con IVA del 19%.
    - Probarlo desde Postman o cURL.