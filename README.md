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



```bash
```

```bash
```

```bash
```

```bash
```

```bash
```

```bash
```

```bash
```

```bash
```

```bash
```