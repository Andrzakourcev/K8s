# K8s

# Лабораторная работа 4

# Техническое задание
Поднять kubernetes кластер локально (например minikube)
В нём развернуть свой сервис, используя 2-3 ресурса kubernetes.
В идеале разворачивать кодом из yaml файлов одной командой запуска.
Показать работоспособность сервиса.


# Теория 

Kubernetes (сокращённо K8s) — это система оркестрации контейнеров с открытым исходным кодом, разработанная Google и сейчас поддерживаемая Cloud Native Computing Foundation (CNCF). Kubernetes автоматизирует развертывание, масштабирование и управление контейнеризованными приложениями.

Kubernetes решает проблемы, возникающие при эксплуатации большого количества контейнеров:

1. Автоматическое масштабирование (вверх/вниз)
2. Самоисцеление (перезапуск упавших контейнеров)
3. Балансировка нагрузки
4. Обновление без простоя (rolling updates)
5. Управление конфигурациями и секретами

Архитектура Kubernetes

Master Node (Узел управления):

- API Server – точка входа для всех команд и REST-запросов.
- Scheduler – выбирает, на каком узле запустить Pod.
- Controller Manager – следит за состоянием кластера.
- etcd – распределённое хранилище конфигурации и состояния.

Worker Nodes (Узлы-исполнители):

- kubelet – агент, управляющий Pod'ами на узле.
- kube-proxy – сетевой прокси и балансировщик.
- Container runtime – например, Docker или containerd.

# Что получилось у меня

Это история о том, как небольшое задание на полчаса превратилось в безвылазную ночь за ноутом. Итак, поехали)

# 1. Подготовка
Для начала устанавливаем всё необходимое - docker, minikube, kubectl. 

*minikube - это инструмент для локального развертывания однодоузлового (single-node) кластера Kubernetes.*
*kubectl — это CLI-утилита для управления кластерами Kubernetes.* 

Теперь можно запустить minikube:

`minikube start driver=docker`

Запускаем с дравером docker'а. 

![image](https://github.com/user-attachments/assets/2487024d-be13-484b-a6cc-b15fa286be8d)

После подготовим в домашеней директории файлы, которые необходимы для запуска. 

# Создаем файл app.py
Этот файл создает простое FastAPI-приложение, которое возвращает JSON-ответ при обращении к корневому пути.  

`
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello from FastAPI in Kubernetes!"}
`
# requirements.txt

Устанавливаем зависимости

`
fastapi==0.109.1
uvicorn==0.27.0
`
Хорошей практикой будет прописывать версии, чтобы было легче отлаживать и откатываться при изменении.

# Теперь необходимо написать Dockerfile

`
FROM python:3.11-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
`

Здесь все также предельно понятно. Подтягиваем базовый образ python:3.11-slim из registry,берем облегченную версию, чтобы не утяжелять образ. Рабочей директорией выбираем заранее созданную /app. После копируем из нее все файлы в рабочую директрию контейнера. Устанавливаем зависимости с удалением кэша pip и запускаем unicorn сервер на 8000 порту. 

# Загрузка образа в minikube 

Тут есть 2 варианта

Первый - создаем докер образ как обычно:
`
docker build -t my-fastapi:0.109.1 .
`
А после загружаем его в minikube:
`
minikube image load my-fastapi:0.109.1
`
Второй - собрать образ сразу в среде minikube:
`
minikube image build -t my-fastapi:0.109.1 .
`

После проверяем попал ли образ в minikube:
`
minikube image ls #Покажет список образов в minikube
`
Теперь можно создавать yaml-манифесты, для описания жизненного цикла pod'ов. 

# Deployment.yaml

`
apiVersion: apps/v1  # Версия API Kubernetes для Deployment
kind: Deployment     # Тип ресурса (развертывание)
metadata:
  name: my-fastapi-deployment  # Имя Deployment
spec:
  replicas: 2        # Количество подов приложения
  selector:
    matchLabels:
      app: fastapi    # Как находить поды для управления 
  template:          # Шаблон пода
    metadata:
      labels:
        app: fastapi  # Метка пода 
    spec:
      containers:
      - name: fastapi  # Имя контейнера внутри пода
        image: my-fastapi:latest  # Docker-образ 
        ports:
        - containerPort: 8000  # Порт, который слушает контейнер
`
Deployment следит за состоянием подов:
Прописываем порт, образ, имя, метку, а также количество подов, я взял 2.
Если один под упадет — Deployment автоматически создаст новый.
Также при обновлении образа  можно выполнить rolling update без downtime.
Поды получат метку app: fastapi — это позволит связать их с Service. 

# Теперь service.yaml

`
apiVersion: v1          # Версия API Kubernetes для Service
kind: Service           # Тип ресурса (сервис)
metadata:
  name: fastapi-service # Имя сервиса
spec:
  selector:
    app: fastapi        # Связь с подами, у которых есть label app: fastapi
  type: NodePort        # Тип сервиса (открывает порт на нодах кластера)
  ports:
    - protocol: TCP     # Протокол передачи данных
      port: 80          # Порт, на котором сервис доступен внутри кластера
      targetPort: 8000  # Порт контейнера (должен совпадать с портом контейнера в Deployment)
`

Этот манифест создает точку доступа к моим FastAPI-подам, балансирует нагрузку между всеми подами с меткой app: fastapi. Открывает порт 80 на нодах кластера внутри кластера, снаружи — через случайный порт в диапазоне 30000-32767.

# Запуск

Теперь все готово. Применяем манифесты:
`
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
`

И смотрим поднялись ли наши pod'ы. 
`
kubectl get pods
`
Казалось бы все шло хорошо, но...

![image](https://github.com/user-attachments/assets/d12610fc-88a4-487a-b6c0-43c510d83477)
(Скриншот был взят вообще с другой VM, но суть ошибки в общем то одинаковая)

ErrImagePull - образ не был виден внутри minikube. После перепроверки загрузил все по новой и вроде бы решилось, но... новая ошибка
`
NAME                                     READY   STATUS         RESTARTS   AGE
my-fastapi-deployment-55d75666b6-427wz   0/1     ErrImagePull   0          64s
my-fastapi-deployment-55d75666b6-bppsx   0/1     ErrImagePull   0          64s
`
ImagePullBackOff и ErrImagePull - этот статус означает, что модуль не удалось запустить, поскольку он попытался получить образ контейнера из реестра с ошибкой. Модуль отказывается запускаться, потому что он не может создать один или несколько контейнеров, определенных в его манифесте.

Я начал смотреть логи и describe pod'ов.
`
kubectl describe pod pod-name
kubectl logs pod-name
`

Ничего дельного я там не нашел 
`
Containers:
  fastapi:
    Container ID:   
    Image:          my-fastapi:latest
    Image ID:       
    Port:           8000/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ErrImagePull
    Ready:          False
    Restart Count:  0
`
Данное предупреждение меня сбило, ведь все разрешения были.
`
  Warning  Failed     23s (x4 over 117s)  kubelet            Failed to pull image "my-fastapi": Error response from daemon: pull access denied for my-fastapi, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
`

После долгих и упорных попыток пересоздания, загрузок образов по новой, чтения логов и гуглешки я решил сделать все заново и радикально пошел на вторую VM :))
Там я сразу обнаружил, что для докера нет места на диске. Надо было расширить его. 

Захожу в работы с вирутальными дисками VirtualBox. Динамически его расширяю до 30 гб.

![image](https://github.com/user-attachments/assets/186cc6d0-3872-4c49-b7df-6138ef88bc97)

Теперь нужно увеличить разделы в гостевой ОС и расширить саму ОС. Просмотреть разделы можно - `sudo lsblk`, а также `df -h`. С помощью команды `sudo growpart /dev/sda 1` можно увеличить размер парта. Теперь появилась ещё одна проблема:

![image](https://github.com/user-attachments/assets/37be373c-5431-4812-8435-08833d0f03f8)

Я не мог слить вместе свободное дисковое пространство и dev/sda1, потому что между ними были ещё разделы sda2, sda5 и свободное место, а это разделы подкачки и extended раздел. Поэтому я решил... просто удалить их и создать заново.

Я создавал файл, а не раздел подкачки. Так мне казалось проще:

1. Проверяем текущий swap
   `
free -h
sudo swapon --show
   `
2. Cоздаем файл подкачки
   `sudo fallocate -l 2G /swapfile`
3. Права, форматирование и активация
   `sudo chmod 600 /swapfile
    sudo mkswap /swapfile
   sudo swapon /swapfile
   `
4. Редактирование /etc/fstab
   `echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab`

Это было небольшое отступление. Я все таки установил docker. Что же по итогу? Все также ImagePullBackOff. Тут у меня сдали нервы и я ушел спать. На следующий день я ещё раз все перепроверил, почитал логи и после запуска тестового pod'а мне наконец-то выдало ошибку:
`
ERROR:    Error loading ASGI app. Could not import module "main".
`

На этом моменте я успытал бурю эмоций. Все оказалось так просто и так долго. Дело в том, что unicorn ошидал файл main.py, который был прописан в Dockerfile, но мой файл с приложением назывался app.py
Делаем `mv app.py main.py`, пересобираем образ, применяем манифесты, удаляем запущенные поды `kubectl delete pod pod-name`, чтобы заново поднялись и о чудо все запустилось.

![image](https://github.com/user-attachments/assets/8b01a4c0-b9d4-4f7e-bff9-63de53c9259f)

После смотрим на url `minikube service my-fastapi-service --url` и видим что все работает:

![image](https://github.com/user-attachments/assets/5d944fef-7543-464b-adfc-16017728efc9)


После всего я также создал sh скрипт, чтобы пересобирать образ и применять манифесты автоматически, чтобы не делать это руками. Автоматизация - это наше все.

![image](https://github.com/user-attachments/assets/eabb6d3a-94e2-4911-b353-92b74a072add)

В заключение хочется сказать, что это лаба многому меня научила, от уставновки minikube до расширения дисков в системе. Я научился поднимать k8s кластер, следить за состоянием подов, отлаживать ошибки, читая логи. Все это поможет мне ещё не один раз. Спасибо за прочтение!
