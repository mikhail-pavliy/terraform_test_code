# Тестирование инфраструктурного кода на Terraform

1. 1. Настройка рабочей станции. Установим GoLnag
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/03-terraform/terraform$ sudo apt  install golang-go -y

mikhail@mikhail-GL703VD:~/Desktop/otus/03-terraform/terraform$ go version
go version go1.18.1 linux/amd64

```
2. Пишем тесты
Слегка облегчим себе работу с output-переменными терраформа. Сначала изменим вывод переменной load_balancer_public_ip на следующий:
```ruby
output "load_balancer_public_ip" {
  description = "Public IP address of load balancer"
  value = tolist(tolist(yandex_lb_network_load_balancer.wp_lb.listener).0.external_address_spec).0.address
}
```
добавим еще одну переменную для определения IP одной из виртуальных машин

```ruby
output "vm_linux_public_ip_address" {
  description = "Virtual machine IP"
  value = yandex_compute_instance.wp-app-1.network_interface[0].nat_ip_address
}
```
3. Создадим в каталоге ```terraform``` подкаталог ```test```.
В test создадим файл ```end2end_test.go``` со следующим содержимым:

```ruby
package test

import (
    "testing"
)

func TestEndToEndDeploymentScenario(t *testing.T) {
}
```


