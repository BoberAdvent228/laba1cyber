Отчёт по практической работе#1 Компьютерная безопасность и защита информации Выполнил студент группы 090301-ПОВа-о25 Мартынов Игорь Иванович Тема: Установка SSL-сертификата для nginx Цель работы: Получить практические навыки настройки защищённого HTTPS-соединения для веб-сервера nginx с использованием самоподписанного SSL-сертификата.
Выполнение работы: Запуск контейнера nginx Для запуска веб-сервера был использован Docker Compose. В терминале в папке с проектом выполнена команда: docker compose up -d
<img width="626" height="153" alt="image" src="https://github.com/user-attachments/assets/caf33d03-5175-41b0-a132-c5e546c76938" /> 

Проверка работы по HTTP: В браузере открыт адрес http://localhost. Веб-сервер корректно отобразил статическую страницу с текстом «Hello from Nginx in Docker!». Это подтверждает, что nginx работает и раздаёт контент.
<img width="1896" height="913" alt="image" src="https://github.com/user-attachments/assets/fda573ec-62f8-4ca6-8c36-73ac164e7ffd" />   local host
Генерация самоподписанного сертификата: 1.Подключение к контейнеру: docker exec -it nginx sh 2.Установка OpenSSL: apk update && apk add openssl 3.Создание директории и генерация ключа с сертификатом: mkdir -p /etc/nginx/ssl openssl req -x509 -nodes -days 365 -newkey rsa:2048
-keyout /etc/nginx/ssl/nginx.key
-out /etc/nginx/ssl/nginx.crt
-subj "/C=RU/ST=Moscow/L=Moscow/O=Lab/CN=localhost" В результате в директории /etc/nginx/ssl появились файлы nginx.crt (сертификат) и nginx.key (закрытый ключ)
<img width="1086" height="196" alt="image" src="https://github.com/user-attachments/assets/57f0ac50-97cc-4ac5-a6ac-33b2b9188b10" />
Настройка nginx для HTTPS: В файл конфигурации ./nginx/default.conf были внесены изменения. Настроен автоматический редирект с HTTP (порт 80) на HTTPS (порт 443), а также указаны пути к сгенерированным сертификатам.

Итоговый конфигурационный файл: server { listen 80; server_name localhost;

return 301 https://$host$request_uri;
}

server { listen 443 ssl; server_name localhost;

ssl_certificate /etc/nginx/ssl/nginx.crt;
ssl_certificate_key /etc/nginx/ssl/nginx.key;

location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
}
}

Проверка синтаксиса конфигурации прошла успешно: docker exec -it nginx nginx -t image
<img width="1004" height="129" alt="image" src="https://github.com/user-attachments/assets/be72605e-e09e-4076-b3d8-701c7e6eee91" />
Изменения применены командой: docker exec -it nginx nginx -s reload image
<img width="985" height="113" alt="image" src="https://github.com/user-attachments/assets/442f32a5-e9b9-4cdc-a22e-44f4203c8779" />
Проверка HTTPS: В браузере открыт адрес https://localhost. Так как сертификат является самоподписанным, браузер выдал предупреждение о недоверенном сертификате. После подтверждения перехода страница успешно загрузилась, что подтверждает корректную работу HTTPS.
<img width="1332" height="418" alt="image" src="https://github.com/user-attachments/assets/96fb51d4-a0b2-4bcd-a988-037f94f67fe2" />

Исследование трафика в Wireshark: С помощью Wireshark был захвачен трафик интерфейса Loopback. При обращении к https://localhost в захвате обнаружены пакеты TLS-соединения.
<img width="1823" height="407" alt="image" src="https://github.com/user-attachments/assets/855a6075-d854-4f5e-86cc-1be22b51da70" />
Client Hello: В данном пакете клиент (браузер) предлагает версию протокола TLS 1.3 (подтверждается расширением supported_versions).
<img width="535" height="137" alt="image" src="https://github.com/user-attachments/assets/3d72167f-f6c6-4210-bdd8-e81a65fe8bab" />
Server Hello: Сервер выбрал версию TLS 1.3 и конкретный шифр (Cipher Suite) для защищённого соединения.
<img width="471" height="27" alt="image" src="https://github.com/user-attachments/assets/89b062b5-8591-473a-ac4f-79edfb07a639" />
<img width="998" height="375" alt="image" src="https://github.com/user-attachments/assets/a28651d1-bd70-4d8f-aa10-90e8598af61f" />
Сертификат сервера: В браузере при просмотре сертификата видно, что он выдан на имя localhost (CN=localhost), издатель — сам сертификат (Self-signed), срок действия — 365 дней.
<img width="606" height="417" alt="image" src="https://github.com/user-attachments/assets/ee422b48-6d87-4877-a011-90dcac9e498b" />

Application Data: После установки защищённого соединения все последующие данные передаются в зашифрованном виде. В пакетах типа Application Data содержимое HTTP-запроса и ответа не отображается в открытом виде.
<img width="838" height="202" alt="image" src="https://github.com/user-attachments/assets/579a1995-a674-4a34-8ff6-c6fab1eeedf9" />
Сравнение HTTP и HTTPS Для сравнения был выполнен захват трафика при обращении к http://localhost. В Wireshark обнаружен пакет, содержащий HTTP-запрос в открытом виде.
<img width="1513" height="406" alt="image" src="https://github.com/user-attachments/assets/405f7f58-b587-4ff6-a65e-5b5b5d25a165" />
Здесь отлично видно, что при использовании HTTP весь запрос передаётся в открытом виде: GET / HTTP/1.1 Host: localhost User-Agent: Mozilla/5.0... (мой браузер Opera GX) Accept-Language: ru-RU... Любой, кто перехватит этот трафик, сможет прочитать всё это без проблем. Вывод: При передаче данных по протоколу HTTP содержимое запроса (GET / HTTP/1.1, Host: localhost, User-Agent) отображается в сетевом трафике в открытом виде, что позволяет перехватить и прочитать его. При использовании HTTPS эти же данные находятся внутри зашифрованного пакета Application Data и не читаются. Таким образом, HTTPS обеспечивает конфиденциальность передаваемой информации за счёт шифрования TLS. Общий вывод по работе: В ходе практической работы были получены навыки настройки защищённого HTTPS-соединения для веб-сервера nginx. Сгенерирован самоподписанный SSL-сертификат, настроен автоматический редирект с HTTP на HTTPS, а также проведён анализ трафика в Wireshark, подтвердивший эффективность шифрования данных.





