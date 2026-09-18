# Cyrus 0.5.0-dev — buildfix2

## Основа и подтверждённый результат предыдущего запуска

Этот архив подготовлен поверх `Cyrus-0.5.0-dev-buildfix1-source.zip`, SHA-256 `95d683bfd73a9db30b7538b466d668385bd007ff82e310944d0b9b0536da4cd1`. Это тот же хеш, который виден в присланном Selected source.

На скриншоте запуска GitHub Actions `35346608067` компиляция `:app:compileDebugKotlin :app:compileReleaseKotlin` завершилась успешно. Новый лог показывает другую ошибку: `:vpn:link:test` → `DefaultLinkParserTest > rejects garbage` → строка 78. Из 51 теста этого модуля упал один. Эти результаты относятся к buildfix1, не к новому архиву и не ко всем этапам pipeline.

## Исправление тестового контракта

В 0.5.0 добавлена поддержка HTTP/HTTPS-прокси, включая анонимный HTTPS endpoint с портом 443 по умолчанию. Старый тест по-прежнему ожидал `null` от `parser.parse("https://example.com")`. Новому контракту это противоречит: в режиме ключей это допустимый адрес прокси.

Тест `rejects garbage` сохранён. Теперь он проверяет неизвестную схему, произвольный текст, HTTPS без адреса и пустой результат разбора мусора. Добавлены шесть JUnit-сценариев: анонимный HTTPS, анонимный HTTP, явный порт/root path, отделение URL подписки от прокси, некорректные адреса/порты/учётные данные и round-trip экспорта HTTPS без авторизации. Всего в исходниках vpn:link теперь 57 тестов вместо 51.

Production-парсер и ProfileLinkWriter не изменены: поддержка HTTPS/VLESS и остальных форматов не урезалась. В частности, HTTPS-ссылка подписки с `/sub/...` по-прежнему должна вводиться в явном режиме подписки, а её VLESS WS/TLS содержимое импортируется как ключ.

## Дополнительная проверка Android API

При просмотре ещё не завершённого lint-этапа выявлены отдельные проблемы исходников. Они НЕ указаны причиной текущего падения в логе и не выдаются за результаты запущенного lint.

1. `ConnectivityManager.requestNetwork(request, callback, handler)` — перегрузка API 26 при minSdk 24. Вызов теперь находится за проверкой SDK >= O. На API 24/25 используется двухаргументный вариант. Новая HandlerThread создаётся только на API 26+.
2. Для активного `requestNetwork` требуется обычное разрешение `CHANGE_NETWORK_STATE` (или системное право менять настройки). В manifest добавлено именно это normal permission: пользовательского диалога для него не нужно. В исходном manifest был только `ACCESS_NETWORK_STATE`, достаточный для чтения состояния, но не для такого активного запроса.
3. При переходе в Connected обычный `NotificationManager.notify()` заменён на обновление существующей foreground-службы через `service.startForeground(notificationId, notification(true))`. Первоначальный startForeground до начала подключения сохранён. Таким образом, обновление FGS не требует разрешения на обычные уведомления POST_NOTIFICATIONS. Новая служба этим кодом не создаётся. Если пользователь запретил уведомления, Android может не показывать FGS в обычной шторке — это системное поведение, не обход разрешения.

Сверка API 24 выполнена по AOSP `android-7.0.0_r1/core/java/android/net/ConnectivityManager.java`: у API 24 имеется двухаргументный requestNetwork, отсутствует публичная перегрузка с Handler, и документация указывает требуемое разрешение.

Источник: https://raw.githubusercontent.com/aosp-mirror/platform_frameworks_base/android-7.0.0_r1/core/java/android/net/ConnectivityManager.java

Защита VPN-сокета из buildfix1, TUN ownership, NonCancellable stop и все прежние проверки сохранены. Версия остаётся 0.5.0-dev / code 7. Application ID, БД, шифрование, UI и протокольные настройки не менялись.

## Что проверено здесь

`python3 tools/check_source.py` завершился успешно: существующие source/design/merge/upgrade suites, 17 buildfix source checks и 16 мутационных проверок этих guards. Проверены Python/XML/TOML, баланс Kotlin-разделителей, bash-синтаксис. Проверены patch против buildfix1, ZIP CRC, состав и FILES.sha256. Удаления тестов, @Ignore, ignoreFailures, отключения lint и continue-on-error не добавлялись.

Доступная локальная среда по-прежнему не позволяет выполнить полноценный Android build. Попытка получить официальный переносимый Kotlin compiler 2.0.21 через URL-загрузчик упёрлась в лимит файла 50 МБ. Компилятор не установлен, JUnit нового архива здесь НЕ выполнялся. Gradle, Android SDK, native AAR и запуск GitHub Actions здесь также недоступны. Проверки исходников не заменяют компиляцию.

Нельзя честно назвать новый ZIP подтверждённо рабочим APK до успешного CI. В нём исправлено конкретное падение из лога и найденные проблемы API; весь pipeline ещё должен завершиться. Предыдущий успешный этап компиляции относится к buildfix1, а изменённый Android-код buildfix2 должен скомпилироваться заново.

Текущий журнал: `validation/source-checks.txt`; patch: `validation/buildfix2-code.patch`; карта изменений: `validation/buildfix2-change-map.json`. Файлы с buildfix1 и старые отчёты оставлены как история.

## Как запустить — workflow уже подходит

Добавьте `Cyrus-0.5.0-dev-buildfix2-source.zip` в корень Eren2932/Cyrus. Не распаковывайте его в архивный репозиторий. Уже установленный `.github/workflows/build.yml` менять НЕ нужно. Он подхватывает изменившийся ZIP при push.

Для ручного запуска Actions → Cyrus APK from ZIP → Run workflow укажите точное имя `Cyrus-0.5.0-dev-buildfix2-source.zip` в поле archive. Старый запуск не перезапускайте через Re-run: он относится к старому commit. В Selected source должно быть новое имя и SHA-256 из приложенного файла.

APK выдаётся только после успешных compilation, tests/lint, native validation и assembleDebug. Артефакт Cyrus-libbox — не APK. После получения APK отдельно проверьте VPN/proxy, внешний IP/DNS, смену Wi-Fi/мобильной сети и остановку соединения. Изменение callback для API 24/25 и запрет уведомлений на Android 13+ требуют проверки на соответствующих устройствах.
