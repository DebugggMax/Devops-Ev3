🚀 Innovatech Chile - Plataforma de Orquestación con Amazon EKS
---
📌 Descripción
Este repositorio contiene la infraestructura y configuración necesaria para desplegar la plataforma
Innovatech Chile sobre Amazon Elastic Kubernetes Service (EKS), utilizando prácticas modernas de Infraestructura como Código, 
contenedores Docker y un pipeline de integración y despliegue continuo (CI/CD).
---
El proyecto fue desarrollado como parte de la Evaluación Parcial N°3, 
teniendo como objetivo implementar una solución:
- Escalable
- Altamente disponible
- Automatizada
- Basada en servicios desacoplados
- Preparada para ambientes productivos
---
👥 Integrantes
Maximiliano Olguin
Camila Ibarra

🏛️ Arquitectura General

[ CLIENTE INTERNET ]
                                         │
                                         ▼ (Tráfico HTTP/HTTPS)
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ AWS ACADEMY (VPC / Cloud Region)                                                       │
│                                                                                        │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ AMAZON EKS CLUSTER (tienda-eks)                                                  │  │
│  │                                                                                  │  │
│  │  ┌────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │ NODO EC2 (t3.large) -> Capacidad Real Asignable: 1930m CPU                 │  │  │
│  │  │                                                                            │  │  │
│  │  │  ├──  CAPA DE ACCESO (Namespace: ingress-nginx)                          │  │  │
│  │  │  │   └── [ Ingress-Nginx Controller Pod ] ─── (Reserva fija: 100m CPU)     │  │  │
│  │  │  │                │                  │                                     │  │  │
│  │  │  │    (/api/ventas)│                  │(/api/despachos)                     │  │  │
│  │  │  │                ▼                  ▼                                     │  │  │
│  │  │  │                                                                         │  │  │
│  │  │  ├──  CAPA DE SERVICIOS (Namespace: default)                              │  │  │
│  │  │  │   ├── [ Service: backend-ventas ]     ├── [ Service: backend-despachos ]│  │  │
│  │  │  │   │     (ClusterIP Interno)           │     (ClusterIP Interno)         │  │  │
│  │  │  │   └────────────┬──────────────────────└────────────┬────────────────────┘  │  │
│  │  │  │                ▼ (Balanceo Round-Robin)            ▼                       │  │  │
│  │  │  │       ┌─────────────────┐                 ┌─────────────────┐           │  │  │
│  │  │  │       │ backend-ventas  │                 │backend-despachos│           │  │  │
│  │  │  │       │      (Pod)      │                 │      (Pod)      │           │  │  │
│  │  │  │       │ Request: 200m   │                 │ Request: 200m   │           │  │  │
│  │  │  │       │ Máximo:  800m   │                 │ Máximo:  800m   │           │  │  │
│  │  │  │       └────────▲────────┘                 └────────▲────────┘           │  │  │
│  │  │  │                │                                   │                    │  │  │
│  │  │  │                └─────────────────┬─────────────────┘                    │  │  │
│  │  │  │                                  │ (Escala según uso)                   │  │  │
│  │  │  ├──  CAPA DE CONTROL (HPA)       ▼                                      │  │  │
│  │  │  │   └── [ Horizontal Pod Autoscaler ]                                     │  │  │
│  │  │  │        ├── Configuración: minReplicas: 1  /  maxReplicas: 2             │  │  │
│  │  │  │        └── Umbral disparo: > 50% CPU Utilización                        │  │  │
│  │  │  │                                  ▲                                      │  │  │
│  │  │  │                                  │ (Métricas en tiempo real 3% / 4%)    │  │  │
│  │  │  ├── CAPA DE MÉTRICAS INTERNAS  │                                      │  │  │
│  │  │  │   └── [ Metrics Server Pod ] ────┘                                      │  │  │
│  │  │  │                                                                         │  │  │
│  │  │  └──  COSTO FIJO DE INFRAESTRUCTURA (Namespace: amazon-cloudwatch / system)│  │  │
│  │  │      ├── [ CloudWatch Agent Pod ] ────────── (Reserva fija: 250m CPU)       │  │  │
│  │  │      ├── [ Fluent-Bit Log Agent Pod ] ────── (Reserva fija: 50m CPU)        │  │  │
│  │  │      └── [ CoreDNS / Kube-Proxy Pods ] ───── (Reserva fija: 350m CPU)       │  │  │
│  │  │                                                                            │  │  │
│  │  └────────────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────────────────┘
---
🏗️ 2. Infraestructura y Redes
---
Para el aprovisionamiento de este entorno,
se implementó una estrategia de despliegue híbrida que combina servicios gestionados con herramientas de Infraestructura como Código (IaC):

Topología de Red e Infraestructura Base (Consola AWS):
---
VPC Dedicada: Segmentación de una red lógica aislada.

Subredes Públicas y Privadas: Distribuidas en múltiples zonas de disponibilidad de AWS para
garantizar que el clúster siga operativo si una zona física falla.

Registros de Contenedores (Amazon ECR): Aprovisionamiento de repositorios privados para 
almacenar de forma segura las imágenes compiladas del Frontend y Backend.

Gobernanza IAM: Configuración del EKS node role para permitir que las instancias de cómputo se acoplen al plano de control de manera segura.

Orquestación Automatizada (Terraform):
---
Uso de manifiestos en Terraform para declarar y desplegar de manera automatizada el clúster de Amazon EKS
y sus Node Groups de forma estandarizada y repetible.


---
 📦3. Despliegue de Servicios 

Los microservicios fueron desplegados y comunicados de manera desacoplada dentro de Kubernetes:
---
Frontend (Capa Pública):
--
Configurado mediante un servicio tipo LoadBalancer. 
AWS genera un DNS público dinámico que actúa como el enlace de acceso para el cliente.

Backend (Capa Privada):
--
Configurado mediante un servicio tipo ClusterIP. 
No posee dirección pública, protegiendo las transacciones del negocio de cualquier acceso directo desde el exterior.

Comunicación Interna:
--
El Frontend se conecta al Backend utilizando la resolución de nombres interna del clúster de Kubernetes:
http://backend-service.innovatech.svc.cluster.local:8000

Esta ruta de comunicación se inyecta dinámicamente mediante variables de entorno, evitando dejar IPs fijas en el código de la aplicación.


---
📈 4. Autoescalado Horizontal - HPA
---
La resiliencia de la plataforma ante incrementos de tráfico masivo se configuró mediante políticas de Horizontal Pod Autoscaler (HPA):

Límites de Escalado: Mínimo de 2 réplicas activas por servicio (alta disponibilidad básica) y un techo máximo de 5 réplicas elásticas.

Métrica de Activación: Umbral de uso promedio de CPU fijado al 50%.

Justificación del Umbral: Un límite del 50% funciona como una alerta preventiva eficaz. 
Permite que Kubernetes detecte el incremento de carga y encienda nuevos contenedores con la holgura de tiempo suficiente para
inicializar el sistema antes de que los pods actuales se saturen y la página web empiece a andar lenta.


---
🐙 5. Pipeline de Integración y Despliegue Continuo 
---
La automatización de entregas de código se configuró mediante GitHub Actions 
(.github/workflows/deploy.yml) ejecutando los siguientes pasos de manera estrictamente vertical:

Checkout: Descarga el código fuente actualizado del repositorio de GitHub.
---
Autenticación AWS: Login seguro en AWS utilizando las credenciales dinámicas de AWS Academy.
---
Build de Imágenes: Compilación aislada de los Dockerfiles locales para las aplicaciones de ./frontend y ./backend.
---
Push a Amazon ECR: Envío automático de las nuevas imágenes compiladas a los registros privados.
---
Despliegue Continuo: Ejecución automática de comandos kubectl apply para actualizar el clúster en caliente sin caída del sistema.
---

---
🔎 6. Gestión de Secrets y Reporte de Logs
---
Como parte del control de calidad, observabilidad y depuración del clúster:

El Problema Detectado (Logs):
Al desplegar los manifiestos de las aplicaciones, los pods se quedaron en error de tipo ImagePullBackOff. 
Al analizar los eventos con el comando kubectl describe pod, 
detectamos errores de denegación de permisos de red (HTTP 403) desde el registro privado de Amazon ECR.

La Causa Raíz:
Las limitaciones de roles del sandbox de AWS Academy bloquean la comunicación anónima del clúster con registros de imágenes privados, 
requiriendo validación por contraseña.

La Mitigación:
Creamos un objeto Kubernetes Secret (docker-registry) llamado ecr-registry-key alimentado por un token temporal generado con aws ecr get-login-password.
Se asoció este secret en los despliegues de la aplicación usando imagePullSecrets, 
logrando la autorización de las descargas de forma segura.

Validación Empírica de Red:
Toque final de control: Desplegamos una base de datos MariaDB usando una imagen pública de Docker Hub. 
Este contenedor pasó inmediatamente a estado 1/1 Running, 
demostrando que el problema era exclusivamente de autenticación en AWS Academy y
que la arquitectura de red general del clúster funciona a la perfección.

