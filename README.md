Aufgabe: Erstelle ein benutzerdefiniertes Bridge-Netzwerk mit dem Namen my_bridge_network.

Starte zwei Container (container1 und container2), die mit diesem Netzwerk verbunden sind.

Installiere ping in beiden Containern und teste, ob sie sich gegenseitig erreichen können.



Lösung A:

docker network create --driver bridge my_bridge_network

docker run -dit --name container1 --network my_bridge_network alpine
docker run -dit --name container2 --network my_bridge_network alpine

for container in container1 container2; do
  docker exec -it $container sh -c "apk update && apk add --no-cache iputils"
done

echo ""
docker exec -it container1 ping -c 4 container2
echo ""
docker exec -it container2 ping -c 4 container1
echo ""

Terminal Ausgabe:

Markus@WKS03:~$ for container in container1 container2; do
  docker exec -it $container sh -c "apk update && apk add --no-cache iputils"
done

fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/community/x86_64/APKINDEX.tar.gz
v3.21.3-144-gc9b177afb6a [https://dl-cdn.alpinelinux.org/alpine/v3.21/main]
v3.21.3-151-gdb4964b88a3 [https://dl-cdn.alpinelinux.org/alpine/v3.21/community]
OK: 25395 distinct packages available
fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/community/x86_64/APKINDEX.tar.gz
(1/6) Installing libcap2 (2.71-r0)
(2/6) Installing iputils-arping (20240905-r0)
(3/6) Installing iputils-clockdiff (20240905-r0)
(4/6) Installing iputils-ping (20240905-r0)
(5/6) Installing iputils-tracepath (20240905-r0)
(6/6) Installing iputils (20240905-r0)
Executing busybox-1.37.0-r12.trigger
OK: 7 MiB in 21 packages
fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/community/x86_64/APKINDEX.tar.gz
v3.21.3-144-gc9b177afb6a [https://dl-cdn.alpinelinux.org/alpine/v3.21/main]
v3.21.3-151-gdb4964b88a3 [https://dl-cdn.alpinelinux.org/alpine/v3.21/community]
OK: 25395 distinct packages available
fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/community/x86_64/APKINDEX.tar.gz
(1/6) Installing libcap2 (2.71-r0)
(2/6) Installing iputils-arping (20240905-r0)
(3/6) Installing iputils-clockdiff (20240905-r0)
(4/6) Installing iputils-ping (20240905-r0)
(5/6) Installing iputils-tracepath (20240905-r0)
(6/6) Installing iputils (20240905-r0)
Executing busybox-1.37.0-r12.trigger
OK: 7 MiB in 21 packages

markus@WKS03:~$ docker exec -it container1 ping -c 4 container2
PING container2 (172.20.0.3) 56(84) bytes of data.
64 bytes from container2.my_bridge_network (172.20.0.3): icmp_seq=1 ttl=64 time=0.274 ms
64 bytes from container2.my_bridge_network (172.20.0.3): icmp_seq=2 ttl=64 time=0.102 ms
64 bytes from container2.my_bridge_network (172.20.0.3): icmp_seq=3 ttl=64 time=0.119 ms
64 bytes from container2.my_bridge_network (172.20.0.3): icmp_seq=4 ttl=64 time=0.051 ms

--- container2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3491ms
rtt min/avg/max/mdev = 0.051/0.136/0.274/0.083 ms

markus@WKS03:~$ docker exec -it container2 ping -c 4 container1
PING container1 (172.20.0.2) 56(84) bytes of data.
64 bytes from container1.my_bridge_network (172.20.0.2): icmp_seq=1 ttl=64 time=0.059 ms
64 bytes from container1.my_bridge_network (172.20.0.2): icmp_seq=2 ttl=64 time=0.137 ms
64 bytes from container1.my_bridge_network (172.20.0.2): icmp_seq=3 ttl=64 time=0.085 ms
64 bytes from container1.my_bridge_network (172.20.0.2): icmp_seq=4 ttl=64 time=0.065 ms

--- container1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3450ms
rtt min/avg/max/mdev = 0.059/0.086/0.137/0.030 ms

----------------------------------------------------------------------------------------

Lösung B: Compose Stack und Healthcheck (mit container 10 und 20) :

sudo apt install docker-compose -y
sudo mkdir pingtest && cd pingtest

sudo bash -c 'echo "services:
  container10:
    image: alpine
    container_name: container10
    command: sh -c \"apk update && apk add --no-cache iputils && tail -f /dev/null\"
    healthcheck:
      test: [\"CMD\", \"ping\", \"-c\", \"1\", \"container20\"]
      interval: 30s
      timeout: 10s
      retries: 2
    networks:
      - my_bridge_network

  container20:
    image: alpine
    container_name: container20
    command: sh -c \"apk update && apk add --no-cache iputils && tail -f /dev/null\"
    healthcheck:
      test: [\"CMD\", \"ping\", \"-c\", \"1\", \"container10\"]
      interval: 30s
      timeout: 10s
      retries: 2
    networks:
      - my_bridge_network

networks:
  my_bridge_network:
    driver: bridge
" > docker-compose.yaml'

docker-compose up -d

echo "Ping und Healthcheck werden durchgeführt..."
echo ""
docker exec -it container10 ping -c 4 container20
echo ""
docker exec -it container20 ping -c 4 container10
echo""
sleep 30
echo "Container10 is $(docker inspect --format='{{.State.Health.Status}}' container10)"
echo "Container20 is $(docker inspect --format='{{.State.Health.Status}}' container20)"
echo ""
echo "all done"

Terminal Ausgabe:

markus@WKS03:/pingtest$ docker-compose up -d

Creating network "pingtest_my_bridge_network" with driver "bridge"
Creating container20 ... done
Creating container10 ... done
Container werden gestartet

PING container20 (172.30.0.3): 56 data bytes
64 bytes from 172.30.0.3: seq=0 ttl=64 time=0.075 ms
64 bytes from 172.30.0.3: seq=1 ttl=64 time=0.052 ms
64 bytes from 172.30.0.3: seq=2 ttl=64 time=0.082 ms
64 bytes from 172.30.0.3: seq=3 ttl=64 time=0.134 ms

--- container20 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.052/0.085/0.134 ms

PING container10 (172.30.0.2) 56(84) bytes of data.
64 bytes from container10.pingtest_my_bridge_network (172.30.0.2): icmp_seq=1 ttl=64 time=0.157 ms
64 bytes from container10.pingtest_my_bridge_network (172.30.0.2): icmp_seq=2 ttl=64 time=0.100 ms
64 bytes from container10.pingtest_my_bridge_network (172.30.0.2): icmp_seq=3 ttl=64 time=0.126 ms
64 bytes from container10.pingtest_my_bridge_network (172.30.0.2): icmp_seq=4 ttl=64 time=0.096 ms

--- container10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3380ms
rtt min/avg/max/mdev = 0.096/0.119/0.157/0.024 ms

Container10 ist healthy
Container20 ist healthy

all done

