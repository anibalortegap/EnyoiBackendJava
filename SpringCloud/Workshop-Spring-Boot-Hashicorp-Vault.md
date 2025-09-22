# Taller Spring Boot + Hashicorp Vault

## Integración con Spring Boot
La integración es muy fluida gracias al proyecto Spring Cloud Vault. Actúa como un puente, permitiendo que tu aplicación Spring Boot trate los secretos de Vault como si fueran propiedades de un archivo ``` application.properties.```

## ¿Cómo funciona?
Al iniciar, tu aplicación Spring Boot se conecta a Vault, se autentica y extrae los secretos de una ruta específica. Luego, inyecta esos secretos en el ```Environment``` de Spring. A partir de ese momento, puedes acceder a ellos usando la anotación ```@Value``` o a través del objeto ```Environment```, como harías con cualquier otra propiedad.

### Ejercicio Básico de Uso
Vamos a simular la recuperación de una contraseña de base de datos desde Vault.

1. Levantar Vault en Modo Desarrollo
Para experimentar, no necesitamos una instalación compleja. Vault tiene un modo de desarrollo que funciona en memoria y es perfecto para pruebas locales.

    Instalar vault:

    ``` bash
    brew tap hashicorp/tap
    brew install hashicorp/tap/vault
    ```

    Ejecutar Vault:

    ``` bash
    vault server -dev
    ```

    La terminal te devolverá información importante. Copia y guarda el Root Token, lo necesitaremos. Por defecto, Vault se ejecutará en http://127.0.0.1:8200.

2. Guardar un Secreto en Vault
    Ahora, vamos a guardar una contraseña en Vault. Abre otra terminal y configura la variable de entorno para que el CLI de Vault sepa a qué servidor apuntar y cómo autenticarse.

    ``` bash 
    # En Linux/macOS
    export VAULT_ADDR='http://127.0.0.1:8200'
    export VAULT_TOKEN='<PEGA_AQUÍ_TU_ROOT_TOKEN>'

    # En Windows
    set VAULT_ADDR=http://127.0.0.1:8200
    set VAULT_TOKEN=<PEGA_AQUÍ_TU_ROOT_TOKEN>
    ```

    Ahora, guardemos un secreto en la ruta secret/banking-app:

    ```bash
    vault kv put secret/banking-app db.password="Secreta123!"
    ```

    Listo! El secreto ya está guardado de forma segura en Vault.

3. Crear el Proyecto Spring Boot
    Ve a [start.spring.io](https://start.spring.io/) y crea un proyecto con estas dependencias:

    Spring Web: Para crear un endpoint de prueba.

    Vault Configuration: La dependencia clave para la integración.

4. Configurar la Aplicación
    Añadir dependencias (si no las tienes):
    Asegúrate de que tu ```pom.xml o build.gradle``` incluya ```spring-cloud-starter-vault-config.```

    Configurar la conexión a Vault:
    Crea el archivo ```bootstrap.properties``` (o ```bootstrap.yml```) en ```src/main/resources```. Este archivo se carga antes que ```application.properties``` y es el lugar ideal para configurar la conexión a Vault.

    ```bootstrap.properties```

    ```properties
    # Nombre de la aplicación, usado para la ruta del secreto
    spring.application.name=nuestra-app

    # Configuración de Spring Cloud Vault
    spring.cloud.vault.uri=http://127.0.0.1:8200
    spring.cloud.vault.token=<PEGA_AQUÍ_TU_ROOT_TOKEN>
    ```

    _Nota: En un entorno real, nunca pondrías el token aquí. Usarías un método de autenticación más seguro como AppRole o Kubernetes Auth._

5. Leer el Secreto en el Código
    Ahora vamos a crear un controlador REST para mostrar que hemos leído el secreto con éxito.

    ```java
    package com.example.vaultdemo;

    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RestController;

    @RestController
    public class SecretController {

        // Spring inyecta aquí el valor de la propiedad 'db.password'
        // que fue leída desde Vault en la ruta 'secret/nuestra-app'
        @Value("${db.password}")
        private String dbPassword;

        @GetMapping("/password")
        public String getDbPassword() {
            // ¡OJO! Nunca expongas una contraseña real en un endpoint.
            // Esto es solo para fines demostrativos.
            return "La contraseña de la BD obtenida desde Vault es: " + dbPassword;
        }
    }
    ```

6. Ejecutar y Probar
    Inicia tu aplicación Spring Boot. Si todo está configurado correctamente, verás en los logs que se conecta a Vault.

    Abre tu navegador o un cliente API y ve a ```http://localhost:8080/password.``` 

    Deberías ver el mensaje:
    ```La contraseña de la BD obtenida desde Vault es: Secreta123!```

7. Crear una Clase de Configuración Type-Safe
    Primero, crearemos una clase POJO (Plain Old Java Object) que representará nuestra configuración obtenida desde Vault. Spring Boot la llenará automáticamente por nosotros.

    Crea una nueva clase llamada ```VaultConfig.java.```

    Añade el siguiente contenido:

    ```java
    package com.example.vaultdemo.config;

    import org.springframework.boot.context.properties.ConfigurationProperties;
    import org.springframework.context.annotation.Configuration;

    /**
     * Esta clase representa un bean de configuración que captura
     * propiedades con el prefijo "db". Spring Boot la llenará
     * automáticamente con los valores encontrados en Vault.
     */
    @Configuration
    @ConfigurationProperties(prefix = "db")
    public class VaultConfig {

        /**
         * Esta variable 'password' se mapeará automáticamente
         * desde la propiedad 'db.password' en Vault.
         */
        private String password;

        // Getters y Setters son necesarios para que Spring pueda inyectar los valores.
        public String getPassword() {
            return password;
        }

        public void setPassword(String password) {
            this.password = password;
        }
    }
    ```

8. Usar ```CommandLineRunner``` para Verificar los Secretos al Inicio
    Ahora, en lugar de un controlador, usaremos ```CommandLineRunner.``` Esta es una interfaz funcional que nos permite ejecutar código justo después de que el contexto de Spring se haya cargado, pero antes de que la aplicación empiece a aceptar peticiones. Es el lugar perfecto para tareas de inicialización.

    Crea una clase llamada ```SecretInitializer.java.```

    Añade el siguiente contenido:

    ```java
    package com.example.vaultdemo;

    import com.example.vaultdemo.config.VaultConfig;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.boot.CommandLineRunner;
    import org.springframework.stereotype.Component;

    /**
     * Este componente se ejecuta al iniciar la aplicación.
     * Su propósito es verificar que los secretos necesarios
     * han sido cargados correctamente desde Vault.
     */
    @Component
    public class SecretInitializer implements CommandLineRunner {

        private static final Logger log = LoggerFactory.getLogger(SecretInitializer.class);

        private final VaultConfig vaultConfig;

        /**
         * Inyectamos nuestra configuración de Vault a través del constructor.
         * Si el bean VaultConfig no se pudo crear (porque los secretos no se encontraron),
         * la aplicación fallará al iniciar, lo cual es una buena práctica (fail-fast).
         * @param vaultConfig El bean de configuración con los secretos.
         */
        public SecretInitializer(VaultConfig vaultConfig) {
            this.vaultConfig = vaultConfig;
        }

        @Override
        public void run(String... args) throws Exception {
            log.info("============================================================");
            log.info("✅ INICIALIZACIÓN DE SECRETOS COMPLETADA");
            if (vaultConfig.getPassword() != null && !vaultConfig.getPassword().isEmpty()) {
                log.info("   -> Contraseña de BD cargada exitosamente desde Vault.");
                // Aquí podríamos inicializar un pool de conexiones con esta contraseña,
                // o realizar cualquier otra tarea que dependa de este secreto.
            } else {
                log.error("   -> 🚨 ¡ERROR! La contraseña de la BD no se pudo cargar desde Vault.");
            }
            log.info("============================================================");
        }
    }
    ```

### Ventajas Clave de este Enfoque
Separación de Responsabilidades: La configuración vive en su propia clase (``VaultConfig``), y la lógica de negocio vive en otras (``SecretInitializer`` en este demo).

Seguridad: No expones secretos en endpoints de prueba. La verificación se hace internamente y de forma segura.

Robustez (Fail-Fast): Si Spring no puede cargar los secretos de Vault para crear el bean ``VaultConfig``, la aplicación fallará al iniciar. Esto es bueno, ya que previene que la aplicación se ejecute en un estado inválido y sin su configuración necesaria.

Código Limpio y Mantenible: Es mucho más fácil de leer, probar y mantener que tener anotaciones ``@Value`` esparcidas por todo el código.

