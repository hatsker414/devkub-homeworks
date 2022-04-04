# Домашнее задание к занятию "12.4 Развертывание кластера на собственных серверах, лекция 2"
Новые проекты пошли стабильным потоком. Каждый проект требует себе несколько кластеров: под тесты и продуктив. Делать все руками — не вариант, поэтому стоит автоматизировать подготовку новых кластеров.

## Задание 1: Подготовить инвентарь kubespray
Новые тестовые кластеры требуют типичных простых настроек. Нужно подготовить инвентарь и проверить его работу. Требования к инвентарю:
* подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды;
* в качестве CRI — containerd;
* запуск etcd производить на мастере.

## Решение задания 1: Подготовить инвентарь kubespray
##### Разварачивать будем в яндек облаке. для этого подготовим файл terraform который создаст 3 машины
```commandline
terraform {
  required_providers {
    yandex = {
      source = "terraform-registry.storage.yandexcloud.net/yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  token     = "token"
  cloud_id  = "cloud_id"
  folder_id = "folder_id"
  zone      = "ru-central1-a"
}
resource "yandex_compute_instance" "vm-1" {
  name = "control-node"

  resources {
    cores  = 4
    memory = 4
  }

  boot_disk {
    initialize_params {
      image_id = "fd81hgrcv6lsnkremf32"
      size = 20
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }

  metadata = {
    user-data = "${file("/home/alexp/yandex-cloud-terraform/meta.txt")}"
  }
}

resource "yandex_compute_instance" "vm-2" {
  name = "node1"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd81hgrcv6lsnkremf32"
      size = 20
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }


  metadata = {
    user-data = "${file("/home/alexp/yandex-cloud-terraform/meta.txt")}"
  }
}

resource "yandex_compute_instance" "vm-3" {
  name = "node2"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd81hgrcv6lsnkremf32"
      size = 20
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }


  metadata = {
    user-data = "${file("/home/alexp/yandex-cloud-terraform/meta.txt")}"
  }
}

resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.network-1.id
  v4_cidr_blocks = ["10.10.0.0/16"]
}

output "internal_ip_address_vm_1" {
  value = yandex_compute_instance.vm-1.network_interface.0.ip_address
}

output "internal_ip_address_vm_2" {
  value = yandex_compute_instance.vm-2.network_interface.0.ip_address
}

output "internal_ip_address_vm_3" {
  value = yandex_compute_instance.vm-3.network_interface.0.ip_address
}


output "external_ip_address_vm_1" {
  value = yandex_compute_instance.vm-1.network_interface.0.nat_ip_address
}

output "external_ip_address_vm_2" {
  value = yandex_compute_instance.vm-2.network_interface.0.nat_ip_address
}

output "external_ip_address_vm_3" {
  value = yandex_compute_instance.vm-3.network_interface.0.nat_ip_address
}
```
### Установка Kubespray
```commandline
alexp@lair :~$ git clone https://github.com/kubernetes-sigs/kubespray.git
```
Установка зависимостей для работы Kybespray 
```commandline
alexp@lair:~$ cd kubespray
alexp@lair:~/kubespray$ sudo pip3 install -r requirements.txt
```
Копируем инвентори из сэмпл в myclaster далее все изменения будем делать в my claster

```commandline
cp -rfp inventory/sample inventory/mycluster
```
с помощью билдера создадим host.yaml (необходимо указать ip виртуальных машин)

```commandline
declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```
пример host.yaml
```commandline
all:
  hosts:
    cp1:
      ansible_host: 51.250.73.106
      ansible_user: alexp
    node1:
      ansible_host: 51.250.65.163
      ansible_user: alexp
    node2:
      ansible_host: 51.250.73.88
      ansible_user: alexp
  children:
    kube_control_plane:
      hosts:
        cp1:
    kube_node:
      hosts:
        node1:
        node2:
    etcd:
      hosts:
        cp1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```
Для использования containerd в качестве CRI меняем слудующие параметры в файлах:
у меня по умолчанию уже стоял containerd и не нужно было править etcd.yaml
### k8s-cluster.yml
```commandline
container_manager: containerd
```
### etcd.yml
```commandline
etcd_deployment_type: host
```
### containerd.yml config
```commandline
containerd_registries:
  "docker.io": "https://registry-1.docker.io"  
```
Далее запускаем плейбук
```commandline
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b -v
```
![1](/img/dz_12_4_1.png)

подключимся к control node
```commandline
ssh alexp@51.250.73.106
alexp@cp1:~$ sudo kubectl get nodes
NAME    STATUS   ROLES                  AGE   VERSION
cp1     Ready    control-plane,master   55m   v1.23.5
node1   Ready    <none>                 53m   v1.23.5
node2   Ready    <none>                 53m   v1.23.5
```
![2](/img/dz_12_4_2.png)

задеплоим 2 реплики nginx
```commandline
sudo kubectl create deploy nginx --image=nginx:latest --replicas=2
```
затем посмотрим информацию о подах
```commandline
sudo kubectl get po -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
nginx-7c658794b9-27hsw   1/1     Running   0          26s   10.233.96.1   node2   <none>           <none>
nginx-7c658794b9-fj9x4   1/1     Running   0          26s   10.233.90.2   node1   <none>           <none>
```
![3](/img/dz_12_4_3.png)



## Задание 2 (*): подготовить и проверить инвентарь для кластера в AWS
Часть новых проектов хотят запускать на мощностях AWS. Требования похожи:
* разворачивать 5 нод: 1 мастер и 4 рабочие ноды;
* работать должны на минимально допустимых EC2 — t3.small.

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
