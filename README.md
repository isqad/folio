# Folio

В этом кратком руководстве я хотел бы поделиться с сообществом своим опытом в постройке веб-приложения на языке Clojure.

# Создание веб-приложения на Clojure

## Ring

[Ring](https://github.com/ring-clojure/ring) - это маленькая clojure-библиотека, помогающая в написании веб-приложений. Ее создание навеяно такими знаменитыми 
библиотеками как WSGI для Python и Rack для Ruby. Концепция ring-приложения схожа с rack. У нас есть request - запрос, есть handler - обработчик запроса и 
response - HTTP-ответ. Request и response - это просто clojure map'ы. handler - это функция, аргументом которой является request, а возвращаемым значением response.
handler единственный на все приложение, это другим словами точка входа.
Для обработки response перед отправкой его в сеть клиенту (например, для указания дополнительных заголовков ответа) существует такое понятие как midleware - 
функция высшего порядка, аргументом которой является handler. Midleware в ring похожи на прослойки в rack. Приведу пример самого простого приложения на ring:

Сгенерируйте новый проект:

```bash
$ lein new hello-world
```

Укажите в зависимостях проекта ring-core и ring-jetty-adapter:

```clojure
; project.clj

(defproject hello-world "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.8.0"]
                 [ring/ring-core "1.4.0"]
                 [ring/ring-jetty-adapter "1.4.0"]])
```

Тем самым мы говорим, что наш проект зависит от библиотеки ring-core и адаптера ring-jetty-adapter для java-веб-сервера Jetty. 
Адаптер служит посредником между вашим приложением  и сервером при обмене запросами-ответами.

```clojure
; src/hello-world/handler.clj 
(ns hello-world.handler)

(defn app
  [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body "Hello world!"})
```

Эта единственная функция и есть handler, ее достаточно для мини веб-приложения hello world. Возращаемое значение функции является map, обязательными ключами которого 
являются ```:status``` - код ответа: 200, 404, 500 и т.п. ```:headers``` - дополнительные заголовки ответа. Значением ```:headers``` также является map,
ключ-значения которого обычные http-заголовки. ```:body``` - само тело ответа, которое может быть типа String, File или ISeq.

Испытаем наше первое приложение. Для этого зайдем в repl:

```bash
$ lein repl
```

```clojure
(use 'ring.adapter.jetty)
(use 'hello-world.handler)

; запускаем jetty-сервер, указав первым аргументом наш handler, а вторым в виде map порт сервера
(run-jetty app {:port 3000})
```

Далее откроем браузер по адресу http://localhost:3000, и увидим наше приветствие.

## lein-ring

Для того, чтобы наделить lein функционалом запуска вашего веб-приложения существует плагин [lein-ring](https://github.com/weavejester/lein-ring).
Целью данного плагина является автоматизация рутинных задач для работы с ring, таких как запуск веб-сервера и указанию ring точки входа - то есть, handler'а.
Дополним наше hello-world приложение плагином lein-ring:

```clojure
; project.clj

(defproject hello-world "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.8.0"]
                 [ring/ring-core "1.4.0"]
                 [ring/ring-jetty-adapter "1.4.0"]]
  :plugins [[lein-ring "0.9.7"]]
  ; укажем handler:
  :ring {:handler hello-world.handler/app})
```

А теперь запустим наше приложение:

```bash
$ lein ring server
```

## Добавляем логирование: ring-logger

Всякое приложение не обходится без логирования запросов. В помощь в разработке существует библиотека [ring-logger](https://github.com/nberger/ring-logger). 
Я буду использовать ring-logger с бэкендом timbre [ring-logger-timbre](https://github.com/nberger/ring-logger-timbre).
Подключить данную библиотеку к нашему приложению не составит большого труда:

```clojure
; project.clj

(defproject hello-world "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.8.0"]
                 [ring/ring-core "1.4.0"]
                 [ring/ring-jetty-adapter "1.4.0"]
				 [ring-logger-timbre "0.7.5"]]
  :plugins [[lein-ring "0.9.7"]]
  :ring {:handler hello-world.handler/app})
  
```

Чтобы задействовать в приложении обернем handler в midleware wrap-with-logger:

```clojure

; src/hello-world/handler.clj

(ns hello-world.handler
  (:require [ring.logger.timbre :as logger.timbre]))

(defn handler
  [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body "<h1>Hello Julia!</h1>"})

(def app
  (logger.timbre/wrap-with-logger handler))
  
```

Запускаем и в консоли можно увидеть, как логируется каждый запрос:

[!screen](https://i.imgur.com/ZvIQOn9.png)

## Отдаем статические файлы: css,js и т.п.

Многие веб-приложения имеют красиво оформленную страничку. Всю красоту обычно наводят с помощью css, html, JavaScript и картинок. Для начала нам необходимо
научиться отдавать статичные файлы клиенту. Определимся, где будем хранить всю статику. Обычно для ring-приложения это директория resources/public. 
Пойдем по стандартному пути: создадим директории: ```$ mkdir -p resources/public/css resources/public/js resources/public/images```.
В качестве примера скачаем базовые стили для responsive веб-приложений [skeleton.css](http://getskeleton.com/). Помещаем ```skeleton.css``` и 
```normalize.css``` в директорию resources/public/css. Мы будем ожидать, что наши css-файлы будут доступны по адресу ```localhost:3000/css/normalize.css```.
Аналогично для картинок, js и вообще для всего, что положим в resources/public. Добавим midleware для обслуживания статики в наше приложение:

```clojure
; src/hello-world/handler.clj

(ns hello-world.handler
  (:require [ring.logger.timbre :as logger.timbre]
             ; "подключаем" middleware для статики
            [ring.middleware.resource :refer :all]))

(defn handler
  [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body "<h1>Hello Julia!</h1>"})

; "->" - это такой удобный макрос, "синтаксический сахар" clojure.
; Он "преобразует" такую конструкцию в более читабельный вид:
; (wrap-resource (logger.timbre/wrap-with-logger handler))
; при этом стоит учесть, что такая "одинарная" стрелка
; "прокидывает" handler в первый аргумент последующих форм.
; Существует еще макрос ->> , подробности ищите в документации clojure.
(def app
  (-> handler
      logger.timbre/wrap-with-logger
      (wrap-resource "public")))

```	  

Сейчас можно запустить сервер и открыть в браузере ```localhost:3000/css/normalize.css```. Вы увидете исходный текст css стилей.
