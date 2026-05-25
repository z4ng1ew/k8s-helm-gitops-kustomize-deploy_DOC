## Part 1. Развертывание приложения с помощью Kustomize





 **Задание**

## 1. Получить набор виртуальных машин с развернутым кластером.

![alt text](<img/1/image copy.png>)


![alt text](<img/1/image copy 8.png>)


В default  остался   Service kubernetes (это сам API-сервер k8s, его не трогаем) 

Очистил namespace default от приложения DO11, чтобы не конфликтовало с DO12




## 2. Перенести манифесты из предыдущих блоков.



![alt text](<img/1/image copy 2.png>)


![alt text](<img/1/image copy 9.png>)









## 3. Установить *kustomize* на локальной машине.




**Скачиваем установщик и переносим в системный PATH (чтобы команда работала из любой папки)**

![alt text](img/1/image.png)


**kustomize version v5.8.1 - версия установлена**















## 4. Cоздать скелет проекта развертывания с одной базовой конфигурацией (base) и одной оверлейной конфигурацией (production):

```
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   ├── service.yaml
│   └── ...
├── overlays
│   └── production
│       ├── kustomization.yaml
│       ├── configMap.yaml
│       ├── secret.yaml
│       └── ...
├── kustomization.yaml
└── ...
```


![alt text](<img/1/image copy 3.png>)


**Структура файлов и деркторий создана**


































## 5. Написать базовые и оверлейные конфигурации для kustomize. В базовых указать сервисы и развертывания, в production добавить конкретные секреты и конфигурационные значения.


![alt text](<img/1/image copy 4.png>)

/home/s/do_12/src/kustomize/base/kustomization.yaml

**Главный файл Kustomize. Он говорит: «вот эти YAML-файлы — части моей базы» (то, что одинаково для всех окружений: dev, staging, production).**

перечислены все 11 манифестов

apiVersion и kind — обязательные «маркеры», что это файл Kustomize. resources — список файлов, которые нужно объединить. Кастомайз их склеит и применит к кластеру всё сразу.



 - apiVersion и kind — обязательные маркеры, что это файл Kustomize
 - resources — список манифестов, которые будут объединены в одну сборку


























![alt text](<img/1/image copy 6.png>)

/home/s/do_12/src/kustomize/overlays/production/kustomization.yaml

Оверлей production: накладывается поверх базовой конфигурации


    Расшифровка:

1) resources — берём всё из ../../base (путь относительный — две точки означают «папка выше», то есть kustomize/, потом base/)
2) Плюс добавляем ConfigMap и Secret из текущей папки
3) patches — список патчей. У меня один — увеличить реплики gateway до 3.























![alt text](<img/1/image copy 7.png>)

/home/s/do_12/src/kustomize/kustomization.yaml

**корневой kustomization.yaml**








## 6. Создать `replicas-patch.yaml` для оверлей production, который модифицирует количество реплик для деплоймента gateway service до 3 реплик.

![alt text](<img/1/image copy 5.png>)


/home/s/do_12/src/kustomize/overlays/production/replicas-patch.yaml 

replicas-patch.yaml — патч, увеличивающий gateway до 3 реплик
Это не полный манифест, а только те поля, которые меняем





**Оверлей добавляет ConfigMap, Secret и патч на 3 реплики для gateway.**































































![alt text](<img/1/image copy 9.png>)



**все файлы в /home/s/do_12/src/kustomize/base**








## 7. Собрать результирующий конфигурационный файл, учитывая оверлей `production`.




![alt text](<img/1/image copy 10.png>)

```
kustomize build overlays/production | head -50
```



![alt text](<img/1/image copy 11.png>)

```
kustomize build overlays/production | grep -A 3 "name: gateway-service" | head -20
```



вывод kustomize build + grep gateway-service показывает replicas: 3 рядом с name: gateway-service.




Патч применился, gateway-service получит 3 реплики, остальные — по 1. final-manifest.yaml собран (764 строки, 17 КБ). Можно применять.



доказательство пункта 6 методички («модифицирует количество реплик для gateway service до 3 реплик»).
















![alt text](<img/1/image copy 13.png>)

[text](kustomize/final-manifest.yaml)

/home/s/do_12/src/kustomize/final-manifest.yaml
















## 7.5 Применение к кластеру



![alt text](<img/1/image copy 12.png>)

**Применяем оверлей production**

Флаг -k — это «kustomize mode» (режим Kustomize) в kubectl. Он сам внутренне выполнит kustomize build и применит результат.





![alt text](<img/1/image copy 14.png>)

3 пода gateway-service (это и есть патч на 3 реплики) и по одному каждого из остальных. 









![alt text](<img/1/image copy 15.png>)

gateway-service   3/3 (3 готовых из 3) и 1/1 у всех остальных.




























## 8. Запустить функциональные тесты Postman и удостовериться в работоспособности приложения.














![alt text](<img/1/image copy 16.png>)

**все результаты 200/201 - УСПЕШНО!**






![alt text](<img/image copy 4.png>)

**Веб-версия результатов теста**





































## Part 2. Развертывание приложения с помощью Helm

Перед началом Part 2 удалил предыдущее развёртывание из Kustomize командой kubectl delete -k overlays/production. Namespace default пуст, namespace hotels будет создан в пункте 7.




## 1. Получить набор виртуальных машин с развернутым кластером.


![alt text](img/2/image.png)

**kubectl get nodes**



















## 2. Перенести манифесты из предыдущих блоков.


![alt text](<img/2/image copy.png>)

**список манифестов**




























## 3. Установить *helm* на локальной машине и удостовериться, что этот инструмент имеет валидное подключение к полученному удаленному кластеру Kubernetes.


![alt text](<img/1/image copy 17.png>)

**kube-prometheus (Prometheus из DO11 - котрый я делал ещё тогда)**


**Версия Helm - v3.21.0**

























## 4. Создать *helm*-чарты и шаблоны для своего приложения с помощью команды `helm create`. Эта команда создаст базовую структуру чарта с шаблонами для ресурсов: deployment, service и ingress.

![alt text](<img/2/image copy 3.png>)

Задание просит использовать helm create

![alt text](<img/2/image copy 2.png>)


```
hotels-app/
├── Chart.yaml          ← метаданные
├── values.yaml         ← значения
├── charts/             ← зависимости (нам не нужно)
├── templates/          ← шаблоны манифестов
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   └── tests/
└── .helmignore
```













Очиcтил дефолтные шаблоны и создал свои
Helm создал шаблон одного приложения. У меня 9 сервисов, поэтому удалил сгенерированное:

![alt text](<img/1/image copy 18.png>)















![alt text](<img/2/image copy 4.png>)

копируем манифесты из kustomize/base/
























## 5. Отредактировать файл `values.yaml` на диаграмме, чтобы указать параметры конфигурации для твоего приложения, необходимые для создания манифестов Kubernetes для указанных развертываний (deployments). Описать объекты развертывания и сервисов в директории шаблонов (templates).



![alt text](<img/2/image copy 5.png>)


values.yaml - значения для подстановки в шаблоны.
Меняя эти значения, можно настроить приложение под разные окружения.

- replicaCount.gateway — внутри шаблона доступ через {{ .Values.replicaCount.gateway }}
- image.postgres — через {{ .Values.image.postgres }}
- Иерархия в YAML = иерархия через точку в шаблоне







Превратил манифесты в шаблоны

![alt text](<img/2/image copy 9.png>)




Для примера смотрим:

![alt text](<img/2/image copy 6.png>)

**cat 12-gateway-service.yaml  ---  видны {{ ... }}.**







![alt text](<img/2/image copy 7.png>)




Переписал Ingress на шаблон


- {{- if .Values.ingress.enabled }} — условный блок. Если в values.yaml написано ingress.enabled: false, весь блок до {{- end }} пропустится, и Ingress не создастся. Это мощный приём: одним флагом включаем/выключаем целые куски.
- Дефис - в {{- убирает пустую строку, которая появляется в результате. Это косметика.




















![alt text](<img/2/image copy 11.png>)

**Исравил Chart.yaml**


- apiVersion: v2 — современная схема Helm 3 (v1 была у Helm 2)
- version — версия чарта. Меняется при правке шаблонов.
- appVersion — версия самого приложения, которое чарт разворачивает. Независимая от version.
- type: application — обычное приложение (есть ещё library — библиотечный чарт без шаблонов)






![alt text](<img/2/image copy 12.png>)


Удалил артефакты helm create
helm create сгенерировал несколько файлов, которые не нужны в проекте:

```
cd ~/do_12/src/hotels-app

# Удаляем папку tests/ — там был дефолтный шаблон connection-теста
rm -rf templates/tests/

# Удаляем _helpers.tpl —  не использую helper-функции
rm -f templates/_helpers.tpl

# NOTES.txt можно было оставить, но я удалил
rm -f templates/NOTES.txt

# charts/ — для зависимостей. У меня их нет.
rm -rf charts/
```




![alt text](<img/2/image copy 15.png>)

**осталось 13 файлов в templates**





![alt text](<img/2/image copy 16.png>)



**helm lint .    ---  1 chart(s) linted, 0 chart(s) failed**





![alt text](<img/2/image copy 17.png>)


**Шаблоны рендерятся без ошибок**


В блоке Ingress подставлены session.10.174.8.95.nip.io и gateway.10.174.8.95.nip.io.
























## 6. Упаковать *helm*-чарт с помощью команды `helm package` для создания файла `*.tgz`, содержащего чарт и его зависимости.



![alt text](<img/2/image copy 18.png>)

архив создан

упаковываем чарт и затем устанавливаем









































## 7. Развернуть *helm*-чарт в кластере Kubernetes с помощью команды `helm install`. Указать произвольный `namespace` и `release-name`.




![alt text](<img/2/image copy 19.png>)


Создаём namespace для нашего релиза


```
s@Vivo:~/do_12/src$ kubectl create namespace hotels
namespace/hotels created
```










![alt text](<img/2/image copy 20.png>)

**helm install со статусом deployed.**








## 8. Проверить статус развернутого приложения при помощи команды `kubectl get`. Результаты представить в отчете.





![alt text](<img/2/image copy 21.png>)

```
helm list -A
helm status my-hotels -n hotels
```

**Смотрим что Helm знает**







```
kubectl get all -n hotels
kubectl get pods -n hotels -w
```

**Что в кластере**

![alt text](<img/2/image copy 22.png>)


![alt text](<img/2/image copy 23.png>)



![alt text](<img/2/image copy 24.png>)




## 9. Внести как минимум одно изменение в `values.yaml` и выполнить команду `helm upgrade`. 






![alt text](<img/1/image copy 19.png>)

**Меняем gateway: 1 → 2**




![alt text](<img/1/image copy 21.png>)

**Применяем**









![alt text](img/image.png)

**Проверил, что реплик стало 2**


kubectl get deployment — gateway 2/2
helm history — две ревизии (revision 1 = superseded, revision 2 = deployed)









## 10. Запустить функциональные тесты Postman и удостовериться в работоспособности приложения.


![alt text](<img/image copy 2.png>)


![alt text](<img/image copy 3.png>)

**Веб версия ньюман, результаты тоже 200/201**










































## Бонус: применение обратной связи от пира из DO11 — StatefulSet для PostgreSQL и RabbitMQ

Реализован более подходящую архитектурную вместо того что бы  использовать Deployment для stateful-нагрузок 
(БД, очереди) — это антипаттерн. В DO12 я учёл это  и перевёл PostgreSQL и RabbitMQ на StatefulSet + Headless Service.




**Теория**

Главное отличие
Deployment делает поды взаимозаменяемыми:

- Имена случайные: postgres-7f8c9-abcde, postgres-7f8c9-xyz12
- Если под умрёт — k8s создаст новый под с новым именем и новым IP
- Все поды одного Deployment одинаковые — k8s не различает их

StatefulSet делает поды с устойчивой идентичностью:

- Имена предсказуемые: postgres-0, postgres-1, postgres-2
- Если postgres-0 умрёт — новый под получит то же имя postgres-0
- Каждому поду — свой собственный PVC (через volumeClaimTemplates)
- Порядок старта/остановки гарантирован: -0 → -1 → -2


**Headless Service** (буквально «безголовый») — это Service без 
ClusterIP. Создаётся флагом `clusterIP: None`. Зачем он нужен 
для StatefulSet:

- Обычный Service даёт **один виртуальный IP** для всей группы 
  подов и балансирует трафик случайно. Кто конкретно ответит — 
  неизвестно.
- Headless Service даёт **прямой DNS к каждому конкретному поду**: 
  `postgres-0.postgres.default.svc.cluster.local`.
- StatefulSet требует Headless Service в поле `serviceName` — это 
  обязательное условие.

Для нашей одной реплики разница маленькая, но если завтра 
понадобится Primary + Replica у PostgreSQL — клиенты смогут 
адресовать конкретно `postgres-0` (мастер) и `postgres-1` (реплика).



[text](kustomize/base/04-postgres.yaml)

/home/s/do_12/src/kustomize/base/04-postgres.yaml




[text](kustomize/base/03-postgres-pv.yaml)

/home/s/do_12/src/kustomize/base/03-postgres-pv.yaml







Отдельный PVC в 03-postgres-pv.yaml больше не нужен. Но сам PV нужен, потому что volumeClaimTemplate будет требовать том на 5Gi с storageClassName: manual, а его кто-то должен предоставить.



/home/s/do_12/src/kustomize/base/05-rabbitmq.yaml

[text](kustomize/base/05-rabbitmq.yaml)


PVC я не добавлял для RabbitMQ — он у меня в учебной задаче работает на emptyDir (временной памяти пода), как и было в DO11. Для production обязательно был бы PVC, но методичка этого не требовала. Достаточно того, что я перевел на StatefulSet — это уже архитектурно правильно.























![alt text](<img/image copy 5.png>)

StatefulSet: postgres 1/1, rabbitmq 1/1








![alt text](<img/image copy 6.png>)


Pods: postgres-0, rabbitmq-0 (а не postgres-7f8c9-xxx как раньше!)







![alt text](<img/image copy 7.png>)


PVC: postgres-storage-postgres-0 (автосозданный)




Deployment с PVC для PostgreSQL — это антипаттерн. В DO12 я переработал манифесты: PostgreSQL и RabbitMQ теперь развёрнуты через StatefulSet с Headless Service. PVC автоматически создаётся через volumeClaimTemplates. На production-нагрузке это даст возможность безопасно масштабировать БД и собирать RabbitMQ-кластер. У нас всё ещё одна реплика — но архитектура готова к росту.





![alt text](<img/image copy 8.png>)

5/5  проходит, как и раньше — для приложения снаружи ничего не изменилось.










## То же самое для Helm (StatefulSet)

![alt text](<img/image copy 16.png>)

Шаблонизировал постгресс и раббит.


![alt text](<img/image copy 9.png>)

ls -la *.tgz — новый архив 0.2.0.



![alt text](<img/image copy 10.png>)

**заново создаю неймспейс**









![alt text](<img/image copy 11.png>)

Устанавл и проверил

Главное —  внимание на имена:

postgres-0 (раньше было postgres-f7896fb9d-xxx)
rabbitmq-0 (раньше было rabbitmq-746df59b5c-xxx)
Остальные сервисы — с хэш-суффиксами, как и были (потому что они остались Deployment'ами)











![alt text](<img/image copy 12.png>)

видны и StatefulSet (postgres, rabbitmq), и Deployment (остальные 7)






![alt text](<img/image copy 13.png>)

видны имена postgres-0, rabbitmq-0




![alt text](<img/image copy 14.png>)


виден автосозданный PVC с именем  postgres-storage-postgres-0






**Postman**

![alt text](<img/image copy 15.png>)

5/5   зелёные.
