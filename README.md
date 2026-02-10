<h1>Swarm local<h1/>

## Preparación

```bash
docker version
docker info | grep -i swarm
```

## Si ya estás en Swarm, sal:
```bash
docker swarm leave --force
```
## si usas Dind (docker in docker)

a) publicar el puerto del mgr hacia tu PC

Recrea el manager así:

1️⃣ Créalo con port-forward

```bash
docker run -d --privileged \
  -p 8080:8080 \
  --name mgr \
  --hostname mgr \
  --network swarm-net \
  docker:24-dind
```

## 1) Crear un Swarm “multi-nodo” en tu máquina con Docker-in-Docker

Vamos a simular 1 manager + 2 workers como contenedores (muy útil en local).

1.1 Crear red para el “cluster”

```bash
docker network create --driver bridge swarm-net
```

1.2 Crear 3 nodos (DinD)

```bash
°°
(solo Dind)
docker run -d --privileged --name mgr --hostname mgr --network swarm-net -p 8080:8080 docker:24-dind

°°
(swarm normal)
docker run -d --privileged --name mgr --hostname mgr --network swarm-net docker:24-dind 

°°
(aplica para ambos)
docker run -d --privileged --name w1  --hostname w1  --network swarm-net docker:24-dind
docker run -d --privileged --name w2  --hostname w2  --network swarm-net docker:24-dind
```

1.3 Instalar cliente docker dentro de cada nodo (rápido)

```bash
docker exec mgr sh -lc "apk add --no-cache docker-cli"
docker exec w1  sh -lc "apk add --no-cache docker-cli"
docker exec w2  sh -lc "apk add --no-cache docker-cli"
```

## 2) Inicializar Swarm (AUTOLock + Quorum)

2.1 Inicializa Swarm en el manager

```bash
docker exec mgr sh -lc "docker swarm init --autolock --advertise-addr mgr"

°°

docker exec mgr sh -lc "docker swarm init --autolock --advertise-addr eth0"

```

2.2 Obtener token de worker y unir workers

```bash
WORKER_TOKEN=$(docker exec mgr sh -lc "docker swarm join-token -q worker" | tr -d '\r')
echo $WORKER_TOKEN

docker exec w1 sh -lc "docker swarm join --token $WORKER_TOKEN mgr:2377"
docker exec w2 sh -lc "docker swarm join --token $WORKER_TOKEN mgr:2377"
```

2.3 Ver nodos

```bash
docker exec mgr sh -lc "docker node ls"
```
✅ Aprendizaje clave: manager controla el estado, workers ejecutan tasks.

## 3) Crear overlay network + servicio replicated

3.1 Crear red overlay

```bash
docker exec mgr sh -lc "docker network create -d overlay app-net"
```

3.2 Crear servicio web (replicated) + publish ingress

```bash
docker exec mgr sh -lc "docker service create --name web --replicas 3 --network app-net -p 8080:80 nginx:alpine"
```

3.3 Ver estado

```bash
docker exec mgr sh -lc "docker service ls"
docker exec mgr sh -lc "docker service ps web"
```

3.4 Probar acceso (desde tu máquina)

Abre en navegador: http://localhost:8080

✅ Aprendizaje clave: publish en ingress expone el puerto “en todos los nodos” (routing mesh).


## 4) Escalar, actualizar, rollback

4.1 Escalar a 5

```bash
docker exec mgr sh -lc "docker service scale web=5"
docker exec mgr sh -lc "docker service ps web"
```

4.2 Update (cambiar imagen) + observar rollout

```bash
docker exec mgr sh -lc "docker service update --image nginx:1.25-alpine web"
docker exec mgr sh -lc "docker service ps web"
```

4.3 Forzar un rollback (opcional)

```bash
docker exec mgr sh -lc "docker service rollback web"
```
✅ Aprendizaje clave: update/rollback y cómo Swarm reconcilia estado.

## 5) Global service (1 por nodo)

```bash
docker exec mgr sh -lc "docker service create --name agent --mode global alpine:3.20 sleep 36000"
docker exec mgr sh -lc "docker service ps agent"
```
✅ Aprendizaje clave: global = 1 task por nodo. (ojo son 2 servicios adentro de este Dind) (web y agent)

## 6) Labels + placement constraints

6.1 Etiquetar un nodo worker

```bash
docker exec mgr sh -lc "docker node update --label-add disk=ssd w1"
```

6.2 Servicio que solo corre en SSD

```bash
docker exec mgr sh -lc "docker service create --name ssd-only --constraint 'node.labels.disk==ssd' alpine:3.20 sleep 36000"
docker exec mgr sh -lc "docker service ps ssd-only"
```
✅ Aprendizaje clave: labels + constraints controlan placement.

## 7) Volúmenes (persistencia) y limitación en Swarm local

Crea un servicio con volumen local (notarás que es local al nodo):

```bash
docker exec mgr sh -lc "docker service create --name data --replicas 1 --mount type=volume,source=myvol,target=/data alpine:3.20 sh -lc 'echo hola > /data/msg && sleep 36000'"

docker exec mgr sh -lc "docker service ps data"
```
✅ Aprendizaje clave: volúmenes en Swarm por defecto son locales, para “shared storage” necesitas NFS/EFS/CEPH/Portworx, etc.

## 8) Stack deploy (compose → stack)

8.1 Crear archivo stack.yml en tu máquina (host)

(compartido en el repositorio)

8.2 Desplegar stack en el manager

Copia el archivo al contenedor manager y despliega:

```bash
docker cp stack.yml mgr:/stack.yml
docker exec mgr sh -lc "docker stack deploy -c /stack.yml demo"
docker exec mgr sh -lc "docker stack services demo"
docker exec mgr sh -lc "docker stack ps demo"
```
Prueba: (solo dentro del docker no Dind)

http://localhost:8090

✅ Aprendizaje clave: docker stack deploy = Swarm.

## 9) Troubleshooting real (simular fallo)

9.1 Crear un servicio con imagen inválida

```bash
docker exec mgr sh -lc "docker service create --name broken --replicas 2 noexiste/imagen:latest"

"CTRL + C"
```

9.2 Diagnóstico (lo que espera el examen)

```bash
docker exec mgr sh -lc "docker service ps broken"
docker exec mgr sh -lc "docker service inspect broken --pretty"
```
✅ Aprendizaje clave: para troubleshooting siempre empezar con docker service ps.

## 10) Limpieza total del lab

```bash
docker exec mgr sh -lc "docker stack rm demo"
docker exec mgr sh -lc "docker service rm web agent ssd-only data broken || true"
docker rm -f mgr w1 w2
docker network rm swarm-net
```
