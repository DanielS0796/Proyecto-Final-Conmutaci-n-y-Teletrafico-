# Proyecto Final: Conmutación y Teletráfico
**Fundación Universitaria Compensar**  
**Materia:** Conmutación y Teletráfico  
**Docente:** Diego Alejandro Barragán Vargas  
**Estudiantes:**
## Daniel Felipe Guatibonza
## Yon Jaider Ruiz
## Oscar David Rada

---

## Descripción

Proyecto integrador que conecta conceptos de conmutación de redes con orquestación moderna de contenedores. Se construyó una red local con switch físico, servidor DHCP, monitoreo centralizado, contenedores Docker con servicios de inteligencia artificial y un servidor de juegos multijugador administrado con Kubernetes y Agones.

---

## Tabla de Contenidos

1. [Arquitectura](#arquitectura)
2. [Infraestructura](#infraestructura)
3. [DHCP](#dhcp)
4. [Contenedor YOLO](#contenedor-yolo)
5. [Contenedor Chatbot](#contenedor-chatbot)
6. [Contenedor Parrot OS](#contenedor-parrot-os)
7. [Kubernetes y SuperTuxKart](#kubernetes-y-supertuxkart)
8. [Monitoreo con Grafana y Prometheus](#monitoreo-con-grafana-y-prometheus)
9. [Comandos de arranque](#comandos-de-arranque)

---

## Arquitectura

La arquitectura fue implementada sobre tres nodos principales conectados a un switch físico:

- **PC Legion (Windows + WSL2):** Corre el clúster de Kubernetes con el servidor de juego SuperTuxKart mediante MicroK8s y Agones. IP fija `192.168.1.2`.
- **VM Administrador (Ubuntu en VMware):** Corre el servidor DHCP, Grafana, Prometheus, Node Exporter y cAdvisor. IP fija `192.168.1.13`.
- **VM Docker (Ubuntu en VMware):** Corre los tres contenedores principales: YOLO, Chatbot y Parrot OS. IP reservada por DHCP `192.168.1.17`.
- **PCs externos:** Se conectan al switch, reciben IP del rango `192.168.1.100–150` por DHCP y pueden conectarse al servidor de juego.

```
Switch Físico
├── PC Legion          192.168.1.2    → Kubernetes + SuperTuxKart
├── VM Administrador   192.168.1.13   → DHCP + Grafana + Prometheus
├── VM Docker          192.168.1.17   → YOLO + Chatbot + Parrot
└── PCs externos       192.168.1.100+ → Clientes del juego
```

> ![Arquitectura de red](<img width="1600" height="1020" alt="arquitectura_red" src="https://github.com/user-attachments/assets/d3351537-ae59-4bc0-a704-fa95c286f101" />
)

---

## Infraestructura

### PC Host (Legion)

| Componente | Detalle |
|---|---|
| Procesador | Intel Core i7-14700HX |
| RAM | 32 GB |
| Disco | 1 TB |
| Sistema operativo | Windows 11 + WSL2 Ubuntu |
| Virtualización | VMware Workstation |

Las dos máquinas virtuales fueron configuradas con adaptador de red en modo **Bridged** apuntando a la tarjeta Ethernet física (Realtek PCIe GbE), lo que les permitió integrarse a la misma red del switch físico.

> ![Configuración VMware Bridged](imagenes/vmware_bridged.png)

---

## DHCP

El servidor DHCP fue instalado en la VM Administrador usando `isc-dhcp-server`. Se configuró para operar sobre la interfaz `ens33` en la subred `192.168.1.0/24`.

### Instalación

```bash
sudo apt install -y isc-dhcp-server
```

### Configuración `/etc/dhcp/dhcpd.conf`

```conf
default-lease-time 600;
max-lease-time 7200;
authoritative;
log-facility local7;

subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.150;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.1.255;

  host vm-admin {
    hardware ethernet 00:0c:29:9b:dd:ee;
    fixed-address 192.168.1.13;
  }

  host vm-docker {
    hardware ethernet 00:0c:29:05:6c:8a;
    fixed-address 192.168.1.17;
  }
}
```

### Configuración del servicio systemd `/etc/systemd/system/`

El servicio fue habilitado para arrancar automáticamente:

```bash
sudo systemctl enable isc-dhcp-server
sudo systemctl start isc-dhcp-server
```

### Verificación

```bash
sudo journalctl -u isc-dhcp-server -f
```

> ![Logs DHCP con DHCPACK](imagenes/dhcp_logs.png)

> ![PC externo con IP asignada](imagenes/dhcp_cliente_ip.png)

---

## Contenedor YOLO

El contenedor YOLO fue creado a partir de la imagen `ultralytics/ultralytics:latest` y se usó para clasificar logos de herramientas tecnológicas.

### Herramientas y logos entrenados

Se recolectaron imágenes de 7 herramientas usando la librería `ddgs` y se etiquetaron en la plataforma Roboflow:

| Clase | Herramienta |
|---|---|
| `docker` | Docker |
| `kubernetes` | Kubernetes |
| `ansible` | Ansible |
| `jenkins` | Jenkins |
| `terraform` | Terraform |
| `podman` | Podman |
| `qemu` | QEMU |

### Dataset

| Parámetro | Valor |
|---|---|
| Imágenes originales | 390 |
| Imágenes con augmentación | 1,012 |
| Split | Train 92% / Valid 5% / Test 3% |
| Augmentaciones aplicadas | Flip horizontal, Rotación ±15°, Brillo ±25% |
| Plataforma de etiquetado | Roboflow |

### Descarga del dataset desde Roboflow

```python
from roboflow import Roboflow
rf = Roboflow(api_key="TU_API_KEY")
project = rf.workspace("tu-workspace").project("tu-proyecto")
version = project.version(1)
dataset = version.download("folder")
```

### Entrenamiento

```bash
yolo classify train \
  data=/usr/src/app/My-First-Project-1 \
  model=yolov8n-cls.pt \
  epochs=100 \
  imgsz=224 \
  batch=16 \
  name=logos_model_v2
```

### Resultados

```
Épocas:     100
Tiempo:     0.457 horas
top1_acc:   77.4%
top5_acc:   100%
```

### Predicción

```bash
yolo classify predict \
  model=/ultralytics/runs/classify/logos_model_v2/weights/best.pt \
  source=/ruta/imagen.jpg
```

> ![Roboflow dataset con 7 clases](imagenes/roboflow_dataset.png)

> ![Resultado predicción ansible 100%](imagenes/yolo_ansible.png)

> ![Resultado predicción terraform 100%](imagenes/yolo_terraform.png)

---

## Contenedor Chatbot

El chatbot fue construido con Flask y corre en el puerto 8000. Responde preguntas sobre el proyecto y las tecnologías utilizadas.

### Estructura del proyecto

```
/app/
├── app.py
├── Dockerfile
├── requirements.txt
└── data/
```

### `app.py`

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/mensaje', methods=['POST'])
def mensaje():
    data = request.get_json()
    texto = data.get("texto", "")
    return jsonify({"respuesta": f"Recibí tu mensaje: {texto}"})

@app.route('/upload', methods=['POST'])
def upload():
    file = request.files['file']
    file.save(f"/app/data/{file.filename}")
    return jsonify({"mensaje": f"Archivo {file.filename} guardado correctamente"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

### Prueba del endpoint

```bash
curl -X POST http://192.168.1.17:8000/mensaje \
  -H "Content-Type: application/json" \
  -d '{"texto": "¿Qué es Docker?"}'
```

> ![Chatbot respondiendo en el navegador](imagenes/chatbot_respuesta.png)

---

## Contenedor Parrot OS

El contenedor Parrot OS fue construido con un Dockerfile personalizado que incluye herramientas de auditoría de red.

### `Dockerfile`

```dockerfile
FROM parrotsec/core:latest

RUN apt update && apt install -y \
    nmap \
    tcpdump \
    net-tools \
    iputils-ping \
    curl \
    wget \
    && apt clean \
    && rm -rf /var/lib/apt/lists/*

CMD ["tail", "-f", "/dev/null"]
```

### Construcción e inicio

```bash
docker build -t parrot-custom ./parrot

docker run -d \
  --name parrot \
  --restart always \
  -p 2223:22 \
  --tty \
  -v ~/proyecto-containers/data:/data \
  parrot-custom
```

### Comandos de auditoría

```bash
# Entrar al contenedor
docker exec -it parrot bash

# Escanear dispositivos en la red
nmap -sn 192.168.1.0/24

# Ver puertos abiertos en el servidor
nmap -p 1-10000 192.168.1.2

# Capturar tráfico en tiempo real
tcpdump -i eth0 -n

# Capturar tráfico del juego
tcpdump -i eth0 port 7509

# Guardar captura en archivo
tcpdump -i eth0 -w /data/captura.pcap

# Analizar captura guardada
tcpdump -r /data/captura.pcap -n
```

> ![nmap escaneando la red](imagenes/parrot_nmap.png)

> ![tcpdump capturando tráfico del juego](imagenes/parrot_tcpdump.png)

---

## Kubernetes y SuperTuxKart

Kubernetes fue instalado mediante MicroK8s sobre WSL2 (Ubuntu) en el PC Legion. Agones fue instalado como orquestador de servidores de juego.

### Instalación de kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### Instalación de MicroK8s

```bash
sudo snap install microk8s --classic
sudo microk8s status --wait-ready
sudo microk8s enable dns dashboard registry helm3
sudo usermod -a -G microk8s $USER
newgrp microk8s
```

### Instalación de Agones

```bash
helm repo add agones https://agones.dev/chart/stable
helm repo update
helm upgrade --install agones agones/agones \
  --namespace agones-system \
  --create-namespace

kubectl get pods --namespace=agones-system
```

### Despliegue de SuperTuxKart con puerto fijo

```bash
cat << 'EOF' | microk8s kubectl apply -f -
apiVersion: agones.dev/v1
kind: Fleet
metadata:
  name: supertuxkart
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    spec:
      health:
        initialDelaySeconds: 30
        periodSeconds: 60
      ports:
      - containerPort: 8080
        hostPort: 7509
        name: default
        portPolicy: Static
      template:
        spec:
          containers:
          - image: us-docker.pkg.dev/agones-images/examples/supertuxkart-example:0.22
            name: supertuxkart
EOF
```

### Verificación

```bash
microk8s kubectl get gameservers
microk8s kubectl get pods --namespace=agones-system
```

### Conexión de jugadores

Los jugadores conectados al switch físico reciben IP por DHCP y se conectan desde SuperTuxKart:

```
Online → Enter server address → 192.168.1.2:7509
```

> ![kubectl get gameservers estado Ready](imagenes/k8s_gameservers.png)

> ![kubectl get pods agones-system](imagenes/k8s_agones_pods.png)

> ![SuperTuxKart conexión al servidor](imagenes/stk_conexion.png)

> ![Tres jugadores en carrera](imagenes/stk_partida.png)

---

## Monitoreo con Grafana y Prometheus

Grafana y Prometheus fueron instalados en la VM Administrador. cAdvisor fue desplegado como contenedor en ambas VMs para obtener métricas de los contenedores Docker. Node Exporter fue instalado como servicio en la VM Administrador y en WSL2.

### Servicios instalados

| Servicio | Puerto | Instalación |
|---|---|---|
| Grafana | 3000 | Paquete APT |
| Prometheus | 9090 | Binario descargado |
| Node Exporter | 9100 | Binario descargado |
| cAdvisor | 8080 | Contenedor Docker |

### Acceso

```
Grafana:    http://192.168.1.13:3000
Prometheus: http://192.168.1.13:9090
```

### Configuración `prometheus.yml`

```yaml
rule_files:
  - "/home/administrador/monitoring/prometheus-2.52.0.linux-amd64/alert.rules.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["localhost:9100", "192.168.1.2:9100"]

  - job_name: "docker_containers"
    static_configs:
      - targets: ["127.0.0.1:9323"]

  - job_name: "cadvisor"
    static_configs:
      - targets: ["localhost:8080"]

  - job_name: "cadvisor_docker_vm"
    static_configs:
      - targets: ["192.168.1.17:8080"]
```

### Servicios systemd creados

```bash
# /etc/systemd/system/prometheus.service
# /etc/systemd/system/node-exporter.service
# /etc/systemd/system/grafana-server.service

sudo systemctl enable prometheus node-exporter grafana-server
sudo systemctl start prometheus node-exporter grafana-server
```

### Dashboards importados en Grafana

| Dashboard | ID | Descripción |
|---|---|---|
| Node Exporter Full | 1860 | CPU, RAM, disco y red de cada nodo |
| cAdvisor | 19792 | Métricas detalladas de contenedores Docker |

> ![Prometheus targets todos UP](imagenes/prometheus_targets.png)

> ![Grafana Node Exporter Full](imagenes/grafana_node_exporter.png)

> ![Grafana cAdvisor contenedores](imagenes/grafana_cadvisor.png)

---

## Comandos de arranque

### PC Legion - Script de inicio automático (Windows Startup)

```batch
@echo off
netsh interface portproxy add v4tov4 listenaddress=192.168.1.2 listenport=9100 connectaddress=172.19.156.28 connectport=9100
netsh interface portproxy add v4tov4 listenaddress=192.168.1.2 listenport=9090 connectaddress=172.19.156.28 connectport=9090
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=7509 connectaddress=172.19.156.28 connectport=7509
wsl -d Ubuntu
```

### WSL2 - Node Exporter automático vía `.bashrc`

```bash
if ! pgrep -x "node_exporter" > /dev/null; then
    node_exporter > /dev/null 2>&1 &
fi
```

### VM Administrador - Verificar servicios

```bash
sudo nmcli connection up "Wired connection 1"
sudo systemctl status grafana-server prometheus node-exporter isc-dhcp-server --no-pager
```

### VM Docker - Verificar contenedores

```bash
docker ps
```

### Kubernetes - Verificar servidor del juego

```bash
microk8s kubectl get gameservers
```

---

## Referencias

- https://agones.dev/site/docs/
- https://microk8s.io/docs
- https://docs.ultralytics.com/
- https://grafana.com/docs/
- https://prometheus.io/docs/
- https://supertuxkart.net/
- https://roboflow.com/
- https://www.isc.org/dhcp/
- https://www.parrotsec.org/
