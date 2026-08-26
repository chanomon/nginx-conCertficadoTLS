# Aplicación de prueba con HTTPS en Kubernetes

Este proyecto despliega una aplicación Nginx sencilla en Kubernetes y la publica mediante HTTPS usando un `Ingress` de Traefik. Está pensado principalmente para pruebas locales con k3s y el dominio `mi-app.local`.

## Componentes

El manifiesto [`app-with-https.yaml`](./app-with-https.yaml) crea los siguientes recursos en el namespace `default`:

- **Deployment `https-test-app`**: ejecuta dos réplicas de `nginx:alpine`, con solicitudes y límites de CPU y memoria.
- **Service `https-test-service`**: expone internamente Nginx por el puerto 80.
- **Ingress `https-test-ingress`**: publica el servicio bajo `mi-app.local`, configura TLS y solicita a Traefik la redirección de HTTP a HTTPS.

El repositorio también contiene:

- `tls.crt`: certificado autofirmado para `mi-app.local`.
- `tls.key`: clave privada correspondiente al certificado.

> **Importante:** `tls.key` es material sensible. No debe compartirse ni almacenarse en un repositorio público. En un entorno real conviene administrar los certificados con cert-manager o con un gestor de secretos.

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

La opción `-k` es necesaria porque el certificado incluido es autofirmado y no pertenece a una autoridad de certificación de confianza.

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

## Información del certificado incluido

- Nombre común (CN): `mi-app.local`
- Emisor: autofirmado
- Vigencia: del 4 de julio de 2026 al 4 de julio de 2027 (UTC)
- La clave privada corresponde al certificado.

El certificado no contiene la extensión **Subject Alternative Name (SAN)**. Algunos clientes modernos pueden rechazarlo incluso si se instala como certificado de confianza. Para uso más allá de una prueba básica, genera un certificado nuevo que incluya `DNS:mi-app.local` en SAN.

## Eliminación

Para retirar los recursos creados:

```bash
kubectl delete -f app-with-https.yaml
kubectl delete secret mi-app-tls -n default
```

## Consideraciones para producción

Este ejemplo usa el namespace `default`, una imagen con etiqueta mutable (`nginx:alpine`) y un certificado local. Para producción se recomienda usar un namespace dedicado, fijar la versión o el digest de la imagen, automatizar la emisión y renovación de certificados, y adaptar la configuración del Ingress a la versión de Traefik instalada.
