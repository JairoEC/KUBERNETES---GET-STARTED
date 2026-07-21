# KUBERNETES---GET-STARTED
Pasos de instalacion inicial de Kubernetes

## 1. Actualización del sistema
1.1 instalar dependecias y paquetes de apt

`sudo apt update && sudo apt upgrade -y`

1.2 Herramientas basicas

    sudo apt install -y curl wget apt-transport-https ca-certificates gnupg lsb-release

## 2. Deshabilitar swap

2.1 Deshabilitar swap

`sudo swapoff -a`

2.2 Deshabilitar wap permanentemente

`sudo sed -i '/ swap / s/^/#/' /etc/fstab`

2.3 Verificar si swap está deshabilitado

`free -h`

## 3. Configuracion del kernel

3.1 Crear archivos de configuracion

    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

3.2 Cargar modulos de inmediato

    sudo modprobe overlay
    sudo modprobe br_netfilter

3.3 Verificar carga de modulos

    lsmod | grep overlay
    lsmod | grep br_netfilter

## 4. Configuracion de parametros

4.1 Crea un archivo de configuracion

    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward = 1
    EOF

4.2 Aplicar configuracion

    sudo sysctl --system

4.3 Verificar

    sudo sysctl net.bridge.bridge-nf-call-iptables \
    net.bridge.bridge-nf-call-ip6tables \
    net.ipv4.ip_forward

## 5. Configurar resolucion DNS (/et/hosts)

5.1 Agregar entradas de host

    cat <<EOF | sudo tee -a /etc/hosts
    192.168.18.2 master-k8s master
    192.168.18.3 worker-1 worker1
    192.168.18.4 worker-2 worker2
    EOF

5.2 Verificar

`cat /etc/hosts`

# INSTALACION DE CRI-O
**EJECUTAR EN TODOS LOS NODOS**
## 1. Definir variables de version

    export CRIO_VERSION=v1.31
    export OS=xUbuntu_24.04

## 2. Agregar repositorios CRI-O

2.1 Dependencias

    echo "deb [signed-by=/usr/share/keyrings/libcontainers-archive-keyring.gpg] \
    https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" \
    | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list

2.2 Repositorio de CRI-O

    echo "deb [signed-by=/usr/share/keyrings/libcontainers-crio-archive-keyring.gpg] \
    https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$CRIO_VERSION/$OS/ /" \
    | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$CRIO_VERSION.list

## 3. Importar claves keyring GPG

3.1 Crea carpeta/directorio para keyring

`sudo mkdir -p /etc/apt/keyrings`

3.2 Descargar keyring oficial de CRI-O

    curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

3.3 Agregar repositorio oficial

    echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

## 4. INSTALACION CRI-O

    # Actualizar repositorios
    sudo apt-get update
    # Instalar CRI-O y dependencias
    sudo apt-get install -y cri-o
    # Recargar systemd
    sudo systemctl daemon-reload
    # Habilitar CRI-O para inicio automático
    sudo systemctl enable crio
    # Iniciar CRI-O
    sudo systemctl start crio
    # Verificar estado
    sudo systemctl status crio

## 5. VERIFICACION DE CRIO

    # Ver versión de CRI-O
    sudo crio version
    # Ver información del runtime
    sudo crictl version
    sudo crictl info

    apt-cache policy cri-o
    # Verificar que el socket existe
    sudo ls -la /var/run/crio/crio.sock

# INSTALACION DE KUBERNETES

**EJECUTAR EN TODOS LOS NODOS**

Asegurar que existe la carpeta /etc/apt/keyrings

## 1. DESCARGAR KEYRING GPG DE KUBERNETES

    curl -4 -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
    | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

## 2. AGREGAR REPOSITORIO DE KUBERNETES

    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

## 3. INSTALAR DEPENDENCIAS DESCARGADAS DE KUBERNETES

    sudo apt install -y kubelet kubeadm kubectl

3.2 Retener la actualizacion de kubernetes

    sudo apt-mark hold kubelet kubeadm kubectl

3.3 Verifficar la instalacion

    dpkg -l | grep kubelet
    dpkg -l | grep kubeadm
    dpkg -l | grep kubectl

## 4. CONFIGURAR KUBELET PARA USAR CRI-O

    cat <<EOF | sudo tee /etc/default/kubelet
    KUBELET_EXTRA_ARGS="--container-runtime-endpoint=unix:///var/run/crio/crio.sock"
    EOF
    # Recargar systemd
    sudo systemctl daemon-reload
    # Reiniciar kubelet
    sudo systemctl restart kubelet
    # Verificar estado (estará en auto-restart hasta que se inicialice el cluster)
    sudo systemctl status kubelet

# SOLO MASTER

    cat <<EOF > kubeadm-config.yaml
    apiVersion: kubeadm.k8s.io/v1beta4
    kind: ClusterConfiguration

    kubernetesVersion: v1.31.14

    controlPlaneEndpoint: "master-k8s:6443"

    networking:
    podSubnet: "10.4.0.0/24"
    serviceSubnet: "10.96.0.0/12"


    ---
    apiVersion: kubeadm.k8s.io/v1beta4
    kind: InitConfiguration

    nodeRegistration:
    criSocket: "unix:///var/run/crio/crio.sock"
    EOF

## DMAS

    sudo apt install -y conntrack

    which conntrack

salida:

    /usr/sbin/conntrack

o

    /usr/bin/conntrack

## INCIALIZAR KUBERNETES

    sudo kubeadm init --config=kubeadm-config.yaml


## config de ctl

    #Crear directorio .kube
    mkdir -p $HOME/.kube
    # Copiar archivo de configuración
    sudo cp-i /etc/kubernetes/admin.conf $HOME/.kube/config
    # Cambiar permisos al usuario
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    # Verificar acceso al cluster
    kubectl get nodes
    # Ver información del cluster
    kubectl cluster-info

# ISNTALAR CILIUM

    # Obtener la última versión estable
    CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
    CLI_ARCH=amd64

    # Descargar Cilium CLI
    curl -L --fail --remote-name-all \
    https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

    # Verificar checksum
    sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum

    # Extraer e instalar
    sudo tar xzvf cilium-linux-${CLI_ARCH}.tar.gz -C /usr/local/bin

    # Limpiar
    rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

    # Verificar instalación
    cilium version --client