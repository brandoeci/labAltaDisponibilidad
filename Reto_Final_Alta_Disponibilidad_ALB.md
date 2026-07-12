# Reto Final: Alta Disponibilidad con Application Load Balancer en AWS

**Asignatura:** Arquitecturas de Software
**Estudiante:** Hildebrando Peña
**Plataforma:** AWS Academy Learner Lab
**Región:** us-east-1 (N. Virginia)

---

## 1. Diagrama de arquitectura implementada

```
Usuario / Navegador / curl
          |
          v
Application Load Balancer (alb-ha-web)
DNS público
          |
   +------+------+
   v             v
EC2 Web A     EC2 Web B
AZ us-east-1a AZ us-east-1b
Apache HTTPD  Apache HTTPD
```

La arquitectura se implementó completamente por línea de comandos (AWS CLI), replicando lo que la guía describe hacer por consola web. Los componentes creados fueron:

| Elemento | Nombre | Función |
|---|---|---|
| Security Group ALB | `alb-ha-sg` | Permite HTTP (puerto 80) desde `0.0.0.0/0` |
| Security Group EC2 | `ec2-ha-sg` | Permite HTTP únicamente desde `alb-ha-sg` |
| Instancia A | `web-ha-a` | EC2 en us-east-1a, Apache HTTPD |
| Instancia B | `web-ha-b` | EC2 en us-east-1b, Apache HTTPD |
| Target Group | `tg-ha-web` | Registra ambas instancias, health check en `/health` |
| Load Balancer | `alb-ha-web` | Internet-facing, distribuye tráfico HTTP |

---

## 2. Captura de las dos instancias EC2

**[PANTALLAZO AQUÍ: INSTANCIAS EC2 web-ha-a Y web-ha-b EN ESTADO RUNNING]**

---

## 3. Captura del Target Group con targets Healthy

**[PANTALLAZO AQUÍ: TARGET GROUP tg-ha-web CON TARGETS EN ESTADO HEALTHY]**

---

## 4. Captura del Application Load Balancer

**[PANTALLAZO AQUÍ: LOAD BALANCER alb-ha-web ESTADO ACTIVE Y DNS]**

---

## 5. Evidencia de respuesta desde instancia A

**[PANTALLAZO AQUÍ: RESPUESTA DEL NAVEGADOR DESDE INSTANCIA A]**

---

## 6. Evidencia de respuesta desde instancia B

**[PANTALLAZO AQUÍ: RESPUESTA DEL NAVEGADOR DESDE INSTANCIA B]**

---

## 7. Evidencia de falla simulada

**[PANTALLAZO AQUÍ: TERMINAL CON describe-target-health, INSTANCIA A UNHEALTHY E INSTANCIA B HEALTHY]**

Durante la prueba, se detuvo manualmente la instancia A (`aws ec2 stop-instances`). El Target Group la marcó como `unhealthy` tras fallar el health check en `/health`. Se realizaron 6 peticiones consecutivas al DNS del balanceador, y el 100% de las respuestas provinieron de la instancia B, sin ningún error visible para el usuario final.

---

## 8. Explicación de cómo el balanceador mantiene la disponibilidad

El Application Load Balancer mantiene la disponibilidad del sistema mediante tres mecanismos combinados:

1. **Redundancia geográfica:** las dos instancias EC2 están desplegadas en zonas de disponibilidad distintas (us-east-1a y us-east-1b). Si toda una zona de disponibilidad falla, la otra sigue operativa.

2. **Health checks activos:** el Target Group consulta la ruta `/health` de cada instancia cada 15 segundos. Si una instancia falla dos chequeos consecutivos, se marca como `unhealthy` y el ALB deja de enviarle tráfico de inmediato, sin intervención humana.

3. **Recuperación automática de la rotación:** cuando una instancia vuelve a responder correctamente (dos chequeos exitosos consecutivos), el ALB la reincorpora automáticamente al balanceo sin que nadie tenga que reconfigurar nada.

En la prueba realizada, al detener la instancia A el sistema completo no presentó downtime: el usuario que consulta el DNS del balanceador nunca percibió la diferencia, porque el tráfico se redirigió por completo a la instancia B.

---

## 9. Limitaciones de la arquitectura

- **No hay recuperación automática de instancias caídas.** El ALB deja de enviar tráfico a una instancia no saludable, pero no crea una instancia nueva para reemplazarla. Si la instancia A hubiera fallado por una razón real (no un stop manual), habría quedado caída indefinidamente hasta que alguien la reinicie manualmente.
- **Capacidad fija:** con solo 2 instancias, si ambas fallan simultáneamente (o si el tráfico crece mucho), no hay manera automática de escalar horizontalmente.
- **Sin HTTPS:** el listener solo maneja HTTP en el puerto 80, sin cifrado de extremo a extremo.
- **Instancias en subnets públicas:** aunque el Security Group restringe el acceso directo, las instancias siguen teniendo IP pública asignada, lo cual amplía la superficie de exposición.
- **Sin gestión centralizada de logs o métricas:** no hay CloudWatch Agent ni logging del ALB habilitado (restricción propia del entorno AWS Academy).

---

## 10. Propuesta de mejora hacia producción

Para evolucionar esta arquitectura hacia un entorno de producción real, se proponen las siguientes mejoras:

**Recuperación automática:**
Reemplazar las instancias EC2 manuales por un **Auto Scaling Group** con un Launch Template, configurado con capacidad mínima de 2 y deseada de 2-3. El ASG usaría los health checks del ELB para detectar y reemplazar automáticamente instancias no saludables, cerrando la limitación principal detectada en la sección 9.

**Instancias privadas:**
Mover las instancias EC2 a **subnets privadas**, sin IP pública, accesibles únicamente a través del ALB. El tráfico saliente (actualizaciones, dependencias) se manejaría con un NAT Gateway.

**HTTPS:**
Agregar un **listener HTTPS (puerto 443)** con un certificado gestionado por **AWS Certificate Manager (ACM)**, y redirigir todo el tráfico HTTP a HTTPS mediante una regla de listener.

**Logs y métricas:**
Habilitar **access logs del ALB** hacia un bucket S3, y usar **CloudWatch** (Agent y Alarms) para monitorear CPU, latencia, conteo de requests y estado de los targets, con alertas automáticas ante degradación.

**Despliegues sin caída (zero-downtime deployments):**
Adoptar una estrategia de **despliegue azul-verde (blue-green)** o **rolling update** mediante el Auto Scaling Group: se lanzan instancias nuevas con la versión actualizada, se espera a que pasen los health checks, y solo entonces se retiran las instancias antiguas del Target Group.

**Base de datos altamente disponible:**
Incorporar **Amazon RDS Multi-AZ** (o Aurora) para persistencia de datos, con failover automático a una réplica en otra zona de disponibilidad en caso de falla del nodo primario, evitando que la base de datos se convierta en el nuevo punto único de falla.

---

## Conclusión

El laboratorio permitió comprobar de forma práctica que la alta disponibilidad no depende de tener "más servidores", sino de combinar correctamente redundancia, balanceo de carga, health checks activos y una red de seguridad bien segmentada. La prueba de falla simulada confirmó que el sistema puede perder una instancia completa sin que el usuario final lo perciba, cumpliendo el objetivo central del laboratorio.
