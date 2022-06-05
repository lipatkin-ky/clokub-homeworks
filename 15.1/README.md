# Домашнее задание к занятию "15.1. Организация сети"

Домашнее задание будет состоять из обязательной части, которую необходимо выполнить на провайдере Яндекс.Облако и дополнительной части в AWS по желанию. Все домашние задания в 15 блоке связаны друг с другом и в конце представляют пример законченной инфраструктуры.  
Все задания требуется выполнить с помощью Terraform, результатом выполненного домашнего задания будет код в репозитории. 

Перед началом работ следует настроить доступ до облачных ресурсов из Terraform используя материалы прошлых лекций и [ДЗ](https://github.com/netology-code/virt-homeworks/tree/master/07-terraform-02-syntax ). А также заранее выбрать регион (в случае AWS) и зону.

---
## Задание 1. Яндекс.Облако (обязательное к выполнению)

1. Создать VPC.
- Создать пустую VPC. Выбрать зону.
2. Публичная подсеть.
- Создать в vpc subnet с названием public, сетью 192.168.10.0/24.
- Создать в этой подсети NAT-инстанс, присвоив ему адрес 192.168.10.254. В качестве image_id использовать fd80mrhj8fl2oe87o4e1
- Создать в этой публичной подсети виртуалку с публичным IP и подключиться к ней, убедиться что есть доступ к интернету.
3. Приватная подсеть.
- Создать в vpc subnet с названием private, сетью 192.168.20.0/24.
- Создать route table. Добавить статический маршрут, направляющий весь исходящий трафик private сети в NAT-инстанс
- Создать в этой приватной подсети виртуалку с внутренним IP, подключиться к ней через виртуалку, созданную ранее и убедиться что есть доступ к интернету

Resource terraform для ЯО
- [VPC subnet](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/vpc_subnet)
- [Route table](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/vpc_route_table)
- [Compute Instance](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/compute_instance)
---
```
% terraform -v
Terraform v1.2.2
on darwin_amd64
+ provider registry.terraform.io/yandex-cloud/yandex v0.75.0
```
```
% ls -l
total 64
-rw-r--r--  1 qos  staff    282 Jun  4 19:47 main.tf
-rw-r--r--  1 qos  staff  14226 Jun  5 22:46 terraform.tfstate
-rw-r--r--  1 qos  staff    156 Jun  5 22:45 terraform.tfstate.backup
-rw-r--r--  1 qos  staff   1455 Jun  5 22:28 vm-netology.tf
-rw-r--r--  1 qos  staff    850 Jun  5 22:45 vpc-netology.tf
```
```
% more main.tf 
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}

provider "yandex" {
  token     = "xxx"
  cloud_id  = "xxx"
  folder_id = "xxx"
  zone      = "ru-central1-b"
}
```
```
% more vpc-netology.tf 

resource "yandex_vpc_network" "vpc-netology" {
  name = "vpc-netology"
}

resource "yandex_vpc_subnet" "public" {
  name           = "public"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.vpc-netology.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_vpc_subnet" "private" {
  name           = "private"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.vpc-netology.id
  v4_cidr_blocks = ["192.168.20.0/24"]
  route_table_id = yandex_vpc_route_table.rtb-netology.id
}

resource "yandex_vpc_route_table" "rtb-netology" {
  network_id     = yandex_vpc_network.vpc-netology.id

  static_route {
    destination_prefix     = "0.0.0.0/0"
    next_hop_address       = "192.168.10.254"
    ### next_hop_address   = yandex_compute_instance.tr-public-nat.network_interface[0].ip_address
  }
}
```
```
% more vm-netology.tf 
### VM-public

resource "yandex_compute_instance" "tr-public-vm01" {
  name = "tr-public-vm01"
  platform_id = "standard-v1"
  zone = "ru-central1-a"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd89ovh4ticpo40dkbvd"
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.public.id
    nat       = true
  }

  metadata = {
    ssh-keys = "ubuntu:${file("~/temp/identity.pub")}"
  }
}

### NAT-instance

resource "yandex_compute_instance" "tr-public-nat" {
  name = "tr-public-nat"
  platform_id = "standard-v1"
  zone = "ru-central1-a"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd80mrhj8fl2oe87o4e1"
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.public.id
    ip_address = "192.168.10.254"
    nat       = true
  }

  metadata = {
    ssh-keys = "ubuntu:${file("~/temp/identity.pub")}"
  }
}

### VM-private

resource "yandex_compute_instance" "tr-private-vm01" {
  name = "tr-private-vm01"
  platform_id = "standard-v1"
  zone = "ru-central1-a"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd89ovh4ticpo40dkbvd"
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.private.id
    nat       = false
  }

  metadata = {
    ssh-keys = "ubuntu:${file("~/temp/identity.pub")}"
  }
}
```
```
% yc compute instance list   
+----------------------+--------+---------------+---------+----------------+-------------+
|          ID          |  NAME  |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP |
+----------------------+--------+---------------+---------+----------------+-------------+
| epd4ha89otbi6jtjd1be | k8s-01 | ru-central1-b | STOPPED | 51.250.107.254 | 10.129.0.20 |
| epdnd78l3e8lfj4cs334 | k8s-02 | ru-central1-b | STOPPED | 51.250.106.21  | 10.129.0.13 |
| epdo00prvipcbdu79f5r | k8s-04 | ru-central1-b | STOPPED |                | 10.129.0.34 |
| epdvbkthc2g96vd8pbk1 | k8s-05 | ru-central1-b | STOPPED |                | 10.129.0.38 |
| epdvm4mcq1903dp0ql8c | k8s-03 | ru-central1-b | STOPPED |                | 10.129.0.24 |
+----------------------+--------+---------------+---------+----------------+-------------+
```
```
% terraform apply         

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # yandex_compute_instance.tr-private-vm01 will be created
  + resource "yandex_compute_instance" "tr-private-vm01" {
      + created_at                = (known after apply)
      + folder_id                 = (known after apply)
      + fqdn                      = (known after apply)
      + hostname                  = (known after apply)
      + id                        = (known after apply)
      + metadata                  = {
          + "ssh-keys" = "ubuntu:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINwYn4yqIzg25NyJAnCwb3WgnO5UXwa3hpz78uSWiQYR 202205-Li-ED"
        }
      + name                      = "tr-private-vm01"
      + network_acceleration_type = "standard"
      + platform_id               = "standard-v1"
      + service_account_id        = (known after apply)
      + status                    = (known after apply)
      + zone                      = "ru-central1-a"

      + boot_disk {
          + auto_delete = true
          + device_name = (known after apply)
          + disk_id     = (known after apply)
          + mode        = (known after apply)

          + initialize_params {
              + block_size  = (known after apply)
              + description = (known after apply)
              + image_id    = "fd89ovh4ticpo40dkbvd"
              + name        = (known after apply)
              + size        = (known after apply)
              + snapshot_id = (known after apply)
              + type        = "network-hdd"
            }
        }

      + network_interface {
          + index              = (known after apply)
          + ip_address         = (known after apply)
          + ipv4               = true
          + ipv6               = (known after apply)
          + ipv6_address       = (known after apply)
          + mac_address        = (known after apply)
          + nat                = false
          + nat_ip_address     = (known after apply)
          + nat_ip_version     = (known after apply)
          + security_group_ids = (known after apply)
          + subnet_id          = (known after apply)
        }

      + placement_policy {
          + host_affinity_rules = (known after apply)
          + placement_group_id  = (known after apply)
        }

      + resources {
          + core_fraction = 100
          + cores         = 2
          + memory        = 2
        }

      + scheduling_policy {
          + preemptible = (known after apply)
        }
    }

  # yandex_compute_instance.tr-public-nat will be created
  + resource "yandex_compute_instance" "tr-public-nat" {
      + created_at                = (known after apply)
      + folder_id                 = (known after apply)
      + fqdn                      = (known after apply)
      + hostname                  = (known after apply)
      + id                        = (known after apply)
      + metadata                  = {
          + "ssh-keys" = "ubuntu:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINwYn4yqIzg25NyJAnCwb3WgnO5UXwa3hpz78uSWiQYR 202205-Li-ED"
        }
      + name                      = "tr-public-nat"
      + network_acceleration_type = "standard"
      + platform_id               = "standard-v1"
      + service_account_id        = (known after apply)
      + status                    = (known after apply)
      + zone                      = "ru-central1-a"

      + boot_disk {
          + auto_delete = true
          + device_name = (known after apply)
          + disk_id     = (known after apply)
          + mode        = (known after apply)

          + initialize_params {
              + block_size  = (known after apply)
              + description = (known after apply)
              + image_id    = "fd80mrhj8fl2oe87o4e1"
              + name        = (known after apply)
              + size        = (known after apply)
              + snapshot_id = (known after apply)
              + type        = "network-hdd"
            }
        }

      + network_interface {
          + index              = (known after apply)
          + ip_address         = "192.168.10.254"
          + ipv4               = true
          + ipv6               = (known after apply)
          + ipv6_address       = (known after apply)
          + mac_address        = (known after apply)
          + nat                = true
          + nat_ip_address     = (known after apply)
          + nat_ip_version     = (known after apply)
          + security_group_ids = (known after apply)
          + subnet_id          = (known after apply)
        }

      + placement_policy {
          + host_affinity_rules = (known after apply)
          + placement_group_id  = (known after apply)
        }

      + resources {
          + core_fraction = 100
          + cores         = 2
          + memory        = 2
        }

      + scheduling_policy {
          + preemptible = (known after apply)
        }
    }

  # yandex_compute_instance.tr-public-vm01 will be created
  + resource "yandex_compute_instance" "tr-public-vm01" {
      + created_at                = (known after apply)
      + folder_id                 = (known after apply)
      + fqdn                      = (known after apply)
      + hostname                  = (known after apply)
      + id                        = (known after apply)
      + metadata                  = {
          + "ssh-keys" = "ubuntu:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINwYn4yqIzg25NyJAnCwb3WgnO5UXwa3hpz78uSWiQYR 202205-Li-ED"
        }
      + name                      = "tr-public-vm01"
      + network_acceleration_type = "standard"
      + platform_id               = "standard-v1"
      + service_account_id        = (known after apply)
      + status                    = (known after apply)
      + zone                      = "ru-central1-a"

      + boot_disk {
          + auto_delete = true
          + device_name = (known after apply)
          + disk_id     = (known after apply)
          + mode        = (known after apply)

          + initialize_params {
              + block_size  = (known after apply)
              + description = (known after apply)
              + image_id    = "fd89ovh4ticpo40dkbvd"
              + name        = (known after apply)
              + size        = (known after apply)
              + snapshot_id = (known after apply)
              + type        = "network-hdd"
            }
        }

      + network_interface {
          + index              = (known after apply)
          + ip_address         = (known after apply)
          + ipv4               = true
          + ipv6               = (known after apply)
          + ipv6_address       = (known after apply)
          + mac_address        = (known after apply)
          + nat                = true
          + nat_ip_address     = (known after apply)
          + nat_ip_version     = (known after apply)
          + security_group_ids = (known after apply)
          + subnet_id          = (known after apply)
        }

      + placement_policy {
          + host_affinity_rules = (known after apply)
          + placement_group_id  = (known after apply)
        }

      + resources {
          + core_fraction = 100
          + cores         = 2
          + memory        = 2
        }

      + scheduling_policy {
          + preemptible = (known after apply)
        }
    }

  # yandex_vpc_network.vpc-netology will be created
  + resource "yandex_vpc_network" "vpc-netology" {
      + created_at                = (known after apply)
      + default_security_group_id = (known after apply)
      + folder_id                 = (known after apply)
      + id                        = (known after apply)
      + labels                    = (known after apply)
      + name                      = "vpc-netology"
      + subnet_ids                = (known after apply)
    }

  # yandex_vpc_route_table.rtb-netology will be created
  + resource "yandex_vpc_route_table" "rtb-netology" {
      + created_at = (known after apply)
      + folder_id  = (known after apply)
      + id         = (known after apply)
      + labels     = (known after apply)
      + network_id = (known after apply)

      + static_route {
          + destination_prefix = "0.0.0.0/0"
          + next_hop_address   = "192.168.10.254"
        }
    }

  # yandex_vpc_subnet.private will be created
  + resource "yandex_vpc_subnet" "private" {
      + created_at     = (known after apply)
      + folder_id      = (known after apply)
      + id             = (known after apply)
      + labels         = (known after apply)
      + name           = "private"
      + network_id     = (known after apply)
      + route_table_id = (known after apply)
      + v4_cidr_blocks = [
          + "192.168.20.0/24",
        ]
      + v6_cidr_blocks = (known after apply)
      + zone           = "ru-central1-a"
    }

  # yandex_vpc_subnet.public will be created
  + resource "yandex_vpc_subnet" "public" {
      + created_at     = (known after apply)
      + folder_id      = (known after apply)
      + id             = (known after apply)
      + labels         = (known after apply)
      + name           = "public"
      + network_id     = (known after apply)
      + v4_cidr_blocks = [
          + "192.168.10.0/24",
        ]
      + v6_cidr_blocks = (known after apply)
      + zone           = "ru-central1-a"
    }

Plan: 7 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

yandex_vpc_network.vpc-netology: Creating...
yandex_vpc_network.vpc-netology: Creation complete after 0s [id=enp915qoj53b096rvngq]
yandex_vpc_route_table.rtb-netology: Creating...
yandex_vpc_subnet.public: Creating...
yandex_vpc_subnet.public: Creation complete after 1s [id=e9bmf2bqj36nsgc6gmm1]
yandex_compute_instance.tr-public-vm01: Creating...
yandex_compute_instance.tr-public-nat: Creating...
yandex_vpc_route_table.rtb-netology: Creation complete after 2s [id=enp1q1qt0m357m7q1m6j]
yandex_vpc_subnet.private: Creating...
yandex_vpc_subnet.private: Creation complete after 0s [id=e9btulum1a3rkdu9klhm]
yandex_compute_instance.tr-private-vm01: Creating...
yandex_compute_instance.tr-public-nat: Still creating... [10s elapsed]
yandex_compute_instance.tr-public-vm01: Still creating... [10s elapsed]
yandex_compute_instance.tr-private-vm01: Still creating... [10s elapsed]
yandex_compute_instance.tr-public-vm01: Still creating... [20s elapsed]
yandex_compute_instance.tr-public-nat: Still creating... [20s elapsed]
yandex_compute_instance.tr-private-vm01: Still creating... [20s elapsed]
yandex_compute_instance.tr-private-vm01: Creation complete after 21s [id=fhm5jrf4ibc9cdvv1iek]
yandex_compute_instance.tr-public-vm01: Creation complete after 25s [id=fhmqn5r9a9in24050koh]
yandex_compute_instance.tr-public-nat: Still creating... [30s elapsed]
yandex_compute_instance.tr-public-nat: Still creating... [40s elapsed]
yandex_compute_instance.tr-public-nat: Creation complete after 46s [id=fhmamofkshv2f4epeija]

Apply complete! Resources: 7 added, 0 changed, 0 destroyed.
```
```
% yc compute instance list
+----------------------+-----------------+---------------+---------+----------------+----------------+
|          ID          |      NAME       |    ZONE ID    | STATUS  |  EXTERNAL IP   |  INTERNAL IP   |
+----------------------+-----------------+---------------+---------+----------------+----------------+
| epd4ha89otbi6jtjd1be | k8s-01          | ru-central1-b | STOPPED | 51.250.107.254 | 10.129.0.20    |
| epdnd78l3e8lfj4cs334 | k8s-02          | ru-central1-b | STOPPED | 51.250.106.21  | 10.129.0.13    |
| epdo00prvipcbdu79f5r | k8s-04          | ru-central1-b | STOPPED |                | 10.129.0.34    |
| epdvbkthc2g96vd8pbk1 | k8s-05          | ru-central1-b | STOPPED |                | 10.129.0.38    |
| epdvm4mcq1903dp0ql8c | k8s-03          | ru-central1-b | STOPPED |                | 10.129.0.24    |
| fhm5jrf4ibc9cdvv1iek | tr-private-vm01 | ru-central1-a | RUNNING |                | 192.168.20.8   |
| fhmamofkshv2f4epeija | tr-public-nat   | ru-central1-a | RUNNING | 51.250.88.109  | 192.168.10.254 |
| fhmqn5r9a9in24050koh | tr-public-vm01  | ru-central1-a | RUNNING | 51.250.78.99   | 192.168.10.25  |
+----------------------+-----------------+---------------+---------+----------------+----------------+
```
```
% ssh -A ubuntu@51.250.78.99 
The authenticity of host '51.250.78.99 (51.250.78.99)' can't be established.
ED25519 key fingerprint is SHA256:Z006jsnLAebBEvm4So8L5WcxHrCh/T/Dq2uQFUyj4xI.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '51.250.78.99' (ED25519) to the list of known hosts.
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-113-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.



ubuntu@fhmqn5r9a9in24050koh:~$ ping 192.168.20.8 
PING 192.168.20.8 (192.168.20.8) 56(84) bytes of data.
64 bytes from 192.168.20.8: icmp_seq=1 ttl=63 time=1.38 ms
64 bytes from 192.168.20.8: icmp_seq=2 ttl=63 time=0.694 ms
64 bytes from 192.168.20.8: icmp_seq=3 ttl=63 time=0.703 ms
64 bytes from 192.168.20.8: icmp_seq=4 ttl=63 time=0.606 ms
^C
--- 192.168.20.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3039ms
rtt min/avg/max/mdev = 0.606/0.846/1.383/0.312 ms




ubuntu@fhmqn5r9a9in24050koh:~$ ssh 192.168.20.8 
The authenticity of host '192.168.20.8 (192.168.20.8)' can't be established.
ECDSA key fingerprint is SHA256:jsdzCTSEB2gycASrXerJw9ox3oI2f/GIuiIE0Lc1WZQ.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.20.8' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-113-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.




ubuntu@fhm5jrf4ibc9cdvv1iek:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=59 time=20.5 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=59 time=19.1 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=59 time=19.3 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=59 time=19.4 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 19.105/19.550/20.466/0.536 ms
```

---
---
## Задание 2*. AWS (необязательное к выполнению)

1. Создать VPC.
- Cоздать пустую VPC с подсетью 10.10.0.0/16.
2. Публичная подсеть.
- Создать в vpc subnet с названием public, сетью 10.10.1.0/24
- Разрешить в данной subnet присвоение public IP по-умолчанию. 
- Создать Internet gateway 
- Добавить в таблицу маршрутизации маршрут, направляющий весь исходящий трафик в Internet gateway.
- Создать security group с разрешающими правилами на SSH и ICMP. Привязать данную security-group на все создаваемые в данном ДЗ виртуалки
- Создать в этой подсети виртуалку и убедиться, что инстанс имеет публичный IP. Подключиться к ней, убедиться что есть доступ к интернету.
- Добавить NAT gateway в public subnet.
3. Приватная подсеть.
- Создать в vpc subnet с названием private, сетью 10.10.2.0/24
- Создать отдельную таблицу маршрутизации и привязать ее к private-подсети
- Добавить Route, направляющий весь исходящий трафик private сети в NAT.
- Создать виртуалку в приватной сети.
- Подключиться к ней по SSH по приватному IP через виртуалку, созданную ранее в публичной подсети и убедиться, что с виртуалки есть выход в интернет.

Resource terraform
- [VPC](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
- [Subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)
- [Internet Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway)