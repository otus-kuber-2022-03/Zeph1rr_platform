# Zeph1rr_platform

## Kubernetes-Debug

### Выполнено
- Ознакомились с работой `kubectl debug`, выявили проблему с агентом debug;
- Установили kube-iptables-tailer и необходимые для него сервис аккаунт, роль и биндинг;
- Установили тестовое приложение netperf-operator (оператор, который позволяет запускать тесты
пропускной способности сети между нодами кластера); 
- Добавили сетевую потилику Calico для эмуляции сетевой проблемы;
- Посмотрели логи calico на воркер-ноде `journalctl -k | grep calico-pack`;
- Запустили iptables-tailer для диагностики сетевой проблемы и добились того, чтобы iptables-tailer нормально запустился и писал лог;

### Выполнено задание со *
- Исправлена ошибка в сетевой политике, чтобы netperf заработал; 
- Исправлен манифест DaemonSet, чтобы показывались имена подов место ip адресов(?). 

### Как запустить: 
#### Предварительные действия: 
- Развернуть кластер (kind не подходит, нужен полноценный кластер минимум с 2 воркер нодами (минимум 2 ноды нужно для работы тестового приложения netperf-operator)); 
- Установить kubectl debug, согласно документации https://github.com/aylei/kubectl-debug
- Клонировать репозиторий netperf-operator - `git clone https://github.com/piontec/netperf-operator.git`
- Выполнить манифесты для запуска оператора в кластере: 
```
cd kubernetes-debug/netperf-operator
kubectl apply -f ./deploy/crd.yaml
kubectl apply -f ./deploy/rbac.yaml
kubectl apply -f ./deploy/operator.yaml

```

#### Основные действия: 
##### Задание 1 (проблема с контейнером-агентом):
При выполнении strace получаем ошибку вида: 

```
bash-5.0# strace -c -p1
strace: test_ptrace_get_syscall_info: PTRACE_TRACEME: Operation not permitted
strace: attach: ptrace(PTRACE_ATTACH, 1): Operation not permitted
```

Смотрим на наличие capabilities с использованием команды `docker inspect` (выполняем для запущенного debug контейнера на воркер-ноде):

```
            "CapAdd": null,
            "CapDrop": null,
``` 

Смотри добавляются ли необходимые capabilities в коде (по той ссылке что в подсказке): 
```
CapAdd:      strslice.StrSlice([]string{"SYS_PTRACE", "SYS_ADMIN"}),
```
- в коде все в порядке.

Смотрим версию образа агента: 

```
Normal  Pulling    16m   kubelet, node2     Pulling image "aylei/debug-agent:0.0.1"
```
- версия подозрительно старая. 

Смотри код в старой версии по ссылке https://github.com/aylei/kubectl-debug/blob/0.0.1/pkg/agent/runtime.go - а нужной строчки с capabilities там нет. 

Удаляем агента: 
`kubectl delete -f https://raw.githubusercontent.com/aylei/kubectl-debug/master/scripts/agent_daemonset.yml`

Устанавливаем последнюю версию агента, используя такой же манифест как и официальный, только с измененной версий образа (изменена на latest):
`kubectl apply -f kubernetes-debug/strace/01-agent_daemonset.yaml`

После этого еще раз проверяем strace и видим, что все успешно отрабатывает:

```
bash-5.0# strace -c -p1
strace: Process 1 attached
^Cstrace: Process 1 detached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0         2           poll
  0.00    0.000000           0         1           restart_syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.000000                     3           total
```

##### Задание 2 (диагностика сетевой проблемы):
Выполнить уже исправленные манифесты в следующей последовательности: 
- `kubectl apply -f kit/kit-clusterrole.yaml`;
- `kubectl apply -f kit/kit-serviceaccount.yaml`;
- `kubectl apply -f kit/kit-clusterrolebinding.yaml`;
- `kubectl apply -f kit/netperf-calico-policy.yaml`;
- `kubectl apply -f kit/iptables-tailer.yaml`;

Для запуска тестового приложения выполнить: 
- `kubectl apply -f netperf-operator/deploy/cr.yaml`;


##### Задание со `*`:
- Для исправления ошибки в сетевой политике необходимо исправить и применить манифест `kit/netperf-calico-policy.yaml`. Нужно строчки `selector: netperf-role == "netperf-client"` заменить на `selector: netperf-type in {"client", "server"}` (согласно документации https://docs.projectcalico.org/v3.7/reference/calicoctl/resources/globalnetworkpolicy#selector). И после этого запустить для проверки тест еще раз;
-  Возможно, судя по документации (https://github.com/box/kube-iptables-tailer), для того, чтобы отображалось имя пода, необходимо в DaemonSet изменить значение переменной окружения POD_IDENTIFIER с `label` на `name` (файл `kit/iptables-tailer.yaml`) и применить манифест, но я разницы не заметил. У меня в обоих случаях в поле `OBJECT` отображается имя пода, а в `MESSAGE` присутствует его ip адрес, делалось все на кластере GCE v1.14.  