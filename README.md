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

##Ответы на вопросы в файле service-worker.js

```javascript
// Вопрос №1: зачем нужен этот вызов?
.then(() => self.skipWaiting())
.then(() => console.log('[ServiceWorker] Installed!'));
```

Вызов метода skipWaiting изменяет поведение так,
что новый service worker начинает свою работу сразу после завершения установки
(пропуск стадии INSTALLED и перехода к стадии ACTIVATING).
При этом новый service worker будет контроллировать и старые страницы!
<pre>
https://bitsofco.de/the-service-worker-lifecycle/
https://developers.google.com/web/fundamentals/instant-and-offline/service-worker/lifecycle
</pre>

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
