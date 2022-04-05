# Домашнее задание к занятию "12.5 Сетевые решения CNI"
После работы с Flannel появилась необходимость обеспечить безопасность для приложения. Для этого лучше всего подойдет Calico.

## Задание 1: установить в кластер CNI плагин Calico
Для проверки других сетевых решений стоит поставить отличный от Flannel плагин — например, Calico. Требования: 
* установка производится через ansible/kubespray;
* после применения следует настроить политику доступа к hello world извне.

## Решение задания 1: установить в кластер CNI плагин Calico
Полиси брал из [kubernetes-for-beginners](https://github.com/aak74/kubernetes-for-beginners/tree/master/16-networking/20-network-policy/templates/network-policy)
единственноt я поправил deny

```commandline
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

![4](/img/dz_12_5_1.png)

![5](/img/dz_12_5_2.png)

![6](/img/dz_12_5_3.png)

![7](/img/dz_12_5_4.png)

![8](/img/dz_12_5_5.png)

![9](/img/dz_12_5_6.png)


## Задание 2: изучить, что запущено по умолчанию
Самый простой способ — проверить командой calicoctl get <type>. Для проверки стоит получить список нод, ipPool и profile.
Требования: 
* установить утилиту calicoctl;
* получить 3 вышеописанных типа в консоли.


## Решение задания 2

![1](/img/dz_12_5__2_1.png)

![2](/img/dz_12_5__2_2.png)

![3](/img/dz_12_5__2_3.png)



---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
