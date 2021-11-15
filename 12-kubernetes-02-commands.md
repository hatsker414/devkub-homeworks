# Домашнее задание к занятию "12.2 Команды для работы с Kubernetes"
Кластер — это сложная система, с которой крайне редко работает один человек. Квалифицированный devops умеет наладить работу всей команды, занимающейся каким-либо сервисом.
После знакомства с кластером вас попросили выдать доступ нескольким разработчикам. Помимо этого требуется служебный аккаунт для просмотра логов.

## Задание 1: Запуск пода из образа в деплойменте
Для начала следует разобраться с прямым запуском приложений из консоли. Такой подход поможет быстро развернуть инструменты отладки в кластере. Требуется запустить деплоймент на основе образа из hello world уже через deployment. Сразу стоит запустить 2 копии приложения (replicas=2). 

Требования:
 * пример из hello world запущен в качестве deployment
 * количество реплик в deployment установлено в 2
 * наличие deployment можно проверить командой kubectl get deployment
 * наличие подов можно проверить командой kubectl get pods


## Задание 1. Решение

```
kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4 --replicas=2
```

![1](/img/dz_12_2_1.png)


## Задание 2: Просмотр логов для разработки
Разработчикам крайне важно получать обратную связь от штатно работающего приложения и, еще важнее, об ошибках в его работе. 
Требуется создать пользователя и выдать ему доступ на чтение конфигурации и логов подов в app-namespace.

Требования: 
 * создан новый токен доступа для пользователя
 * пользователь прописан в локальный конфиг (~/.kube/config, блок users)
 * пользователь может просматривать логи подов и их конфигурацию (kubectl logs pod <pod_id>, kubectl describe pod <pod_id>)

## Задание 2. Решение 
 
задание выполнял по [этой](https://stackoverflow.com/questions/44948483/create-user-in-kubernetes-for-kubectl) инструкции 


Create a ServiceAccount, say 'readonlyuser'.
```
kubectl create serviceaccount readonlyuser
```
Create cluster role, say 'readonlyuser'.
```
kubectl create clusterrole readonlyuser --verb=get --verb=list --verb=watch --resource=pods
```
Create cluster role binding, say 'readonlyuser'.
```
kubectl create clusterrolebinding readonlyuser --serviceaccount=default:readonlyuser --clusterrole=readonlyuser
```
Now get the token from secret of ServiceAccount we have created before. we will use this token to authenticate user.
```
TOKEN=$(kubectl describe secrets "$(kubectl describe serviceaccount readonlyuser | grep -i Tokens | awk '{print $2}')" | grep token: | awk '{print $2}')
```
Now set the credentials for the user in kube config file. I am using 'hatsker' as username.
```
kubectl config set-credentials hatsker --token=$TOKEN
```
Now Create a Context say podreader. I am using my clustername 'minikube' here.
```
kubectl config set-context podreader --cluster=minikube --user=hatsker
```
Finally use the context .
```
kubectl config use-context podreader
```
And that's it. Now one can execute kubectl get pods --all-namespaces. One can also check the access by executing as given:
```
~ : $ kubectl auth can-i get pods --all-namespaces
yes
~ : $ kubectl auth can-i create pods
no
~ : $ kubectl auth can-i delete pods
no
```
![3](/img/dz_12_2_2.png)
![4](/img/dz_12_2_2_1.png)

## Задание 3: Изменение количества реплик 
Поработав с приложением, вы получили запрос на увеличение количества реплик приложения для нагрузки. Необходимо изменить запущенный deployment, увеличив количество реплик до 5. Посмотрите статус запущенных подов после увеличения реплик. 

Требования:
 * в deployment из задания 1 изменено количество реплик на 5
 * проверить что все поды перешли в статус running (kubectl get pods)

## Задание 3. Решение 

```kubectl scale --replicas=5 deploy hello-node
```
![2](/img/dz_12_2_3.png)

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
