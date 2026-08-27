# Aplicación de prueba con HTTPS en Kubernetes

Este proyecto despliega una aplicación Nginx sencilla en Kubernetes y la publica mediante HTTPS usando un `Ingress` de Traefik. Está pensado principalmente para pruebas locales con k3s y el dominio `mi-app.local`.

## Objetivo del proyecto

El objetivo es aprender cómo Kubernetes publica una aplicación web mediante HTTPS y cómo se relacionan algunos de sus recursos principales. En este ejemplo:

- Nginx representa nuestra aplicación web.
- Un `Deployment` mantiene la aplicación funcionando.
- Un `Service` proporciona una dirección estable dentro del clúster.
- Un `Ingress` publica la aplicación fuera del clúster.
- Traefik recibe las peticiones y gestiona la conexión HTTPS.
- Un `Secret` almacena el certificado TLS y su clave privada.

Este proyecto **utiliza un certificado existente**, pero todavía no lo genera ni lo renueva automáticamente. La automatización de certificados con herramientas como cert-manager se plantea como un paso posterior.

## Explicación para principiantes

Puedes imaginar un clúster de Kubernetes como un pequeño centro comercial y la aplicación como una tienda llamada `mi-app.local`. Kubernetes organiza la tienda, mantiene copias disponibles y decide cómo llegan los visitantes hasta ella.

### Pod: donde se ejecuta la aplicación

Un **Pod** es una de las unidades básicas de Kubernetes. Dentro de un Pod se ejecutan uno o más contenedores. En este proyecto, cada Pod contiene un servidor web basado en la imagen `nginx:alpine`.

Los Pods son temporales: pueden fallar, eliminarse y volver a crearse con otra dirección IP. Por esa razón normalmente no se administran de forma manual ni se accede directamente a ellos.

### Deployment: mantiene los Pods funcionando

El `Deployment` llamado `https-test-app` declara que deben existir dos réplicas de Nginx:

```yaml
spec:
  replicas: 2
```

Kubernetes intenta conservar siempre ese **estado deseado**:

```text
Deployment https-test-app
├── Pod con Nginx
└── Pod con Nginx
```

Si uno de los Pods falla, el Deployment solicita la creación de otro. Esta es una idea central de Kubernetes: nosotros describimos cómo queremos que se vea el sistema y Kubernetes trabaja continuamente para mantenerlo así.

### Service: una dirección estable para los Pods

Como los Pods pueden cambiar, el `Service` llamado `https-test-service` proporciona un punto de acceso estable dentro del clúster. Encuentra los Pods mediante la etiqueta `app: https-test`:

```text
Service https-test-service
├── Pod con la etiqueta app=https-test
└── Pod con la etiqueta app=https-test
```

Cuando el Service recibe una petición, puede enviarla a cualquiera de los dos Pods. De esta manera, otros componentes no necesitan conocer la dirección individual de cada Pod.

### Ingress: la entrada a la aplicación

El Service permite llegar a Nginx desde dentro del clúster. Para publicar la aplicación se utiliza el `Ingress` llamado `https-test-ingress`, que contiene una regla para el dominio `mi-app.local`.

Un Ingress es un conjunto de reglas; necesita un **Ingress Controller** para ejecutarlas. En este proyecto se utiliza Traefik, incluido de manera predeterminada en k3s. Traefik actúa como el recepcionista del clúster: recibe una petición, examina su dominio y la dirige al Service correcto.

```text
Petición para mi-app.local
            │
            ▼
     Traefik / Ingress
            │
            ▼
 https-test-service
       ┌────┴────┐
       ▼         ▼
   Pod Nginx  Pod Nginx
```

### Secret: dónde se guarda el certificado

Para establecer una conexión HTTPS se necesitan un certificado y su clave privada:

```text
tls.crt → certificado público
tls.key → clave privada y sensible
```

El comando de despliegue guarda ambos archivos en un `Secret` de tipo TLS llamado `mi-app-tls`. El Ingress hace referencia a ese Secret para que Traefik pueda presentar el certificado al visitante.

> **Importante:** un Secret de Kubernetes permite separar información sensible de los manifiestos de la aplicación, pero su contenido no está necesariamente cifrado dentro del clúster. En un entorno real también se deben configurar controles de acceso y cifrado, o utilizar un gestor de secretos.

### Recorrido completo de una petición

Cuando se visita `https://mi-app.local`, sucede lo siguiente:

1. La computadora resuelve `mi-app.local` hacia la dirección de entrada del clúster.
2. La petición llega a Traefik.
3. Traefik obtiene el certificado desde el Secret `mi-app-tls` y establece la conexión HTTPS.
4. La regla del Ingress envía la petición al Service `https-test-service`.
5. El Service selecciona uno de los Pods de Nginx.
6. Nginx genera la respuesta, que regresa al visitante a través de Traefik.

La conexión HTTPS termina en Traefik. Dentro de este ejemplo, Traefik se comunica con Nginx mediante HTTP:

```text
Cliente ── HTTPS ──> Traefik ── HTTP ──> Service ──> Pod Nginx
```

Este patrón se conoce como **terminación TLS**. Gracias a él, cada Pod no necesita configurar su propia copia del certificado.

### ¿Qué hace `kubectl apply`?

Al ejecutar:

```bash
kubectl apply -f app-with-https.yaml
```

se le pide a Kubernetes que lea el manifiesto y cree o actualice los recursos descritos. Kubernetes compara el archivo con el estado actual del clúster y realiza los cambios necesarios.

## Componentes

El manifiesto [`app-with-https.yaml`](./app-with-https.yaml) crea los siguientes recursos en el namespace `default`:

- **Deployment `https-test-app`**: ejecuta dos réplicas de `nginx:alpine`, con solicitudes y límites de CPU y memoria.
- **Service `https-test-service`**: expone internamente Nginx por el puerto 80.
- **Ingress `https-test-ingress`**: publica el servicio bajo `mi-app.local`, configura TLS y solicita a Traefik la redirección de HTTP a HTTPS.

Para crear el Secret también debes proporcionar:

- `tls.crt`: un certificado para `mi-app.local`.
- `tls.key`: la clave privada correspondiente al certificado.

Estos archivos no están incluidos en el repositorio. `tls.key` es material sensible y no debe compartirse ni almacenarse en un repositorio público.

## Requisitos

- Un clúster Kubernetes accesible mediante `kubectl`.
- Traefik instalado como Ingress Controller (incluido de forma predeterminada en k3s).
- Permisos para crear recursos en el namespace `default`.
- Acceso para resolver `mi-app.local` hacia la IP de entrada del clúster.

## Despliegue

1. Comprueba que el clúster esté disponible:

   ```bash
   kubectl cluster-info
   kubectl get nodes
   ```

2. Crea el Secret TLS que espera el Ingress:

   ```bash
   kubectl create secret tls mi-app-tls \
     --cert=tls.crt \
     --key=tls.key \
     --namespace=default
   ```

   Si el Secret ya existe y necesitas actualizarlo:

   ```bash
   kubectl create secret tls mi-app-tls \
     --cert=tls.crt \
     --key=tls.key \
     --namespace=default \
     --dry-run=client -o yaml | kubectl apply -f -
   ```

3. Aplica el manifiesto:

   ```bash
   kubectl apply -f app-with-https.yaml
   ```

4. Consulta la dirección asignada al Ingress:

   ```bash
   kubectl get ingress https-test-ingress -n default
   ```

5. Haz que `mi-app.local` resuelva a la IP del Ingress. Para una prueba local puedes agregar una entrada en `/etc/hosts`:

   ```text
   <IP_DEL_INGRESS> mi-app.local
   ```

6. Prueba la aplicación:

   ```bash
   curl -k https://mi-app.local
   ```

Si utilizas un certificado autofirmado, la opción `-k` permite hacer la prueba sin que `curl` rechace el certificado por no pertenecer a una autoridad de confianza. Utilízala únicamente en entornos de prueba.

## Verificación

Puedes revisar el estado de todos los recursos con:

```bash
kubectl get deployment,pods,service,ingress -n default
kubectl describe ingress https-test-ingress -n default
```

Para comprobar la redirección desde HTTP:

```bash
curl -I http://mi-app.local
```

Para inspeccionar los logs de Nginx:

```bash
kubectl logs -l app=https-test -n default --tail=100
```

## Requisitos del certificado

El certificado que proporciones debe corresponder a su clave privada y debe incluir `DNS:mi-app.local` en la extensión **Subject Alternative Name (SAN)**. Los clientes modernos validan el SAN y pueden rechazar certificados que solamente contengan un nombre común (CN).

Puedes inspeccionar un certificado antes de crear el Secret con:

```bash
openssl x509 -in tls.crt -noout -subject -issuer -dates -ext subjectAltName
```

## Alcance actual de la gestión de certificados

En este momento el proceso es manual:

```text
El usuario obtiene o genera el certificado
                  │
                  ▼
      Lo guarda en un Secret TLS
                  │
                  ▼
         Traefik utiliza el Secret
```

El proyecto sabe almacenar y utilizar un certificado, pero no emitirlo ni renovarlo. Una evolución natural consiste en instalar **cert-manager** y definir un `Issuer` o `ClusterIssuer`. cert-manager podría solicitar o generar el certificado, guardarlo en el Secret y renovarlo antes de que expire.

## Eliminación

Para retirar los recursos creados:

```bash
kubectl delete -f app-with-https.yaml
kubectl delete secret mi-app-tls -n default
```

## Consideraciones para producción

Este ejemplo usa el namespace `default`, una imagen con etiqueta mutable (`nginx:alpine`) y un certificado local. Para producción se recomienda usar un namespace dedicado, fijar la versión o el digest de la imagen, automatizar la emisión y renovación de certificados, y adaptar la configuración del Ingress a la versión de Traefik instalada.
