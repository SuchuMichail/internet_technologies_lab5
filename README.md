# Анализ эмоциональности (Sentiment Analysis) на Kubernetes

Лабораторная работа по курсу «Технологии интернет». Развертывание микросервисного приложения в кластере Kubernetes.

## Архитектура приложения

Приложение состоит из трёх микросервисов:

| Микросервис | Язык | Назначение |
|-------------|------|------------|
| **sa-frontend** | React (JavaScript) | Пользовательский интерфейс. Принимает текст от пользователя и отображает результат анализа. |
| **sa-webapp** | Java (Spring Boot) | Сервер приложений. Принимает запросы от фронтенда, передаёт их на анализ и возвращает результат. |
| **sa-logic** | Python (Flask + TextBlob) | Ядро анализа. Определяет эмоциональную окраску текста (позитивная, негативная, нейтральная). |

### Схема взаимодействия
Пользователь → [sa-frontend:80] → [sa-webapp:8080] → [sa-logic:5000] → ответ возвращается обратно

Все сервисы упакованы в Docker-контейнеры и управляются Kubernetes.

## Предварительные требования

Для локального запуска потребуются:

- **Docker Desktop** — для сборки и запуска контейнеров
- **Minikube** — для локального кластера Kubernetes
- **kubectl** — для управления кластером (устанавливается вместе с Minikube)
- **Node.js и npm** — для сборки React-приложения
- **Аккаунт на Docker Hub** — для хранения образов
- **Git** — для клонирования репозитория

## Быстрый старт

### 1. Клонирование репозитория

git clone https://github.com/ВАШ_ЛОГИН/ИМЯ_РЕПОЗИТОРИЯ.git
cd ИМЯ_РЕПОЗИТОРИЯ

### 2. Сборка и публикация Docker-образов
Войдите в Docker Hub:

docker login -u ВАШ_DOCKER_ID

cd sa-logic
docker build -f Dockerfile -t ВАШ_DOCKER_ID/sentiment-analysis-logic .
docker push ВАШ_DOCKER_ID/sentiment-analysis-logic

cd ..\sa-webapp
docker pull rinormaloku/sentiment-analysis-web-app
docker tag rinormaloku/sentiment-analysis-web-app ВАШ_DOCKER_ID/sentiment-analysis-web-app
docker push ВАШ_DOCKER_ID/sentiment-analysis-web-app


cd ..\sa-frontend
npm install
npm run build
docker build -f Dockerfile -t ВАШ_DOCKER_ID/sentiment-analysis-frontend .
docker push ВАШ_DOCKER_ID/sentiment-analysis-frontend

### 3. Запуск Kubernetes-кластера

minikube start


### 4. Развертывание в Kubernetes

Все манифесты находятся в папке resource-manifests.

cd ..\resource-manifests

#### 4.1. Развертывание Python-сервиса (sa-logic)
kubectl apply -f sa-logic-deployment.yaml
kubectl apply -f service-sa-logic.yaml

#### 4.2. Развертывание Java-сервиса (sa-webapp)
kubectl apply -f sa-web-app-deployment.yaml
kubectl apply -f service-sa-web-app-lb.yaml

#### 4.3. Развертывание React-сервиса (sa-frontend)
kubectl apply -f sa-frontend-deployment.yaml
kubectl apply -f service-sa-frontend-lb.yaml

Проверьте, что все поды запущены:

kubectl get pods

Все поды должны быть в статусе Running.


### 5. Настройка и обновление фронтенда
На этом этапе фронтенд запущен, но ещё не знает адрес Java-сервиса. Нужно получить URL и обновить образ.

minikube service sa-web-app-lb

Оставьте этот терминал открытым! Туннель работает только пока процесс активен.

Скопируйте URL из вывода (например, http://127.0.0.1:65045).

В файле sa-frontend/src/App.js найдите строку:

`fetch('http://192.168.99.100:31691/sentiment', {`

Замените URL на скопированный адрес (сохраняя /sentiment в конце):

`fetch('http://127.0.0.1:65045/sentiment', {`

В новом окне терминала:

cd ..\sa-frontend
npm run build
docker build -f Dockerfile -t ВАШ_DOCKER_ID/sentiment-analysis-frontend:minikube .
docker push ВАШ_DOCKER_ID/sentiment-analysis-frontend:minikube

В файле resource-manifests/sa-frontend-deployment.yaml замените строку:

- image: rinormaloku/sentiment-analysis-frontend

на:

- image: ВАШ_DOCKER_ID/sentiment-analysis-frontend:minikube

Примените изменения:

cd ..\resource-manifests
kubectl apply -f sa-frontend-deployment.yaml

Дождитесь обновления подов:

kubectl get pods -w

### 6. Открытие приложения

minikube service sa-frontend-lb

Введите любую фразу на английском языке в появившемся окне браузера и нажмите кнопку анализа.