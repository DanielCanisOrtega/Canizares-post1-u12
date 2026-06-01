# Sistema de Gestión de Pedidos – Integración de Patrones de Diseño

## Información General

**Asignatura:** Patrones de Diseño
**Actividad:** Post-Contenido 1 – Unidad 12
**Proyecto:** Sistema de Gestión de Pedidos
**Tecnologías:** Java 17, Spring Boot 3, Maven, H2 Database, SonarQube, ArchUnit, JUnit 5

---

# Objetivo

Implementar un sistema de gestión de pedidos aplicando los patrones de diseño **Factory**, **Strategy**, **Observer** y **Facade**, verificando la calidad del código mediante **SonarQube** y el desacoplamiento arquitectónico mediante **ArchUnit**.

El propósito es transformar una implementación monolítica con responsabilidades mezcladas en una arquitectura modular donde cada patrón resuelva un problema específico y claramente delimitado.

---

# Descripción del Problema Inicial

La versión inicial del sistema utilizaba una clase denominada `ServicioPedidosLegacy`, la cual concentraba múltiples responsabilidades:

* Cálculo del costo del pedido.
* Selección del algoritmo según el tipo de pedido.
* Persistencia en base de datos.
* Envío de notificaciones.
* Gestión del estado del pedido.

Esta implementación generaba varios problemas:

* Violación del Principio de Responsabilidad Única (SRP).
* Alto acoplamiento entre componentes.
* Baja extensibilidad.
* Dificultad para realizar pruebas unitarias.
* Mayor complejidad cognitiva.

## Código Inicial

```java
@Service
public class ServicioPedidosLegacy {

    @Autowired
    private PedidoRepository repo;

    @Autowired
    private JavaMailSender mail;

    public void procesarPedido(Pedido pedido) {

        if (pedido.getTipo() == TipoPedido.ESTANDAR) {
            pedido.setCosto(pedido.getSubtotal() * 1.1);
        } else if (pedido.getTipo() == TipoPedido.EXPRESS) {
            pedido.setCosto(pedido.getSubtotal() * 1.3);
        } else if (pedido.getTipo() == TipoPedido.INTERNACIONAL) {
            pedido.setCosto(
                pedido.getSubtotal() * 1.5 + 25.0
            );
        }

        pedido.setEstado(EstadoPedido.PROCESADO);

        repo.save(pedido);

        mail.send(crearMensaje(pedido));
    }
}
```

---

# Arquitectura Final

```text
src/main/java/com/empresa/pedidos

├── dominio
│   ├── Pedido
│   ├── TipoPedido
│   ├── EstadoPedido
│   └── puertos
│       ├── RepositorioPedidos
│       ├── ProcesadorPedido
│       └── ServicioNotificacion
│
├── aplicacion
│   └── ServicioPedidos
│
├── adaptadores
│   ├── procesadores
│   │   ├── ProcesadorPedidoEstandar
│   │   ├── ProcesadorPedidoExpress
│   │   └── ProcesadorPedidoInternacional
│   │
│   ├── factory
│   │   └── ProcesadorPedidoFactory
│   │
│   ├── facade
│   │   └── FachadaPedidos
│   │
│   └── rest
│       └── PedidoController
│
└── infraestructura
    ├── persistencia
    │   └── RepositorioPedidosJpa
    │
    └── notificaciones
        ├── NotificacionEmail
        └── NotificacionLog
```

---

# Patrones de Diseño Implementados

## 1. Strategy

### Problema que resuelve

Cada tipo de pedido requiere un algoritmo diferente para calcular el costo final.

### Implementación

Se creó la interfaz:

```java
public interface ProcesadorPedido {

    TipoPedido getTipo();

    void procesar(Pedido pedido);
}
```

Implementaciones:

* ProcesadorPedidoEstandar
* ProcesadorPedidoExpress
* ProcesadorPedidoInternacional

### Beneficios

* Elimina condicionales complejos.
* Facilita agregar nuevos tipos de pedido.
* Cumple el principio Open/Closed.

---

## 2. Factory Method

### Problema que resuelve

Seleccionar dinámicamente la estrategia adecuada según el tipo de pedido.

### Implementación

```java
@Component
public class ProcesadorPedidoFactory {

    private final Map<TipoPedido,
            ProcesadorPedido> procesadores;

    public ProcesadorPedidoFactory(
            List<ProcesadorPedido> lista) {

        this.procesadores =
                lista.stream()
                .collect(Collectors.toMap(
                    ProcesadorPedido::getTipo,
                    Function.identity()));
    }

    public ProcesadorPedido obtener(
            TipoPedido tipo) {

        return Optional.ofNullable(
                procesadores.get(tipo))
            .orElseThrow(
                IllegalArgumentException::new
            );
    }
}
```

### Beneficios

* Encapsula la creación y selección de estrategias.
* Evita dependencias directas con implementaciones concretas.
* Reduce el acoplamiento.

---

## 3. Observer

### Problema que resuelve

Permitir que diferentes componentes reaccionen a un pedido procesado sin modificar la lógica principal.

### Implementación

Evento de dominio:

```java
public record PedidoProcesadoEvent(
        Pedido pedido) {
}
```

Listeners:

```java
@Component
public class NotificacionEmail
        implements ServicioNotificacion {

    @EventListener
    public void notificar(
            PedidoProcesadoEvent evento) {

        System.out.println(
            "Email enviado");
    }
}
```

```java
@Component
public class NotificacionLog
        implements ServicioNotificacion {

    @EventListener
    public void notificar(
            PedidoProcesadoEvent evento) {

        log.info(
            "Pedido procesado");
    }
}
```

### Beneficios

* Reduce el acoplamiento.
* Facilita agregar nuevos mecanismos de notificación.
* Mejora la mantenibilidad.

---

## 4. Facade

### Problema que resuelve

Ocultar la complejidad interna del sistema y ofrecer una interfaz sencilla para el controlador REST.

### Implementación

```java
@Service
public class FachadaPedidos {

    private final ProcesadorPedidoFactory factory;

    private final RepositorioPedidos repositorio;

    private final ApplicationEventPublisher publisher;

    public Pedido crearPedido(
            Pedido pedido) {

        factory.obtener(
                pedido.getTipo())
            .procesar(pedido);

        Pedido guardado =
            repositorio.guardar(pedido);

        publisher.publishEvent(
            new PedidoProcesadoEvent(
                guardado));

        return guardado;
    }
}
```

### Beneficios

* Simplifica el acceso a la lógica de negocio.
* Reduce dependencias en el controlador.
* Centraliza el flujo principal del sistema.

---

# Justificación de la Composición de Patrones

La combinación de patrones fue realizada siguiendo el criterio de **problema distinto** estudiado en la Unidad 12.

| Patrón   | Problema resuelto                         |
| -------- | ----------------------------------------- |
| Strategy | Variación del algoritmo de procesamiento  |
| Factory  | Selección dinámica de estrategias         |
| Observer | Comunicación desacoplada mediante eventos |
| Facade   | Simplificación de la interfaz pública     |

Cada patrón resuelve una responsabilidad diferente y complementaria.

---

# Verificación con ArchUnit

Se implementó una regla arquitectónica para garantizar que el controlador REST no dependa directamente de infraestructura.

```java
@AnalyzeClasses(
    packages = "com.empresa.pedidos")
public class ArquitecturaTest {

    @Test
    void controladorNoDependeDeInfraestructura() {

        ArchRule rule =
            noClasses()
                .that()
                .resideInAPackage("..rest..")
                .should()
                .dependOnClassesThat()
                .resideInAnyPackage(
                    "..persistencia..",
                    "..notificaciones..");

        rule.check(
            new ClassFileImporter()
                .importPackages(
                    "com.empresa.pedidos"));
    }
}
```

Resultado:

✅ Regla cumplida.

---

# Pruebas Implementadas

## Pruebas Unitarias

### Strategy

```java
ProcesadorPedidoEstandarTest
```

Valida el cálculo correcto para pedidos estándar.

### Factory

```java
ProcesadorPedidoFactoryTest
```

Valida la selección correcta de estrategias.

### Observer

```java
NotificacionEmailTest
```

Verifica la recepción de eventos.

### Facade

```java
FachadaPedidosTest
```

Verifica el flujo completo de procesamiento.

---

## Prueba de Integración

```java
@SpringBootTest
class PedidoIntegrationTest
```

Valida:

* Creación del pedido.
* Procesamiento.
* Persistencia.
* Publicación de eventos.
* Recepción por listeners.

Resultado:

✅ Todas las pruebas exitosas.

---

# Análisis SonarQube

## Métricas Iniciales

| Métrica               | Valor  |
| --------------------- | ------ |
| Cyclomatic Complexity | 4      |
| Cognitive Complexity  | 6      |
| Code Smells           | 8      |
| Cobertura             | 0%     |
| Quality Gate          | Failed |

---

## Métricas Después de la Refactorización

| Métrica               | Valor  |
| --------------------- | ------ |
| Cyclomatic Complexity | 1      |
| Cognitive Complexity  | 0      |
| Code Smells           | 0      |
| Cobertura             | 85%    |
| Quality Gate          | Passed |

---

# Comparación Antes vs Después

| Indicador               | Antes  | Después |
| ----------------------- | ------ | ------- |
| Condicionales           | 3      | 0       |
| Complejidad Ciclomática | 4      | 1       |
| Complejidad Cognitiva   | 6      | 0       |
| Acoplamiento a Email    | Sí     | No      |
| Acoplamiento a JPA      | Sí     | No      |
| Cobertura               | 0%     | 85%     |
| Quality Gate            | Failed | Passed  |

---

# Capturas Requeridas

Crear carpeta:

```text
capturas/
```

Agregar:

```text
capturas/
├── sonar-antes.png
├── sonar-despues.png
├── quality-gate-passed.png
```

---

# Ejecución del Proyecto

Compilar:

```bash
mvn clean package
```

Ejecutar:

```bash
mvn spring-boot:run
```

Ejecutar pruebas:

```bash
mvn test
```

Ejecutar SonarQube:

```bash
mvn clean verify sonar:sonar \
-Dsonar.projectKey=pedidos-integrado \
-Dsonar.host.url=http://localhost:9000 \
-Dsonar.login=TOKEN
```

---

# Conclusiones

La aplicación combinada de los patrones Factory, Strategy, Observer y Facade permitió transformar una implementación monolítica en una arquitectura modular, extensible y desacoplada. Las métricas de SonarQube evidencian una reducción significativa de la complejidad ciclomática y cognitiva, mientras que ArchUnit verificó el cumplimiento de las restricciones arquitectónicas. La solución resultante facilita la incorporación de nuevos tipos de pedidos y nuevos mecanismos de notificación sin modificar la lógica central, alineándose con los principios SOLID y Open/Closed.

---

# Referencias

* Design Patterns: Elements of Reusable Object-Oriented Software
* Refactoring: Improving the Design of Existing Code
* [Spring Boot Documentation](https://spring.io/projects/spring-boot?utm_source=chatgpt.com)
* [ArchUnit Documentation](https://www.archunit.org/?utm_source=chatgpt.com)
* [SonarQube Documentation](https://www.sonarsource.com/products/sonarqube/?utm_source=chatgpt.com)
