# Cyrus 0.5.0-dev — buildfix1

Дата: 2026-09-18. Статус: исправленные исходники для повторной проверки CI, НЕ подтверждённая сборка APK.

## Основа и точное совпадение с падением

Получен публичный snapshot Eren2932/Cyrus, ветка main. Git archive comment: `36ff74519fdde11c5a55a6829c93dc12df509d01` — соответствует сокращённому commit на скриншоте.

Исходный ZIP: `Cyrus-0.5.0-dev-source.zip`.
SHA-256: `182cb5b6eaa7534b5a8715d8b179d0b8c75ac0e8fe97f363c05006a96b32c24b`.
Хеш совпадает с Selected source в присланном запуске Actions. Исходный FILES.sha256 проверен: 175 записей совпали.

Имя исправленного архива: `Cyrus-0.5.0-dev-buildfix1-source.zip`. Версия Android намеренно оставлена `0.5.0-dev`, versionCode 7: это узкое исправление той же dev-версии, не новая функциональная версия. applicationId, БД, ключ Keystore, ресурсы и схемы хранения не менялись.

## Причина показанного падения

В присланном логе задача `:vpn:service:compileDebugKotlin` завершилась ошибкой в `LibboxVpnEngine.kt:285`: `Unresolved reference 'protect'` и два сообщения о невозможности вывести тип. Это ошибка компиляции Kotlin, не исчерпание памяти и не сбой Gradle daemon. Сообщение о single-use daemon в начале лога штатное.

После добавления proxy mode поле `vpn` имеет общий тип `android.app.Service`. API `protect(Int)` принадлежит `android.net.VpnService`. Исходник пытался использовать сужение типа свойства через `if (vpn !is VpnService) return`, а затем обращаться к нему внутри `runCatching`. В данном запуске компилятор не разрешил вызов. Внутреннюю причину поведения компилятора отдельно не воспроизводили; утверждать, что любой такой smart cast в Kotlin не работает, нельзя.

В исправлении используется явно типизированная локальная ссылка:

````kotlin
val vpnService: VpnService = vpn as? VpnService ?: return
val protected: Boolean = runCatching { vpnService.protect(fd) }.getOrDefault(false)
if (!protected) {
    error("Не удалось защитить сокет VPN от петли маршрутизации")
}
````

Обычная proxy Service по-прежнему выходит из callback без VPN-операции. VPN вызывает protect с исходным fd; false или исключение означают ошибку. Защита не отключалась, результат не подменялся true. Запрет TUN в proxy mode сохранён. Lifecycle, TUN ownership и NonCancellable stop не переписывались.

## Дополнительные исправления CI

**Файлы для native validation и Gradle cache.** `UpgradeConfigTest` создаёт `build/native-fixtures`, `ServerSetupTest` — `build/server-script-fixtures`. Следующий shell-шаг читает эти файлы, но раньше они не были объявлены выходами test task. На чистом runner с восстановленным результатом теста из кэша файлы могли отсутствовать. Оба каталога добавлены через `outputs.dir` к `:vpn:config:test`. Эффект восстановления из реального Gradle cache ещё требует проверки CI.

**Go при тёплом AAR-кэше.** Внешний workflow устанавливал Go 1.23.6 только при необходимости собрать AAR. Но `check_generated_configs.sh` собирает проверяющий Go-бинарник и при cache hit. Теперь setup-go безусловный, включая ZIP с готовым AAR. Отключён cache setup-go, зависящий от `.ci-native/go.sum`: checkout этого каталога остаётся условным. Сам AAR-кэш сохранён.

**Явная компиляция до тестов.** Во внешнем и внутреннем CI добавлены `:app:compileDebugKotlin :app:compileReleaseKotlin`. Это раньше выявляет ошибки обеих компиляционных веток. Unit tests, Android lint, native validation и сборка APK не отключались, `continue-on-error` не добавлялся.

Обновлён существующий source guard для нового имени receiver. Добавлено 11 узких source checks и 11 mutation cases; все подключены к `tools/check_source.py`. Мутации проверяют, что guard обнаруживает возврат старого кода, отключение protect, игнорирование false/exception и регрессии CI. Это тесты проверяющих скриптов, НЕ выполнение VPN callback.

## Общее устройство клиента по коду

В проекте 16 Gradle-модулей и 75 Kotlin-файлов. Android UI построен на Compose/MVI, внедрение зависимостей — Koin. Domain отделён от Room, DataStore и нативного адаптера. Ключи/подписки хранятся через AES-GCM и Android Keystore; сетевые настройки — в DataStore.

Цепочка подключения: UI → TunnelControllerImpl → отдельная VPN либо proxy Service → TunnelServiceRunner → LibboxVpnEngine → libbox/sing-box. ConfigFactory формирует JSON, а vpn:link импортирует ссылки и поддерживаемые Xray JSON-узлы. Ядро закреплено на sing-box v1.11.1, commit `92d245ad040cbda2f84b21c2a847a470e532c179`; в этом исправлении его версия и ABI не менялись.

Существенные существующие ограничения: XHTTP, gRPC multiMode и HY2 pinSHA256 не поддерживаются этим адаптером; рабочая native-статистика через CommandServer не завершена; kill switch и end-to-end health check не реализованы; proxy mode не защищает всё устройство. Магазин — локальный каталог с отключёнными продажами, а не готовый backend. Старая ARCHITECTURE.md содержит планы: их нельзя считать доказательством уже работающей функции или актуальной безопасности старого ядра.

Эта работа — обзор структуры и узкое исправление сборки, не полный аудит всех протоколов, гонок и безопасности клиента.

## Реально выполненные проверки

`python3 tools/check_source.py` завершился успешно. В том числе: 48 основных source checks, 16 merge guards, 89 upgrade checks, 11 новых buildfix checks; дизайн — 10 160 вычислительных assertions / 71 цветовая пара; mutation suites — 12 design, 10 merge, 10 upgrade и 11 buildfix cases; 6 синтетических проверок AAR-validator. Синтетический AAR здесь не является настоящей нативной библиотекой.

Проверены XML, TOML, баланс Kotlin-разделителей, Python-синтаксис, `bash -n` трёх shell-файлов и 15 многострочных shell-блоков workflow, синтаксис встроенного CI Python. Это не полноценная проверка YAML/GitHub Actions. Новый guard отвергает исходное дерево с проблемным callback. Выполнены упаковочная проверка ZIP и проверка FILES.sha256.

Журналы: `validation/source-checks.txt`, `validation/buildfix-syntax-checks.txt`. Изменения: `validation/buildfix-change-map.json` и `validation/buildfix-code.patch`. Остальные ранее существовавшие файлы validation относятся к исходной 0.5.0-dev, если явно не помечены buildfix.

**Не выполнялись:** Kotlin/Gradle compilation, JUnit, Android lint, реальная native schema validation, создание/установка APK и испытания VPN/proxy на телефоне. В the Computer есть Java, но нет Gradle, kotlinc, Android SDK и настоящего libbox.aar; установка пакетов недоступна. Правки не отправлялись в GitHub, Actions от имени владельца не запускался. Соответственно, снят выявленный дефект исходника, но успешность всего pipeline ещё не подтверждена.

## Применение в архивном репозитории

1. Добавьте `Cyrus-0.5.0-dev-buildfix1-source.zip` в корень Eren2932/Cyrus, не распаковывая его туда.
2. Замените `.github/workflows/build.yml` приложенным `Cyrus-buildfix1-build.yml`. Та же версия лежит в `Cyrus/docs/workflows/build-zip.yml` внутри ZIP. Желательно загрузить оба изменения одним коммитом.
3. Actions → Cyrus APK from ZIP → Run workflow: поле `archive` = `Cyrus-0.5.0-dev-buildfix1-source.zip`.
4. Убедитесь, что Selected source показывает именно это имя и хеш из приложенного SHA256SUMS.txt. Простое Re-run старого запуска продолжит работать со старым commit и не подхватит новый ZIP.
5. Дождитесь компиляции, tests/lint, native validation и assembleDebug. Артефакт `Cyrus-libbox-*` — это только библиотека, не готовый APK. Устанавливаемый результат появится как `Cyrus-debug-APK-*`; для большинства современных телефонов нужен arm64-v8a.

Не удаляйте приложение для решения проблемы подписи без безопасного экспорта ключей. Установка поверх существующей версии требует совместимой подписи. После зелёного CI отдельно нужны испытания VPN/proxy, внешнего IP/DNS, переключения профилей и сети, остановки из уведомления и ошибочных подключений. buildfix1 остаётся dev-исходником до этих проверок.
