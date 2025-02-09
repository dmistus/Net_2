# «Вычислительные мощности. Балансировщики нагрузки»

###  &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Задание 1. Yandex Cloud

Что нужно сделать:

## 1.
 &nbsp; &nbsp; &nbsp; &nbsp;*-Создать бакет Object Storage и разместить в нём файл с картинкой:*
 
 &nbsp; &nbsp; &nbsp; &nbsp;*-Создать бакет в Object Storage с произвольным именем (например, _имя_студента_дата_).*
 
 &nbsp; &nbsp; &nbsp; &nbsp;*-Положить в бакет файл с картинкой.*
 
 &nbsp; &nbsp; &nbsp; &nbsp;*-Положить в бакет файл с картинкой.*
 
 &nbsp; &nbsp; &nbsp; &nbsp;*- Сделать файл доступным из интернета.*&ensp;

## 2.

 &nbsp;&nbsp; &nbsp; &nbsp;*- Создать группу ВМ в public подсети фиксированного размера с шаблоном LAMP и веб-страницей, содержащей ссылку на картинку из бакета:*&ensp;
 
&nbsp; &nbsp; &nbsp; &nbsp;*- Создать Instance Group с тремя ВМ и шаблоном LAMP. Для LAMP рекомендуется использовать `image_id = fd827b91d99psvq5fjit`.*&ensp; 

&nbsp; &nbsp; &nbsp; &nbsp;*- Создать Instance Group с тремя ВМ и шаблоном LAMP. Для LAMP рекомендуется использовать `image_id = fd827b91d99psvq5fjit`.*&ensp; 

&nbsp; &nbsp; &nbsp; &nbsp;*-  - Для создания стартовой веб-страницы рекомендуется использовать раздел `user_data` в [meta_data](https://cloud.yandex.ru/docs/compute/concepts/vm-metadata).*&ensp; 

 &nbsp; &nbsp; &nbsp; &nbsp;*- Разместить в стартовой веб-странице шаблонной ВМ ссылку на картинку из бакета.*&ensp;
 
  &nbsp; &nbsp; &nbsp; &nbsp;*- Настроить проверку состояния ВМ.*
 
## 3. Подключить группу к сетевому балансировщику:

 &nbsp; &nbsp; &nbsp; &nbsp;*- Создать сетевой балансировщик.*
 
 &nbsp; &nbsp; &nbsp; &nbsp;*- Проверить работоспособность, удалив одну или несколько ВМ.*

4. (дополнительно)* Создать Application Load Balancer с использованием Instance group и проверкой состояния.


Создем бакет в Object Storage с моими инициалами и текущей датой:

```
// Создаем сервисный аккаунт для backet
resource "yandex_iam_service_account" "service" {
  folder_id = var.folder_id

  name      = "bucket-sa"
}

resource "yandex_resourcemanager_folder_iam_member" "bucket-editor" {
  folder_id = var.folder_id
  role      = "storage.editor"
  member    = "serviceAccount:${yandex_iam_service_account.service.id}"
  depends_on = [yandex_iam_service_account.service]
}

resource "yandex_iam_service_account_static_access_key" "sa-static-key" {
  service_account_id = yandex_iam_service_account.service.id
  description        = "static access key for object storage"
}

resource "yandex_storage_bucket" "levchenkods" {
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  bucket = local.bucket_name
  acl    = "public-read"
}
```


Переменная для текущей даты:

```
locals {
    current_timestamp = timestamp()
    formatted_date = formatdate("DD-MM-YYYY", local.current_timestamp)
    bucket_name = "levchenkods-${local.formatted_date}"
}
```

Загрузем в бакет файл с картинкой:

```
resource "yandex_storage_object" "deadline-picture" {
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  bucket = local.bucket_name
  key    = "deadline-cat"
  source = "~/deadline-cat.jpg"
  acl = "public-read"
  depends_on = [yandex_storage_bucket.levchenkods]
}
```



 Проверяем созданный бакет:

![](img/1.png)
![](img/2.png)

Создадим группу ВМ в public подсети фиксированного размера с шаблоном LAMP и веб-страницей, содержащей ссылку на картинку из бакета.


```
variable "default_cidr" {
  type        = list(string)
  default     = ["10.0.1.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "vpc_name" {
  type        = string
  default     = "develop"
  description = "VPC network&subnet name"
}

variable "public_subnet" {
  type        = string
  default     = "public-subnet"
  description = "subnet name"
}

resource "yandex_vpc_network" "develop" {
  name = var.vpc_name
}

resource "yandex_vpc_subnet" "public" {
  name           = var.public_subnet
  zone           = var.default_zone
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = var.default_cidr
}
```

Cоздание стартовой веб-страницы:

```
    user-data  = <<EOF
#!/bin/bash
cd /var/www/html
echo '<html><head><title>Picture of my cat</title></head> <body><h1> </h1><img src="http://${yandex_storage_bucket.levchenkods.bucket_domain_name}/deadline-cat.jpg"/></body></html>' > index.html
EOF
```


Проверка состояния виртуальной машины:

```
  health_check {
    interval = 30
    timeout  = 10
    tcp_options {
      port = 80
    }
  }
```

Примененим код Terraform:

![](img/3.png)

Получили три настроенные по шаблону LAMP виртуальные машины.

Создим сетевой балансировщик и подключим к нему группу виртуальных машин:

```
resource "yandex_lb_network_load_balancer" "balancer" {
  name = "lamp-balancer"
  deletion_protection = "false"
  listener {
    name = "http-check"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }
  attached_target_group {
    target_group_id = yandex_compute_instance_group.group-vms.load_balancer[0].target_group_id
    healthcheck {
      name = "http"
      interval = 2
      timeout = 1
      unhealthy_threshold = 2
      healthy_threshold = 5
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}
```

Проверим статус балансировщика нагрузки и подключенной к нему группе виртуальных машин после применения кода: 

![](img/4.png)


![](img/5.png)


Проверю доступность сайта, через балансировщик нагрузки:

![](img/6.png)


![](img/7.png)

Проверим, доступность сайта после отключения пары виртуальных машин.

![](img/8.png)

Сайт по прежнему доступен.

После срабатывания Healthcheck, выключенные виртуальные машины LAMP были заново запущены:

![](img/9.png)




