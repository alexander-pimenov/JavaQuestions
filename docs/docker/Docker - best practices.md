# Docker - best practice
### Источник - видео:
[DOCKER BEST PRACTICE Для PYTHON Разработчика](https://youtu.be/7Wx0R8YHfh8)

## {{1}}
--- Используй базовые образы с точными версиями пакетами и Операционных Систем
```
#ПЛОХО
FROM python:3
```

```
#ХОРОШО
FROM python:3.11.5-slim-bookworm
```


## {{2}}
--- Файл requirements.txt/poetry.lock с привязанными версиями
#ПЛОХО
RUN python -m pip install django pands numpy

#ХОРОШО
COPY ./requirements.txt /requirements.txt
RUN python -m pip install -r requirements.txt
---------------------------------------------
# requirements.txt
django==4.2.1
numpy==1.25.2
pands==2.0.3


{{3}}
Количество слоев. Делай минимум слоев через объединение команд.
Каждая команда в DockerFile создает новый слой в Image.
И это увеличивает размер Image. Чтобы уменьшить количество слоёв
нужно объединять команды в цепочки.

#ПЛОХО
# создали 2 слоя
RUN apt update
RUN apt -y install htop

#ХОРОШО
# создали 1 слой
RUN apt update && apt -y install htop

# или так создали 1 слой - вы великолепны
RUN apt update && apt -y install \
	htop \
	tree \
	mc


{{4}}
Нужно почистить образ от лишних данных
#ПЛОХО 
Так мы создаем ещё один дополнительный слой без этих данных.
RUN rm -rf /usr/local/bundle/cache/*.pyc \
	&& apt-get clean \
	&& rm -rf /var/lib/apt/list/* /tmp/* /var/tmp/*
	
#ХОРОШО	
а так мы сразу почистим текущий слой
RUN curl -sL /usr/local/bundle/cache/*.pyc \
	&& apt-get -y install libpg-dev imagemagick gsfonts nodejs \
	&& bundle install --without development test --path vendor/bundle \
	&& rm -rf /usr/local/bundle/cache/*.pyc \
	&& apt-get clean \
	&& rm -rf /var/lib/apt/list/* /tmp/* /var/tmp/*

	
{{5}}
Используйте преимущество кэширования слоёв.
Чтобы ускорить сборку контейнера.
При запускке сборки Docker пытается использовать повторно слой 
из предыдущих запусков.
И если слой не изменился, то Docker берет слой из кэша сборки.
А если слой изменился, то этот слой и ВСЕ слой, которые идут после него,
должны быть заново созданы!!! И только предыдущие закэшируются.

#ПЛОХО
		LAYERS										Cache
FROM python:3.11.5-slim-bookword					V
WORKDIR /src										V
COPY . .											X
RUN python -m pip install -r requirements.txt		X
RUN что-то ещё										X
RUN что-то ещё										X
ENTRYPOINT [ "/bin/server" ]						X

Что бы увеличить количество слоёв, в которых ничего не изменяется, мы моежем
изменить их последовательность. Сначала поставить слой с загрузкой 
зависисмостей.И только потом копирование исходного кода в контейнер.

#ХОРОШО
		LAYERS										Cache
FROM python:3.11.5-slim-bookword					V
WORKDIR /src										V
COPY ./requirements.txt /requirements.txt			V
RUN python -m pip install -r requirements.txt		V
COPY . .											X
RUN что-то ещё										X
RUN что-то ещё										X
ENTRYPOINT [ "/bin/server" ]						X

Кэширование может привести к проблеме небезопасныйх образов,
это значит, что мы не будем получать обновление безопасности
для системных пакетов. Поэтому неоходимо пересобирать переодически образ
с нуля, чтобы обеспечить обновление систем безопасности.
Например, раз в месяц, пересобирать образ с помощью:

docker build --pull --no-cache

{{6}}
Не используйте ROOT.
По умолчанию контейнер Docker запускается от имени root.
Что представляет угрозу безопасности.

Нужно запускать update и upgrade от root, а потом нужно 
изменить пользователя на не root. И потом производим все действия
уже не от админ пользователя.

#ХОРОШО

FROM debian:booster
# запускаем от root
RUN apt-get update && apt-get -y upgrade

# меняем на не root
RUN useradd --create-home appuser
WORKDIR /home/appuser
USER appuser
ENTRYPOINT ["whoami"]

{{7}}
Используйте оптимизированные базовые образы.


{{8}}
Используйте мульти-стедж для уменьшения размера
и времени сборки контейнера.
Много-этапные (multi-stage builds) сборки.
Отделить стадию сборки от стадии выполнения.
Т.е. исключить лишние зависимости сборки из
образа. При этом оставиви их доступными во время
сборки образа за счет multi-stage.
Можно копировать необходимые ресурсы из одной стадии в 
другую, а значит лишние файлы остануться в промежуточной
сборке и не попадут в финальную.
Но делаем мы всё это в одном Dockerfile и это позволяет проводить
этапы сьорки параллельно.

# Dockerfile
# build stage
FROM buildbase as build
...
...
...

# production ready stage
FROM runbase
...
COPY --from=build
/artifact /app

Этот проет не оптимизирован, и весит много и работает не очень здорово.
#было
FROM python:3.11.5-slim-bookword

RUN curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.ru | python
ENV PATH /root/.poetry/bin:$PATH
WORKDIR /app
RUN python -m venv /app/venv
COPY pyproject.toml poetry.loc ./
RUN . /app/venv/bin/activate && poetry install
ENV PATH /app/venv/bin:$PATH
COPY . ./
CMD ["gunicorn", "blog.wsgi", "-b 0.0.0.0:8080", "--log-file", "-"]

Переписываем под мульти-стедж с двумя стадиями:
#стало
# 1 стадия - стадия зависимости и называем её venv
FROM python:3.11.5-slim-bookword
ENV PATH /app/venv/bin:$PATH
WORKDIR /app
COPY pyproject.toml poetry.loc ./
RUN python -m venv --copies /app/venv
RUN . /app/venv/bin/activate && poetry install

# 2 стадия, просто копируем собранные зависимости
FROM python:3.11.5-slim-bookword
# просто копируем уже готовую папку, которая была сделана в 1 стадии
COPY --from=venv /app/venv /app/venv
ENV PATH /app/venv/bin:$PATH
WORKDIR /app
COPY . ./
CMD ["gunicorn", "blog.wsgi", "-b 0.0.0.0:8080", "--log-file", "-"] 

Тут важно не сломать кэширование.


{{9}}
Используйте якоря [&] и псевдонимы [<<:*] для повторяющихся
блоков docker compose. Т.е. избавляемся от дублирования в docker compose.
якоря - yml функция, которая позволяет помечать элементы, а 
потом ссылаться на них в других местах файла.

-----------------------------------------|
version: "3.4"                           |
										 |
services:                                |
  web:                                   |
    build:                               |
	  context: "."                       |
	  args:                              |
	    - "APP_ENV=${APP_ENV:-prod}"     |
	depends_on:                          |
	  - "postgres"                       |
	  - "redis"                          |
	env_file:                            |
	  - ".env"                           |
	image: "myapp"                       |
	restart: "unless-stoped"             |
	stop_grace_period: "3s"              |
	volumes:                             |
	  - ".:/app"                         |
	posts:                               |
	  - "8003:8080"                      |
  web2:                                  |
      web:                               |
        build:                           |
    	  context: "."                   |
    	  args:                          |
    	    - "APP_ENV=${APP_ENV:-prod}" |
    	depends_on:                      |
    	  - "postgres"                   |
    	  - "redis"                      |
    	env_file:                        |
    	  - ".env"                       |
    	image: "myapp"                   |
    	restart: "unless-stoped"         |
    	stop_grace_period: "3s"          |
    	volumes:                         |
    	  - ".:/app"                     |
    	posts:                           |
    	  - "8004:8080"                  |
										 |
-----------------------------------------|

А теперь с использованием якоря:
просто указываем ссылку на наш   <<: *default-app
а что нужно изменить, то явно указываем, тут например, 
это порт - "8004:8080"
Т.е. полностью берем, то что было в web контейнере, но
заменяем порт.

---------------------------------------|
version: "3.4"                         |
									   |
services:                              |
  web: &default-app                    |
    build:                             |
	  context: "."                     |
	  args:                            |
	    - "APP_ENV=${APP_ENV:-prod}"   |
	depends_on:                        |
	  - "postgres"                     |
	  - "redis"                        |
	env_file:                          |
	  - ".env"                         |
	image: "myapp"                     |
	restart: "unless-stoped"           |
	stop_grace_period: "3s"            |
	volumes:                           |
	  - ".:/app"                       |
	posts:                             |
	  - "8003:8080"                    |
  web2:                                |
      <<: *default-app                 |
      posts:                           |
        - "8004:8080"                  |
---------------------------------------|		

{{10}}
Используйте файл .dockerignore
В образ не должны попадать какие-то файлы, например, логи, ключи.
Это аналог файла .gitignore
Пример:
----------------|
__pycache__/    |
				|
**/migrations   |
src/media       |
src/db.sqlite3  |
Procfile        |
.git            |
----------------|


{{11}}
Используйте hadolint.
Это линтер для Dockerfile-ов.
Помагает создавать образы Docker по всем канонам best practices.

Ссылка на него:
https://hadolint.github.io/hadolint/


Умный линтер Dockerfile, который поможет вам создавать лучшие 
образы Docker . Линтер анализирует Dockerfile в AST и выполняет 
правила поверх AST. Он также использует знаменитую Shellcheck 
для проверки кода Bash внутри RUN инструкций.

Пример:
--------------------------------------------------------------------------------------------------|
# Always tag the version of an image explicitly                                                   |
FROM debian                                                                                       |
																								  |
# node_verion is referenced but not assigned (did you mean 'nod_version'?).                       |
# Delete the apt-get lists after installing something                                             |
# Avoid additional packages by specifying `--no-install-recommends`                               |
RUN export node_version="0.10" \                                                                  |
&& apt-get update && apt-get -y install nodejs="$node_verion"                                     |
COPY package.json usr/src/app                                                                     |
																								  |
# Use WORKDIR to switch to a directory                                                            |
# Pin versions in npm. Instead of `npm install <package>` use `npm install <package>@<version>`   |
RUN cd /usr/src/app \                                                                             |
&& npm install node-static                                                                        |
																								  |
# Valid UNIX ports range from 0 to 65535                                                          |
EXPOSE 80000                                                                                      |
CMD ["npm", "start"]                                                                              |
--------------------------------------------------------------------------------------------------|

Либо его нужно установить. и тогда все проверки осуществляются в одну строчку.

{{12}}
Используйте разные docker-compose файлы.
Это антипаттерн, но он очень удобен, если нужно менять приложение под разные среды.

Docker-compose по умодчанию читает два файла: docker-compose.yml и docker-compose.override.yml.

Пример:
-----------------------------------|
--docker-compose.yml--             |
								   |
web:                               |
  image: example/my_web_app:latest |
  depends_on:                      |
    - db                           |
	- cache                        |
								   |
db:                                |
  image: postgres:latest           |
								   |
cache:                             |
  image: redis:latest              |
-----------------------------------|

# В этом примере конфигурация разработки предоставляет доступ 
# к некоторым портам хоста, монтирует наш код в виде тома и создает
# веб-образ:
--------------------------------|
--docker-compose.override.yml-- |
                                |
web:                            |
  build:                        |
  volumes:                      |
    - '.:/code'                 |
  ports:                        |
    - 8083:80                   |
  environment:                  |
	DEBUG: 'true'               |
					            |
db:                             |
  command: '-d'                 |
  ports:                        |
    - 5432:5432                 |
					            |
cache:                          |
  ports: 6379:6379              |
--------------------------------|

--docker-compose.prod.yml--
-----------------------|
web:                   |
  ports:               |
    - 80:80            |
  environment:         |
	PRODUCTION: 'true' |
					   |
cache:                 |
  environment:         |
	TITL: '500'        |
-----------------------|

# Для деплоя с этим production Compose file вы должны запустить docker с 
ключами -f таким образом:
-------------------------------------------------------------------------|
$  docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d |
-------------------------------------------------------------------------|

Так же можно создавать БАЗОВЫЙ config - docker-compose.base.yml и потом переопределять
некоторые участки, в зависисмости от требований.
Например, 
-----------------------------------|
--docker-compose.yml--             |
								   |
web:                               |
  image: example/my_web_app:latest |
  depends_on:                      |
    - db                           |
								   |
db:                                |
  image: postgres:latest           |
-----------------------------------|

А в docker-compose.admin.yml добавить новый сервис для запуска базы данных,
экспорта или резервного копирования:
-----------------------------------|
--docker-compose.admin.yml--       |
								   |
dbadmin:                           |
  build: database_admin/           |
  depends_on:                      |
    - db                           |
-----------------------------------|

Для запуска с нормальным окружением:
------------------------|
$  docker compose up -d |
------------------------|

а для запуска резервного копирования базы данных так же 
включите команду:
-----------------------------------------------------------------------|
$  docker compose -f docker-compose.yml -f docker-compose.admin.yml \  |
   run  dbadmin db-dackup                                              |
-----------------------------------------------------------------------|

Еще рассмотри пример. Есть стандартный docker-compose.yml
--------------------------------|
--docker-compose.yml--          |
                                |
web:                            |
  build:                        |
  volumes:                      |
    - '.:/code'                 |
  ports:                        |
    - 8883:80                   |
  environment:                  |
	DEBUG: 'False'              |
					            |
db:                             |
  command: '-d'                 |
  ports:                        |
    - 5432:5432                 |
					            |
cache:                          |
  ports:                        |
    - 6379:6379                 |
--------------------------------|

и для локальной разработки нам нужно заменить порт и environment.
Создаем docker-compose.override.yml со следующим содержимым:
--------------------------------|
--docker-compose.override.yml-- |
                                |
web:                            |
  ports:                        |
    - 8001:80                   |
  environment:                  |
	DEBUG: 'true'               |
--------------------------------|

И теперь при зауске 
`docker compose up` 
docker возмет всё из docker-compose.yml и заменит те строчки, 
которые есть в docker-compose.override.yml


{{13}}
Всегда обновляйте менеджер пакетов.
Потому что образный pip устареет и откажется с вами сотрудничать

----------------------------------|
RUN . venv/bin/activate \         |
   && pip install --upgrade pip   |
----------------------------------|
 
{{14}}
`Depends_on` - не панацея, поэтому используйте `Wait for it`
`Depends_on` говорит чтобы запускали указанный контейнер толькопосле 
тоже указанного контейнера:
--------------------------------|
--docker-compose.yml--          |
                                |
web:                            |
  build:                        |
  volumes:                      |
    - '.:/code'                 |
  ports:                        |
    - 8883:80                   |
  environment:                  |
	DEBUG: 'False'              |
  depends_on:                   |
    - db                        |
					            |
db:                             |
  command: '-d'                 |
  ports:                        |
    - 5432:5432                 |
					            |
cache:                          |
  ports:                        |
    - 6379:6379                 |
--------------------------------|

Но если нам нужно что бы БД еще и начала принимать запросы или накатила
миграцию, то `depends_on` не хватит и нужно использовать `wait-for-it`.

---------------------------------------------------|
--docker-compose.yml--                             |
services:                                          |
  web:                                             |
    build: .                                       |
    ports:                                         |
      - "80:8000"                                  |
    depends_on:                                    |
      - db                                         |
    entrypoint: ["./wait-for-it.sh", "db:5432" ]   |
                                                   |
  db:                                              |
    image: postgres                                |
                                                   |
---------------------------------------------------|

`wait-for-it` будет проверять хосты и порты до тех пор пока оно не будет подтверджено.
`wait-for-it.sh` можно скачать в интернете либо использовать отдельный модуль.

{{15}}
Думай над флагами запуска контейнера.
Для конфигурации политики перезапуска контейнера используй флаг `--restart`
когда используешь команду `docker run`.
Значения для флага могут быть следующимими:
no                       --- не перезапускать автоматически контейнер.
on-failure[:max-retries] --- перезапустить контейнер,е сли он существуеь, из-за ошибки в виде ненулевого кода возврата. Так же можно указать колиество перезапусков.
always                   --- всегда запускайте контейнер если он остановился. Если он остановлен вручную, то он перезапустится только при перезапуске Docker Diamond или самого контейнера вручную.
unless-stoped            --- 




https://youtu.be/7Wx0R8YHfh8?t=1013




 