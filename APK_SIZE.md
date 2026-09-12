# Вес APK: бюджет и приёмы

## Бюджет

| Что | arm64 release, R8 + shrinkResources |
|---|---|
| Compose + Material3 + Navigation | ~1.8–2.5 МБ |
| Room + DataStore | ~0.3 МБ |
| Ktor 3 + OkHttp | ~0.6–0.9 МБ |
| Koin | ~0.15 МБ |
| Наш код и ресурсы | ~0.3 МБ |
| **Итерация 1 (движок-симулятор)** | **2.5–4 МБ** |
| libbox (sing-box, gomobile, один ABI) | **+10–16 МБ** |
| **Итерация 2, per-ABI** | **13–19 МБ** |

Универсальный APK на 3 ABI — 35–50 МБ. Именно поэтому в `app/build.gradle.kts` включены
ABI-сплиты, а для Play готовится App Bundle.

## Почему ниже 10 МБ с ядром не получится

VLESS/Reality/XHTTP реализованы только в Go-ядрах (sing-box, xray-core). Go-бинарь тянет
рантайм, GC и планировщик. Это физический пол для любого современного VPN-клиента:
v2rayNG, Hiddify, NekoBox — все в диапазоне 30–70 МБ универсальным APK.

## Чеклист сжатия

### Уже включено
- [x] `isMinifyEnabled = true`, `isShrinkResources = true`, R8 full mode
- [x] ABI-сплиты + universal APK отдельно
- [x] `resourceConfigurations += setOf("ru", "en")`
- [x] `ui-tooling` только в debug
- [x] нет `material-icons-extended`, нет Firebase, нет аналитик-SDK
- [x] `packaging.resources.excludes` для META-INF мусора

### При интеграции ядра
- [ ] `gomobile bind -trimpath -ldflags "-s -w -buildid="` — минус ~25%
- [ ] отказ от `with_gvisor`, `stack=system` в TUN — минус ~4 МБ
- [ ] только продаваемые протоколы в build tags
- [ ] `android:extractNativeLibs="false"` (по умолчанию в AGP 8) — меньше объём установки
- [ ] AAB для Play: пользователь качает свою архитектуру и свою плотность

### Измерение
```bash
gradle :app:bundleRelease
# отчёт по вкладу каждой библиотеки:
gradle :app:assembleRelease --scan
unzip -l app/build/outputs/apk/release/*.apk | sort -k1 -n | tail -30
```
