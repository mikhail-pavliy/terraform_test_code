# Тестирование инфраструктурного кода на Terraform

1. 1. Настройка рабочей станции. Установим GoLnag
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/03-terraform/terraform$ sudo apt  install golang-go -y

mikhail@mikhail-GL703VD:~/Desktop/otus/03-terraform/terraform$ go version
go version go1.18.1 linux/amd64

```
2. Пишем тесты, 
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
выполним команду инициализации модуля Go

```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform$ go mod init test
go: creating new go.mod: module test
go: to add module requirements and sums:
    go mod tidy

```
в каталоге появился файл go.mod
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform$ head go.mod
module test

go 1.18
```
4. Создание инфраструктуры

Одна из основных задач Terratest создавать и удалять тестируемые ресурсы, то мы начнем с того, что дополнить ```end2end_test.go``` необходимыми для этого командами:

```ruby
package test

import (
	"fmt"
	"flag"
	"testing"
	"time"

	"github.com/gruntwork-io/terratest/modules/terraform"
)

var folder = flag.String("folder", "", "Folder ID in Yandex.Cloud")

func TestEndToEndDeploymentScenario(t *testing.T) {

	terraformOptions := &terraform.Options{
			TerraformDir: "../",

			Vars: map[string]interface{}{
			"yc_folder":    *folder,
		    },
	}

	defer terraform.Destroy(t, terraformOptions)

	terraform.InitAndApply(t, terraformOptions)

	fmt.Println("Finish infra.....")

    time.Sleep(30 * time.Second)

    fmt.Println("Destroy infra.....")
}
```

Прежде чем запустить тест, выполним команду, скачивающую необходимые зависимости
```ruby
go mod vendor
```
Скачаем необходимые зависимости

```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go get github.com/gruntwork-io/terratest/modules/terraform
```
после чего выполним команду
```
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go test -v ./ -timeout 30m -folder  'b1gjo0j7hjhobmiqtukr'
```
В итоге мы получаем сообщение, что тестирование прошло без ошибок.

```ruby
--- PASS: TestEndToEndDeploymentScenario (491.47s)
PASS
ok      test/terraform/test     491.478s
```
5. Управление стадиями тестирования

Придадим файлу end2end_test.go следующий вид:
```ruby
package test

import (
	"fmt"
	"flag"
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	test_structure "github.com/gruntwork-io/terratest/modules/test-structure"
)

var folder = flag.String("folder", "", "Folder ID in Yandex.Cloud")

func TestEndToEndDeploymentScenario(t *testing.T) {
    fixtureFolder := "../"

    test_structure.RunTestStage(t, "setup", func() {
		terraformOptions := &terraform.Options{
			TerraformDir: fixtureFolder,

			Vars: map[string]interface{}{
			"yc_folder":    *folder,
		    },
	    }

		test_structure.SaveTerraformOptions(t, fixtureFolder, terraformOptions)

		terraform.InitAndApply(t, terraformOptions)
	})

	test_structure.RunTestStage(t, "validate", func() {
	    fmt.Println("Run some tests...")

    })

	test_structure.RunTestStage(t, "teardown", func() {
		terraformOptions := test_structure.LoadTerraformOptions(t, fixtureFolder)
		terraform.Destroy(t, terraformOptions)
	})
}
```
выполним команду
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go mod vendor
```
И запустим тест

```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go test -v ./ -timeout 30m -folder  'b1gjo0j7hjhobmiqtukr'
```
У нас точно так же отработало создание и удаление инфраструктуры, но обратите внимание на следующие строки, которые появились в логе:
```ruby
TestEndToEndDeploymentScenario 2022-08-19T21:16:55+06:00 test_structure.go:27: The 'SKIP_setup' environment variable is not set, so executing stage 'setup'.

TestEndToEndDeploymentScenario 2022-08-19T21:22:31+06:00 test_structure.go:27: The 'SKIP_validate' environment variable is not set, so executing stage 'validate'.

TestEndToEndDeploymentScenario 2022-08-19T21:22:31+06:00 test_structure.go:27: The 'SKIP_teardown' environment variable is not set, so executing stage 'teardown'.

```
Упоминаются несколько переменных окружения, имя которых состоит из префикса “SKIP_” и имени стадии. И именно при помощи этих переменных мы можем управлять тем, будет ли выполняться та или иная стадия, или нет. Проверим так ли это. Допустим мы хотим, чтобы при очередном запуске выполнилась только стадия setup (создания инфраструктуры).

Для этого мы определим две переменные окружения:

```export SKIP_validate=true```
```export SKIP_teardown=true```

И снова запустим тест:
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go test -v ./ -timeout 30m -folder  'b1gjo0j7hjhobmiqtukr'
```
На этот раз отработала только стадия создания инфраструктуры. Стадии непосредственно тестирования и удаления инфраструктуры не запустились.
Соответственно, в логе можно увидеть:
```ruby
TestEndToEndDeploymentScenario 2022-08-19T21:31:32+06:00 test_structure.go:27: The 'SKIP_setup' environment variable is not set, so executing stage 'setup'.

TestEndToEndDeploymentScenario 2022-08-19T21:38:00+06:00 test_structure.go:30: The 'SKIP_validate' environment variable is set, so skipping stage 'validate'.

TestEndToEndDeploymentScenario 2022-08-19T21:38:00+06:00 test_structure.go:30: The 'SKIP_teardown' environment variable is set, so skipping stage 'teardown'.
```
И, если при следующих итерациях, мы хотим чтобы запускалась только стадия ```validate```, то мы сделаем следующее:
```export SKIP_setup=true```
```unset SKIP_validate```

И проверим, получилось ли то, что нам нужно:
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go test -v ./ -timeout 30m -folder  'b1gjo0j7hjhobmiqtukr'
=== RUN   TestEndToEndDeploymentScenario
TestEndToEndDeploymentScenario 2022-08-19T21:51:03+06:00 test_structure.go:30: The 'SKIP_setup' environment variable is set, so skipping stage 'setup'.
TestEndToEndDeploymentScenario 2022-08-19T21:51:03+06:00 test_structure.go:27: The 'SKIP_validate' environment variable is not set, so executing stage 'validate'.
Run some tests...
TestEndToEndDeploymentScenario 2022-08-19T21:51:03+06:00 test_structure.go:30: The 'SKIP_teardown' environment variable is set, so skipping stage 'teardown'.
--- PASS: TestEndToEndDeploymentScenario (0.00s)
PASS
ok      test/terraform/test     0.017s
```
Да, мы получили то, что хотели.

6. Пишем тесты

Для проверки наличия IP балансировщика добавим в стадию validate после строчки

```fmt.Println("Run some tests...")```

следующий блок
```ruby
  terraformOptions := test_structure.LoadTerraformOptions(t, fixtureFolder)

        // test load balancer ip existing
	    loadbalancerIPAddress := terraform.Output(t, terraformOptions, "load_balancer_public_ip")

	    if loadbalancerIPAddress == "" {
			t.Fatal("Cannot retrieve the public IP address value for the load balancer.")
		}
```
Запустим тест:
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go test -v ./ -timeout 30m -folder  'b1grqtq9v9u92suhmdcg'
=== RUN   TestEndToEndDeploymentScenario
TestEndToEndDeploymentScenario 2022-08-19T22:38:35+06:00 test_structure.go:30: The 'SKIP_setup' environment variable is set, so skipping stage 'setup'.
TestEndToEndDeploymentScenario 2022-08-19T22:38:35+06:00 test_structure.go:27: The 'SKIP_validate' environment variable is not set, so executing stage 'validate'.
Run some tests...
TestEndToEndDeploymentScenario 2022-08-19T22:38:35+06:00 save_test_data.go:215: Loading test data from ../.test-data/TerraformOptions.json
TestEndToEndDeploymentScenario 2022-08-19T22:38:35+06:00 retry.go:91: terraform [output -no-color -json load_balancer_public_ip]
TestEndToEndDeploymentScenario 2022-08-19T22:38:35+06:00 logger.go:66: Running command terraform with args [output -no-color -json load_balancer_public_ip]
TestEndToEndDeploymentScenario 2022-08-19T22:38:35+06:00 logger.go:66: "62.84.118.48"
TestEndToEndDeploymentScenario 2022-08-19T22:38:35+06:00 test_structure.go:30: The 'SKIP_teardown' environment variable is set, so skipping stage 'teardown'.
--- PASS: TestEndToEndDeploymentScenario (0.18s)
PASS
ok      test/terraform/test     0.194s
```
перейдем к более сложному тесту - проверки возможности подключения по ssh.
Для этого мы добавим возможность указание еще одного флага - пути к приватному ключу.

После определения переменной
```var folder = ...```

Добавим еще одну

```ruby
var sshKeyPath = flag.String("ssh-key-pass", "", "Private ssh key for access to virtual machines")
```
Далее запускаем тест
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go test -v ./ -timeout 30m -folder  'b1gjo0j7hjhobmiqtukr' -ssh-key-pass '/home/mikhail/.ssh/id_rsa'

=== RUN   TestEndToEndDeploymentScenario
TestEndToEndDeploymentScenario 2022-08-21T16:39:32+06:00 test_structure.go:30: The 'SKIP_setup' environment variable is set, so skipping stage 'setup'.
TestEndToEndDeploymentScenario 2022-08-21T16:39:32+06:00 test_structure.go:27: The 'SKIP_validate' environment variable is not set, so executing stage 'validate'.
Run some tests...
TestEndToEndDeploymentScenario 2022-08-21T16:39:32+06:00 save_test_data.go:215: Loading test data from ../.test-data/TerraformOptions.json
TestEndToEndDeploymentScenario 2022-08-21T16:39:32+06:00 retry.go:91: terraform [output -no-color -json load_balancer_public_ip]
TestEndToEndDeploymentScenario 2022-08-21T16:39:32+06:00 logger.go:66: Running command terraform with args [output -no-color -json load_balancer_public_ip]
TestEndToEndDeploymentScenario 2022-08-21T16:39:32+06:00 logger.go:66: "158.160.9.141"
TestEndToEndDeploymentScenario 2022-08-21T16:39:32+06:00 retry.go:91: terraform [output -no-color -json vm_linux_public_ip_address]
TestEndToEndDeploymentScenario 2022-08-21T16:39:32+06:00 logger.go:66: Running command terraform with args [output -no-color -json vm_linux_public_ip_address]
TestEndToEndDeploymentScenario 2022-08-21T16:39:32+06:00 logger.go:66: "51.250.83.49"
TestEndToEndDeploymentScenario 2022-08-21T16:39:34+06:00 test_structure.go:30: The 'SKIP_teardown' environment variable is set, so skipping stage 'teardown'.
--- PASS: TestEndToEndDeploymentScenario (1.85s)
PASS
ok      test    1.873s
```
Оба теста прошли успешно.

В финале, проверим полный цикл прохождения всех стадий.
Сначала удалим нашу текущую инфраструктуру.

Для этого удалим переменную окружения SKIP_teardown:

```unset SKIP_teardown```

Снова запустим тесты и на этот раз после их выполнения будет проведено удаление инфраструктуры:
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go test -v ./ -timeout 30m -folder  'b1gjo0j7hjhobmiqtukr' -ssh-key-pass '/home/mikhail/.ssh/id_rsa'

....

TestEndToEndDeploymentScenario 2022-08-21T16:42:42+06:00 logger.go:66: Destroy complete! Resources: 9 destroyed.
TestEndToEndDeploymentScenario 2022-08-21T16:42:42+06:00 logger.go:66: 
--- PASS: TestEndToEndDeploymentScenario (80.75s)
PASS
ok      test    80.773s
```
Теперь удалим оставшуюся переменную окружения ```SKIP_```
```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ env | grep SKIP
SKIP_setup=true

mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ unset SKIP_setup

```
И запустим полный цикл тестирования

```ruby
mikhail@mikhail-GL703VD:~/Desktop/otus/06-terraform/terraform/test$ go test -v ./ -timeout 30m -folder  'b1gjo0j7hjhobmiqtukr' -ssh-key-pass '/home/mikhail/.ssh/id_rsa'

```


