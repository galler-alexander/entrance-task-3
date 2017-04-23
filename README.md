# Задание 3

Мобилизация.Гифки – сервис для поиска гифок в перерывах между занятиями.

Сервис написан с использованием [bem-components](https://ru.bem.info/platform/libs/bem-components/5.0.0/).

Работа избранного в оффлайне реализована с помощью технологии [Service Worker](https://developer.mozilla.org/ru/docs/Web/API/Service_Worker_API/Using_Service_Workers).

Для поиска изображений используется [API сервиса Giphy](https://github.com/Giphy/GiphyAPI).

В браузерах, не поддерживающих сервис-воркеры, приложение так же должно корректно работать, 
за исключением возможности работы в оффлайне.

## Структура проекта

  * `gifs.html` – точка входа
  * `assets` – статические файлы проекта
  * `vendor` –  статические файлы внешних библиотек
  * `service-worker.js` – скрипт сервис-воркера

Открывать `gifs.html` нужно с помощью локального веб-сервера – не как файл. 
Это можно сделать с помощью встроенного в WebStorm/Idea веб-сервера, с помощью простого сервера
из состава PHP или Python. Можно воспользоваться и любым другим способом.

## Исследование

1. Выявлено, что у service worker неверный scope, из-за чего он не работает. Скорее всего проблема возникла из-за рефакторинга под названием «Разложить файлы красиво». Состояние service worker и scope можно посмотреть в Firefox: Chrome Developer Tools -> Service Workers и Google Chrome: DevTools -> Application -> Service Workers. Исправлено в https://github.com/galler-alexander/entrance-task-3/commit/7532ab190e0f3ae6dcef29e36340042da88052ef
2. Страница gifs.html не попадает в кэш, из-за этого не работает offline режим. Состояние service worker и scope можно посмотреть в Firefox: Chrome Developer Tools -> Storage -> Cache Storage и Google Chrome: DevTools -> Application -> Cache Storage. Исправлено в https://github.com/galler-alexander/entrance-task-3/commit/3ab35d51d3d5194a060c0091fbe28fa095c3be96 
3. Подтверждена проблема, что стало невозможно обновить HTML-страницу: у клиентов она стала браться из кеша не только в офлайн-режиме, а всегда. Скорее всего проблема возникла после рефакторинга под названием «Более надёжное кеширование на этапе fetch». Исправлено в https://github.com/galler-alexander/entrance-task-3/commit/68862752ec5cae81855dc3a53e1f0cf9f0559e81
4. Реализована возможность переключения в офлайн-режим после первого же запроса, а не после второго, как это происходило в работающем приложении до всех рефакторингов. Добавлено кэширование ресурсов приложения на этапе "install" service worker. Реализовано в https://github.com/galler-alexander/entrance-task-3/commit/1334d420eaf1c7887b4d711a7d0da031962332bb
5. Также выявлено, что в service worker нет обработчика события favorite:remove. Реализовано в https://github.com/galler-alexander/entrance-task-3/commit/3142905e1b5ea19ba78e37124632c38619b36873  

## Оставшиеся вопросы

1. Несмотря на поддержку кеширующих заголовков при запросе скрипта сервис-воркера, браузеры уменьшают время жизни кеша до 24 часов. http://stackoverflow.com/questions/38843970/service-worker-javascript-update-frequency-every-24-hours/38854905#38854905
2. Обновление срабатывает, только если сам скрипт сервис-воркера обновился, и определение этого происходит побайтово. Из этого следует, что обновление файлов, которые подключены через importScripts, не приведёт к обновлению самого сервис-воркера. http://blog.pushpad.xyz/2016/07/service-worker-importscripts-never-updates-scripts/

## Ответы на вопросы в файле service-worker.js

```javascript
// Вопрос №1: зачем нужен этот вызов?
.then(() => self.skipWaiting())
.then(() => console.log('[ServiceWorker] Installed!'));
```

Вызов метода skipWaiting изменяет поведение так,
что новый service worker начинает свою работу сразу после завершения установки
(пропуск стадии INSTALLED и перехода к стадии ACTIVATING).
При этом новый service worker будет контроллировать и старые страницы!
* https://bitsofco.de/the-service-worker-lifecycle/
* https://developers.google.com/web/fundamentals/instant-and-offline/service-worker/lifecycle


```javascript
// Вопрос №2: зачем нужен этот вызов?
self.clients.claim();
```

Даже после успешной регистрации service worker начинает контролировать страницы только после их перезагрузки. Вызов clients.claim() позволяет активировать service worker немедленно для всех страниц, которые находятся в его scope, без их перезагрузки.

```javascript
//Вопрос №3: для всех ли случаев подойдёт такое построение ключа?
const cacheKey = url.origin + url.pathname;
```

В текущей реализации - да. Однако, eсли у запросов будет параметр url.search, то он не будет учтен.

```javascript
// Вопрос №4: зачем нужна эта цепочка вызовов?
return Promise.all(
  names.filter(name => name !== CACHE_VERSION)
    .map(name => {
      console.log('[ServiceWorker] Deleting obsolete cache:', name);
      return caches.delete(name);
    })
 );
 ```
 
Данная цепочка вызовов необходима для удаления предыдущей версии cache:
<pre>
names.filter Получаем массив со всеми версиями cache, кроме текущей
map На каждый элемент массива получаем promise, возвращенный вызовом caches.delete
Promise.all Возвращает promise, который будет разрешен только когда разрешатся все promises из caches.delete
</pre>

```javascript
// Вопрос №5: для чего нужно клонирование?
cache.put(cacheKey, response.clone());
```

Потоки запроса (request) и ответа (response) могут быть прочитаны только один раз.
Поэтому в cache помещается клон ответа, а оригинальный ответ передаётся браузеру.
