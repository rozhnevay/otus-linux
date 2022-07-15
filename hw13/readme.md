# **Homework 13**
Подготовим Dockerfile, c кастомной index.html.
ENTRYPOINT/CMD не используем - используется дефолтный из базового образа
Запускаем build
```shell
docker build -t 17709729/nginx-custom:latest .
```
Результат
```
alexon@Aleksejs-MacBook-Pro hw12 % docker build -t 17709729/nginx-custom:latest .
[+] Building 13.2s (9/9) FINISHED                                                                                                                                           
 => [internal] load build definition from Dockerfile                                                                                                                   0.0s
 => => transferring dockerfile: 156B    
 ...
 alexon@Aleksejs-MacBook-Pro hw12 % docker images
REPOSITORY              TAG       IMAGE ID       CREATED              SIZE
17709729/nginx-custom   latest    839feeb9eb9c   About a minute ago   152MB
           
```
Запускаем контейнер
```shell
docker run -d -p 8080:80 17709729/nginx-custom
```
Результат:
```shell
alexon@Aleksejs-MacBook-Pro hw12 % docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED         STATUS         PORTS                  NAMES
b71307227c8a   17709729/nginx-custom   "/docker-entrypoint.…"   4 seconds ago   Up 4 seconds   0.0.0.0:8080->80/tcp   sharp_bhabha
```
Проверяем страницу
```shell
curl http://localhost:8080
```
```shell
alexon@Aleksejs-MacBook-Pro hw12 % curl http://localhost:8080
<html>
        <h1>Hello World!</h1>
</html>           
```
Видим, что наша страница на месте.
Остановим и убьем контейнер.
```shell
docker stop b7
docker rm b7 
```
Залогинимся и запушим образ в dockerhub
```shell
docker push 17709729/nginx-custom
```
Ссылка на образ: https://hub.docker.com/repository/docker/17709729/nginx-custom

### Определите разницу между контейнером и образом
Образ - это собранная версия приложения, состоящая из слоев. Каждая строка докерфайла - слой
Контейнер - запущенный экземпляр образа

### Можно ли в контейнере собрать ядро
Нет, так как контейнер запускается в user namespace, при этом используется ядро хост-системы.

