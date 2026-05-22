# Анализ эмоциональности (Sentiment Analysis) на Kubernetes

Лабораторная работа по курсу «Технологии интернет». Развертывание микросервисного приложения в кластере Kubernetes с фиксированной сетевой связкой.

## Архитектура приложения

Приложение состоит из трёх микросервисов, упакованных в Docker-контейнеры и управляемых Kubernetes:

| Микросервис | Язык/Стек | Функция | Внутренний порт |
|-------------|-----------|---------|-----------------|
| **sa-frontend** | React (Nginx) | Пользовательский интерфейс (ввод текста, отображение результата) | 80 |
| **sa-webapp** | Java (Spring Boot) | Сервер приложений, маршрутизация запросов | 8080 |
| **sa-logic** | Python (Flask + TextBlob) | Ядро анализа эмоциональной окраски текста | 5000 |

### Схема взаимодействия
Пользователь (Браузер) → [sa-frontend:80] → [sa-webapp:8080] → [sa-logic:5000] → ответ возвращается по цепочке

Все сервисы упакованы в Docker-контейнеры и управляются Kubernetes.

## Предварительные требования

- **Docker Desktop** (для сборки и запуска контейнеров)
- **Minikube** (локальный кластер Kubernetes)
- **kubectl** (клиент для управления кластером, устанавливается вместе с Minikube)
- **Node.js и npm** (для сборки React-приложения)
- **Аккаунт на Docker Hub** (для хранения образов)
- **Git** (для клонирования репозитория)

## Инструкция по запуску

### 1. Клонирование репозитория

git clone https://github.com/ВАШ_ЛОГИН/ИМЯ_РЕПОЗИТОРИЯ.git
cd ИМЯ_РЕПОЗИТОРИЯ

### 2. Сборка и публикация Docker-образов

Войдите в Docker Hub:

docker login -u ВАШ_DOCKER_ID

Все команды ниже выполняются из корня проекта. ВАШ_DOCKER_ID нужно заменить на ваш логин в Docker Hub (например, suchumichail).

#### SA-Logic (Python-анализатор)

cd sa-logic
docker build -f Dockerfile -t ВАШ_DOCKER_ID/sentiment-analysis-logic .
docker push ВАШ_DOCKER_ID/sentiment-analysis-logic

#### SA-WebApp (Java-сервер)

Из-за недоступности старых Java-образов используется готовая сборка автора с последующей перепривязкой к вашему аккаунту.

cd ..\sa-webapp
docker pull rinormaloku/sentiment-analysis-web-app
docker tag rinormaloku/sentiment-analysis-web-app ВАШ_DOCKER_ID/sentiment-analysis-web-app
docker push ВАШ_DOCKER_ID/sentiment-analysis-web-app

#### SA-Frontend (React-интерфейс)

cd ..\sa-frontend
npm install
npm run build
docker build -f Dockerfile -t ВАШ_DOCKER_ID/sentiment-analysis-frontend .
docker push ВАШ_DOCKER_ID/sentiment-analysis-frontend

### 3. Запуск кластера и развертывание

minikube start
cd ..\resource-manifests

#### Развертывание Python-сервиса
kubectl apply -f sa-logic-deployment.yaml
kubectl apply -f service-sa-logic.yaml

#### Развертывание Java-сервиса
kubectl apply -f sa-web-app-deployment.yaml
kubectl apply -f service-sa-web-app-lb.yaml

#### Развертывание React-фронтенда
kubectl apply -f sa-frontend-deployment.yaml
kubectl apply -f service-sa-frontend-lb.yaml

Проверьте, что все поды запущены:

kubectl get pods

Все поды должны быть в статусе Running.

### 4. Настройка связи фронтенда с бэкендом

В браузере мы используем фиксированный порт 8080, который пробрасывается к Java-сервису. Это гарантирует, что адрес не изменится после перезапуска Minikube.

#### 4.1. Обновите URL в коде фронтенда

В файле sa-frontend/src/App.js найдите строку с fetch и установите:

fetch('http://127.0.0.1:8080/sentiment', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ sentence: this.state.sentence })
})

#### 4.2. Пересоберите образ

cd ..\sa-frontend
npm run build
docker build -f Dockerfile -t ВАШ_DOCKER_ID/sentiment-analysis-frontend:v2 .
docker push ВАШ_DOCKER_ID/sentiment-analysis-frontend:v2

#### 4.3. Обновите манифест развертывания

В файле resource-manifests/sa-frontend-deployment.yaml замените строку с образом:

- image: ВАШ_DOCKER_ID/sentiment-analysis-frontend:v2

И примените изменения:

cd ..\resource-manifests
kubectl apply -f sa-frontend-deployment.yaml
kubectl rollout restart deployment sa-frontend

### 5. Запуск приложения

Для работы приложения нужно держать открытыми два окна терминала.

#### Терминал 1 (держать открытым всегда):

kubectl port-forward service/sa-web-app-lb 8080:80

Это свяжет порт 8080 на вашем компьютере с Java-сервисом внутри кластера.

#### Терминал 2:

minikube service sa-frontend-lb

Откроет браузер с вашим приложением.

### 6. Тестирование

Введите в открывшемся приложении любую фразу на английском языке и нажмите кнопку анализа.