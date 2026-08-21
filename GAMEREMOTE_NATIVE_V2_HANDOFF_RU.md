# GameRemote NativeV2: handoff по пути 2

Дата последней актуализации: 21 августа 2026 года.

Этот документ предназначен для продолжения работы другим разработчиком или
агентом, в том числе Claude. Он описывает выбранную архитектуру, состояние
кода, уже внесённые изменения, найденные причины нестабильности, ограничения,
проверки и последовательность дальнейшей реализации.

## 0. Авторитетный срез после завершения Path 2

Этот раздел добавлен после реализации основных пунктов Path 2 и имеет
приоритет над историческими формулировками ниже. Если ниже сказано, что
multi-viewer, нативный capture/codec или файловый канал ещё не реализованы,
это описание промежуточного этапа.

### 0.1 Что сделал Codex к 21 августа 2026 года

Кодовая часть Path 2 завершена по следующим границам:

- ABI `2.4` (`0x00020004`) в `GameRemote.NativeV2.dll`;
- WebRTC PeerConnection, trickle ICE, ICE restart, RTCStats, Opus/NetEq;
- три data channel: надёжный `control`, latest-only `input-fast` и
  низкоприоритетный надёжный `files`;
- DXGI Desktop Duplication внутри нативной DLL;
- BGRA texture -> D3D11 Video Processor -> NV12 texture без копирования
  полного кадра в managed RAM;
- аппаратный H.264 Media Foundation encoder с выбором адаптера
  NVENC/QSV/AMF и диагностируемым fallback;
- аппаратный H.264 Media Foundation/DXVA decoder внутри нативной DLL;
- передача декодированной NV12 D3D11 texture в собственный Viewer presenter;
- decoder recovery: очередь до трёх access units, сброс зависимой P-frame
  цепочки при переполнении, ожидание IDR и ограниченный запрос key frame;
- один DXGI/capture/encoder на ПК и fan-out готового H.264 для максимум трёх
  Host-сеансов;
- отдельная bounded очередь каждого подписчика, поэтому медленный Viewer не
  создаёт backpressure владельцу или другим администраторам;
- файловый канал с SHA-256, `.part`, просмотром дисков/каталогов,
  upload/download/delete/create-directory и ограничением buffered amount;
- выбор и сохранение `winsta0\default` / `winsta0\winlogon` службой;
- остановка только Host-сеансов затронутой Windows session при реальной смене
  desktop; Viewer затем использует существующий автоматический reconnect;
- защита от старого async loop: завершившийся Host прошлого поколения не
  может остановить новый Host с тем же SessionId.

Код собран и проверен native loopback, полным managed ABI 2.4 loopback,
живым DXGI/H.264/аудиопроходом и probe конечных пакетов. Полный ABI 2.4
artifact теперь выбирается как основной product route по умолчанию.
`legacy-sipsorcery-v1` сохранён только как аварийный fallback, если NativeV2
не прошёл строгий probe, не установил соединение или не восстановил живой
медиатракт двумя новыми ICE/Host-сеансами.

### 0.2 Как связаны проекты при сборке

```text
AdminClient.exe
  -> CommonModels
  -> GameRemote.Viewer
       -> GameRemote.NativeV2.Managed
            -> NativeV2/GameRemote.NativeV2.dll

BotStatsMTS.exe
  -> CommonModels
  -> GameRemote.Viewer
       -> GameRemote.NativeV2.Managed
  -> явно импортирует GameRemote.NativeV2.Artifact.targets

6D6X6.exe
  -> 6D6X6.Service
       -> публикует GameRemote.Host как self-contained win-x64
            -> GameRemote.NativeV2.Managed
                 -> NativeV2/GameRemote.NativeV2.dll
```

`AdminClient`, `BotStatsMTS`, `GameRemote.Viewer` и `GameRemote.Host`
импортируют artifact target напрямую, потому что content-файлы нативной DLL
не всегда транзитивно доходят через `ProjectReference`.
`6D6X6.Service.csproj` сам публикует полную папку `GameRemote.Host`, а
`6D6X6.csproj` копирует её в пакет клиента и проверяет наличие runtime-файлов
и актуальность `GameRemote.Host.dll`.

#### 0.2.1 Корни репозиториев и источники истины

Работа разделена между двумя рабочими деревьями и отдельным checkout
официального `libwebrtc`:

| Корень | За что отвечает | Что нельзя делать |
|---|---|---|
| `Y:\BotStatsMTS` | BotStats Server, AdminClient, общий Viewer, Host, managed ABI, исходники C++ NativeV2, smoke и handoff | Не чинить сгенерированные копии под `bin`, `obj` или `.release` |
| `Y:\6D6X6` | клиент ПК, Windows-служба, user-agent IPC, запуск Host в нужной session/desktop и упаковка Host | Не создавать вторую копию медиатракта в службе или `6D6X6.CaptureWorker` |
| `D:\GameRemoteNativeV2\webrtc\src` | закреплённый официальный checkout `libwebrtc` и GN/Ninja build tree | Не менять ревизию без обновления artifact manifest, smoke и handoff |
| `D:\GameRemoteNativeV2\staged\GameRemote.NativeV2` | проверенный staging artifact, который подхватывает MSBuild target | Не редактировать DLL вручную и не подменять один файл без manifest/notices |

Направление зависимости одностороннее: `6D6X6.Service.csproj` вызывает publish
проекта `Y:\BotStatsMTS\GameRemote.Host\GameRemote.Host.csproj`; сам Host
ссылается на `GameRemote.NativeV2.Managed`, а тот импортирует единый
`GameRemote.NativeV2.Artifact.targets`. Поэтому исправление Host/media всегда
начинается в `Y:\BotStatsMTS`, после чего пересобирается `Y:\6D6X6`.

`AdminClient` и `BotStatsMTS` оба используют один проект
`GameRemote.Viewer`. Если дефект одинаков в обоих приложениях, первым делом
нужно смотреть Viewer/managed/native слой, а не дублировать исправление в двух
UI. Если дефект есть только в одном приложении до появления окна Viewer,
нужно смотреть соответствующий signaling adapter и вызов в `MainWindow`.

`6D6X6.CaptureWorker` не является рабочим NativeV2 capture-процессом. В
Path 2 DXGI, GPU-конвертация и codec находятся в
`GameRemote.NativeV2.dll`, загруженной процессом `GameRemote.Host`. Старый
worker нельзя возвращать в product route как обходное решение.

### 0.3 Как связаны проекты во время работы

```text
AdminClient / BotStats UI
  -> GameRemote.Viewer.RemoteDesktopWindow
  -> IRemoteDesktopSignaling
  -> BotStats Server + RemoteDesktopSessionBroker
  -> постоянный server channel клиента 6D6X
  -> 6D6X6.Service.RemoteDesktopHostManager
  -> GameRemote.Host в нужных Windows session/desktop
  -> GameRemote.NativeV2.dll

После SDP/ICE и одноразовой авторизации:

GameRemote.Viewer <===== прямой WebRTC =====> GameRemote.Host
      video / audio / control / input-fast / files
```

BotStats переносит только команды, SDP/ICE и lease. Видео, звук, ввод и файлы
через BotStats не проходят.

### 0.4 Точная карта файлов: куда заходить с конкретной проблемой

| Зона | Первый файл | Что в нём искать |
|---|---|---|
| Кнопка подключения AdminClient | `AdminClient/MainWindow.xaml.cs` | создание `RemoteDesktopWindow` и `AdminRemoteDesktopSignaling` |
| Кнопка подключения BotStats | `BotStatsMTS/MainWindow.xaml.cs` | создание `RemoteDesktopWindow` и `BotStatsRemoteDesktopSignaling` |
| Общий UI Viewer, reconnect, fallback, файлы | `GameRemote.Viewer/RemoteDesktopWindow.cs` | `StartNativeV2Async`, `CompleteNativeV2SignalingAsync`, `Reconnect`, `SendFilesCoreAsync` |
| Контракты request/answer/signal/stop | `CommonModels/RemoteDesktopContracts.cs` | `RemoteDesktopSessionRequest`, `RemoteDesktopSessionAnswer`, signal DTO |
| Сигналинг AdminClient | `AdminClient/AdminRemoteDesktopSignaling.cs` | `StartAsync`, `ExchangeSignalAsync`, `StopAsync` |
| Сигналинг встроенного Viewer BotStats | `BotStatsMTS/BotStatsRemoteDesktopSignaling.cs` | прямые вызовы методов `Server` |
| Lease, до трёх Viewer, token, replay/cursor | `BotStatsMTS/RemoteDesktopSessionBroker.cs` | reserve/commit/signal/release и validation envelope |
| Пересылка запросов к 6D6X | `BotStatsMTS/Server.cs` | `BeginRemoteDesktopSessionAsync`, `ExchangeRemoteDesktopSessionSignalAsync`, `StopRemoteDesktopSessionAsync` |
| Приём команды на службе клиента | `Y:\6D6X6\6D6X6.Service\ServerChannel.cs` | ветки `RemoteDesktopSessionRequest/Signal/Stop` |
| Владение Host-процессами | `Y:\6D6X6\6D6X6.Service\RemoteDesktopHostManager.cs` | `Start`, `ExchangeSignal`, `NotifyDesktopChanged`, `HostRuntime.Generation` |
| Запуск процесса в Windows session | `Y:\6D6X6\6D6X6.Service\SessionLauncher.cs` | user token, SYSTEM fallback, `winsta0\default`, `winsta0\winlogon` |
| Связь службы с user-agent | `Y:\6D6X6\6D6X6.Service\IpcServer.cs` и `Y:\6D6X6\6D6X6\ServiceIpcClient.cs` | `HELLO`, preferred session, `HOST_START` |
| Точка входа Host | `GameRemote.Host/Program.cs` | разбор offer, выбор `NativeV2HostSessionController` |
| NativeV2 Host session | `GameRemote.Host/Session/NativeV2HostSessionController.cs` | answer, auth, input, audio, files, trickle/restart |
| Native Host media adapter | `GameRemote.Host/Native/NativeV2HostVideoPipeline.cs` | `StartHostMedia`, native events, quality, IDR |
| Один encoder на 2-3 Viewer | `GameRemote.Host/Session/SharedEncodedVideoRelay.cs` | owner election, named pipe, subscriber queue, GOP recovery |
| Операции удалённой ФС | `GameRemote.Host/Transfer/FileTransferReceiver.cs` | bounded upload/download, SHA-256, fs-list/delete/create |
| NativeV2 Viewer session | `GameRemote.Viewer/Native/NativeV2ProductViewerSession.cs` | offer, auth, input, files, ICE restart, watchdog |
| Декодирование и показ texture | `GameRemote.Viewer/Native/NativeV2ViewerVideoPipeline.cs` | `VideoTexture`, latest-frame mailbox, key-frame recovery |
| D3D11 поверхность Viewer | `GameRemote.Viewer/Native/NativeVideoSurface.cs` | aspect ratio, present, resize/crop |
| Managed ABI | `GameRemote.NativeV2.Managed/NativeV2Runtime.cs` и `NativeV2Engine.cs` | загрузка DLL, exports, callback lifetime, ABI validation |
| Публичный C ABI | `GameRemote.NativeV2.Managed/native/include/gameremote_native_v2.h` | ABI 2.4, structs, events, exports |
| libwebrtc engine | `GameRemote.NativeV2.Managed/native/src/gameremote_native_v2_webrtc.cpp` | PeerConnection, channels, RTP, stats, native media bridge |
| DXGI/GPU/codec | `GameRemote.NativeV2.Managed/native/src/gameremote_native_v2_windows_media.cpp` | capture, Video Processor, H.264 MFT encode/decode |
| Native GN build | `GameRemote.NativeV2.Managed/native/BUILD.gn` | Windows media sources and system libraries |
| Artifact contract | `GameRemote.NativeV2.Managed/build/GameRemote.NativeV2.Artifact.targets` | staging, manifest, strict output validation |
| Smoke | `GameRemote.NativeV2.Managed.Smoke/Program.cs` | native/managed loopback, channels, H.264, PLI, ICE restart |

### 0.5 Последовательность одного подключения

1. `RemoteDesktopWindow` создаёт `NativeV2Engine` в роли Viewer и формирует
   offer/envelope с fingerprint DLL.
2. `AdminRemoteDesktopSignaling` или `BotStatsRemoteDesktopSignaling` передаёт
   request в `BotStatsMTS.Server`.
3. `RemoteDesktopSessionBroker` выдаёт lease и одноразовый токен, проверяет
   лимит администраторов и поколение сигналинга.
4. `Server` пересылает request по уже открытому каналу к службе 6D6X.
5. `RemoteDesktopHostManager` выбирает desktop, UDP-порт и запускает
   `GameRemote.Host` через подключённый user-agent либо `SessionLauncher`.
6. `GameRemote.Host/Program.cs` выбирает `NativeV2HostSessionController`,
   создаёт answer и возвращает его по отдельному named pipe службе.
7. Answer идёт обратно `Service -> BotStats -> Viewer`.
8. Viewer подтверждает одноразовый токен по `control` data channel.
9. После подтверждения video/audio/input/files идут напрямую через WebRTC.
10. Поздние ICE candidates и ICE restart продолжают проходить через broker,
    но медиапакеты через сервер не идут.

### 0.6 Один capture/encoder и несколько администраторов

Служба оставляет отдельный Host/PeerConnection для каждого администратора.
Это нужно для отдельного DTLS/SRTP, token, input и file channel. Внутри
`NativeV2HostVideoPipeline.Start` каждый Host подключается к
`SharedEncodedVideoRelay` с ключом ПК/desktop/monitor:

- первый процесс получает owner lock, запускает `StartHostMedia` в DLL и
  публикует готовые Annex-B access units;
- второй и третий процессы становятся subscriber и не запускают DXGI/MFT;
- каждому subscriber выделена bounded очередь размером 2;
- при отставании удаляется зависимая GOP-цепочка, subscriber ждёт следующий
  IDR, а не задерживает владельца;
- PLI любого Viewer передаётся владельцу как запрос локального key frame;
- при завершении владельца подписчики завершают текущую media session, Viewer
  автоматически переподключается, после чего новый Host выигрывает owner lock.

Последний пункт требует полевого soak после убийства owner Host. Это уже
контролируемое восстановление, но не бесшовная миграция PeerConnection.

### 0.7 Файловый канал

`files` создаётся ordered/reliable с `webrtc::Priority::kLow`. Передача не
может бесконечно заполнять SCTP:

- managed Viewer ждёт, пока buffered amount опустится ниже порога;
- Host ограничивает исходящую очередь 512 КБ;
- файл сначала пишется в `.part`, затем проверяется размер и SHA-256;
- rename в итоговый путь выполняется только после успешной проверки;
- отмена/обрыв удаляет незавершённый `.part`;
- файловая нагрузка не использует очередь video/audio/input.

### 0.8 Desktop, перезагрузка и восстановление

Служба хранит последний авторитетный desktop в
`%ProgramData%\6D6X6\State\GameRemote.desktop`.

- без интерактивного пользователя новый Host всегда стартует на
  `winsta0\winlogon`, даже если с прошлого запуска остался `default`;
- `SessionLock/SessionLogoff` сохраняют `winlogon`;
- `SessionUnlock/SessionLogon` сохраняют `default`;
- события другой Windows session не разрывают здоровые Host;
- при реальной смене desktop останавливаются только затронутые Host;
- окно Viewer остаётся открытым и выполняет штатное повторное согласование;
- после перезагрузки user-agent снова сообщает session через `HELLO`, служба
  выбирает правильный desktop, а Viewer повторяет подключение.

### 0.9 Artifact, который прошёл текущие тесты

```text
Staging: D:\GameRemoteNativeV2\staged\GameRemote.NativeV2
ABI: 2.4
WebRTC revision: e12c39e03c3dcab594f73a1e524b1f2c17dfdcb8
GameRemote.NativeV2.dll size: 8713216
SHA-256: 05913a99ebbdff449be95aa0e7165978f9a9eb09bcf912013814e9a3e7970207
```

Файлы artifact обязательны все вместе:

- `GameRemote.NativeV2.dll`;
- `GameRemote.NativeV2.manifest.json`;
- `WEBRTC_REVISION.txt`;
- `LICENSE.libwebrtc.txt`;
- `THIRD_PARTY_NOTICES.libwebrtc.md`.

Нельзя подменять только DLL: fingerprint и strict packaging намеренно
останавливают смешивание разных ABI/revision.

`GameRemote.NativeV2.Artifact.targets` копирует весь набор с режимом `Always`
и после сборки сравнивает SHA-256 DLL в `TargetDir` с выбранным staging
artifact. Не возвращать `PreserveNewest`: время файла не доказывает его ABI и
ранее из-за этого старый ABI 2.2 оставался в части `bin\Release` каталогов.

### 0.10 Что ещё не считается закрытым

Основной код и production route реализованы. Осталась эксплуатационная
проверка на реальных игровых ПК:

1. Windows 10 и Windows 11 на NVENC, Intel QSV и AMD AMF.
2. LAN, P1-P2, P1-P3 и P2-P3 с сохранением selected ICE pair/RTT/loss.
3. Два и три Viewer, включая медленного участника и закрытие owner Host.
4. Lock/unlock, logoff/logon, winlogon, reboot и автоматический reconnect.
5. Одновременные игра + звук + ввод + фоновая передача большого файла.
6. Длительный soak без роста RAM/handles/queued bytes.

Полный ABI 2.4 включается по умолчанию. Операционные предохранители не требуют
замены пакета:

```text
GAMEREMOTE_NATIVE_V2_DISABLE=1       # немедленно выключить NativeV2
GAMEREMOTE_NATIVE_V2_PCS=X5;X16     # необязательно ограничить список ПК
```

Старые `GAMEREMOTE_NATIVE_V2_PRODUCT_TEST` и
`GAMEREMOTE_NATIVE_V2_TEST_PCS` оставлены только как известные имена для
диагностики старых установок и больше не управляют маршрутом.

При проблеме NativeV2 сначала смотреть файлы из таблицы 0.4 и логи Host,
Viewer, selected ICE pair и backend encoder/decoder. Не исправлять NativeV2
правкой `WebRtcHostSession`, `NativeWebRtcSession` или WebView2: это legacy
route и другое поколение медиатракта.

### 0.11 Последняя воспроизводимая проверка

На 21 августа 2026 года после всех правок выполнено:

- native GN/Ninja build и нативный loopback: успешно;
- полный managed ABI 2.4 loopback: успешно, включая signaling, роли,
  `control`/`input-fast`/`files`, PCM/Opus/NetEq, ordered H.264, PLI/IDR,
  RTCStats, ICE restart и штатное завершение;
- три последовательных native PeerConnection loopback, три прохода
  аппаратного H.264 decode и живой Host media прошли без managed/native/audio
  drops; в живом проходе получено около 275 видео и 500 аудиокадров;
- product-gate smoke подтвердил default NativeV2, точный allow-list и
  аварийный `GAMEREMOTE_NATIVE_V2_DISABLE`;
- упакованный `GameRemote.Host` прошёл `--shared-relay-self-test`,
  `--latency-self-test` и `--files-self-test`;
- `CodeTests` contract test `AdminClient -> Server -> 6D6X6`: успешно;
- `AdminClient`, `BotStatsMTS` и полный `6D6X6` Release x64/self-contained
  собраны; `6D6X6` содержит self-contained `GameRemote.Host`, `hostfxr`,
  `hostpolicy` и `coreclr`;
- конечные output-каталоги всех трёх продуктов содержат одинаковые пять
  NativeV2-файлов; DLL имеет ABI 2.4, размер `8713216` и SHA-256
  `05913a99ebbdff449be95aa0e7165978f9a9eb09bcf912013814e9a3e7970207`;
- probe выполнен не только по staging, но и прямо по DLL внутри пакетов
  AdminClient, BotStatsMTS и `6D6X6\Service\GameRemote.Host`.

Контрольные каталоги последней сборки:

```text
AdminClient: Y:\BotStatsMTS\.test-output\NativeV2Product\AdminClient
BotStatsMTS: Y:\BotStatsMTS\.test-output\NativeV2Product\BotStatsMTS-verified
6D6X6:      D:\GameRemoteNativeV2\product-validation\6D6X6
```

При несовпадении поведения сначала сравнить SHA-256 реального
`C:\6D6X\Service\GameRemote.Host\NativeV2\GameRemote.NativeV2.dll`, а уже
потом менять код.

Оставшиеся предупреждения сборки существовали до Path 2: WinRT generator в
службе, предупреждения WPF/nullable в приложениях и NuGet advisories для
`AngleSharp`, `MailKit`, `System.Security.Cryptography.Xml`. Они не скрыли
ошибок сборки NativeV2, но должны разбираться отдельной задачей, без смешения
с полевой валидацией медиатракта.

### 0.12 Последняя первопричина и сопутствующие исправления

Последняя блокирующая причина находилась в
`GameRemote.NativeV2.Managed/native/src/gameremote_native_v2_windows_media.cpp`.
Нативная NV12 texture для Media Foundation encoder создавалась с
`D3D11_BIND_DECODER`. На рабочем D3D11 Video Processor/H.264 пути это не
гарантировало пригодность output texture для render/encode и давало запуск
без стабильного первого кадра. Bind исправлен на:

```text
D3D11_BIND_RENDER_TARGET | D3D11_BIND_VIDEO_ENCODER
```

После этой правки живой DXGI -> GPU BGRA/NV12 -> hardware H.264 -> libwebrtc
проход стал воспроизводимо отдавать видео и звук.

Дополнительно Codex:

- сделал NativeV2 production-default только для полного fingerprinted ABI 2.4
  с полным набором feature bits и `maxPeers >= 3`;
- добавил аварийное отключение и необязательный production allow-list;
- добавил строгий прямой artifact import в Host, Viewer и AdminClient;
- исправил утечку внешнего `RuntimeIdentifier=win-x64` во вложенную сборку
  `ServerManagement.Agent` из `BotStatsMTS.csproj` через `RemoveProperties`;
- убрал hardcoded `lan` из Host capability/quality-сообщений до определения
  фактического selected ICE route;
- добавил gate smoke, который проверяет default, allow-list и disable.

### 0.13 Релизный комплект, собранный Codex 21 августа 2026 года

После завершения кодовой части Path 2 Codex поднял версии и собрал единый
self-contained комплект:

```text
AdminClient  0.0.0.119
BotStatsMTS  1.0.1.125
6D6X6        2.5.109.0
GitHub tag   6d6x-2.5.109.0-admin-0.0.0.119-botstats-1.0.1.125-native-v2
```

Финальные assets и их SHA-256:

```text
AdminClient-0.0.0.119-win-x64-selfcontained.zip
  59749737ce5f04518c62ee58aa73908565a66de85752ca1acc30aa582c9514db
BotStatsMTS-1.0.1.125-win-x64-selfcontained.zip
  d2361bb23cf1fd7fcef44445e7b03758e8a6c23cf5f12d9059035b599fefb5d1
6D6X6-2.5.109.0-win-x64-selfcontained.zip
  8b7cb913f9f6e9d787357c156d06b3b60811ef626d343a494de1e9a7d33e4cc1
```

Локальный корень релиза:

```text
D:\GameRemoteNativeV2\release-20260821
```

Codex не ограничился проверкой publish-каталогов: все три ZIP были полностью
распакованы в `verification`, после чего повторно проверены версии EXE,
корневая updater-структура, полный self-contained Host runtime, helper-файлы и
SHA-256 NativeV2 DLL внутри каждого пакета. Упакованный Host версии 2.5.109.0
прошёл `--shared-relay-self-test`, `--latency-self-test` и
`--files-self-test` с кодом `0`. Probe каждой конечной DLL подтвердил ABI 2.4,
полный набор capabilities, `maxPeers=3` и product-default gate.

Updater-манифесты `AdminClient`, `BotStatsMTS` и `6D6X6` обновлены на эти
версии, URL и SHA. Тестовый `GameRemote.Host.log`, созданный самотестом, явно
исключён из финального архива 6D6X6. Следующим этапом остаётся только полевая
матрица из раздела 0.10; она не должна сопровождаться возвратом второго
медиатракта или изменением ABI без новой подтверждённой первопричины.

## 1. Коротко о решении

Выбран путь 2: собственный нативный медиатракт поверх официального
BSD-лицензированного `libwebrtc`.

Цель пути 2:

- не использовать и не копировать код Sunshine;
- не вносить GPL-компоненты в GameRemote;
- сохранить закрытый код приложения;
- получить транспорт, congestion control, jitter buffer, NACK/PLI, ICE,
  SCTP/data channels и аудиотракт промышленного WebRTC;
- оставить под нашим контролем DXGI-захват, D3D11, аппаратные кодеки,
  ввод, общий курсор, файлы и интерфейс администратора;
- мигрировать постепенно, не ломая работающий удалённый доступ.

Кодовая часть product route NativeV2 реализована. Собрана нативная DLL на
закреплённом официальном libwebrtc, реализованы PeerConnection, trickle ICE,
ICE restart, три data channel, нативные DXGI/GPU/H.264 encode/decode, rate
control, PLI, PCM/Opus/NetEq-аудиомост, bounded файловый канал и общий
capture/encoder для трёх администраторов. Найдена и устранена скрытая потеря
кадров: native `EventDispatcher` ошибочно заменял ещё не прочитанные encoded
H.264 access units последним кадром. После перехода на ordered handoff все
видеограницы сохраняют полный темп.

Рабочий удалённый стол по умолчанию использует `libwebrtc-v2`, если строгий
probe подтвердил полный fingerprinted ABI 2.4. `legacy-sipsorcery-v1`
оставлен только аварийным fallback. Post-SDP trickle, ICE restart,
multi-viewer fan-out и восстановление desktop уже реализованы и подтверждены
автоматическими контрактными и loopback-тестами. Полевая матрица из раздела
0.10 нужна для эксплуатационного подтверждения, а не для дописывания тракта.

## 2. Важное предупреждение о Git

Оба репозитория имеют большой незакоммиченный рабочий набор. Значительная часть
реального Host, Viewer и сервисного кода отображается как `untracked`.

Запрещено без отдельного согласования выполнять:

- `git reset --hard`;
- `git clean`;
- `git checkout -- <path>`;
- массовое удаление `untracked`-файлов;
- восстановление дерева только из текущего `HEAD`.

Такие команды удалят рабочую реализацию удалённого стола.

Репозитории на момент фиксации:

| Репозиторий | Путь | Ветка | HEAD |
|---|---|---|---|
| BotStatsMTS | `Y:\BotStatsMTS` | `master` | `21e9c05809b04f5825743084b352c875c2d6f075` |
| 6D6X6 | `Y:\6D6X6` | `master` | `df250b42ac929cec33ccdca4bd0609e6e2cdd86d` |

Эти хеши являются только базой Git. Рабочее дерево значительно новее HEAD.

## 3. Лицензия и границы заимствования

Официальный WebRTC распространяется под BSD-лицензией:

- `https://webrtc.googlesource.com/src/+/refs/heads/main/LICENSE`
- FAQ: `https://webrtc.googlesource.com/src/+show/main/docs/faq.md`
- Native API: `https://webrtc.github.io/webrtc-org/native-code/native-apis/`
- Windows build: `https://webrtc.github.io/webrtc-org/native-code/development/`

BSD допускает закрытое и коммерческое использование. В пакет с готовой
нативной DLL необходимо положить лицензию WebRTC и применимые third-party
notices из закреплённого checkout.

Нельзя:

- копировать исходники Sunshine;
- линковать Sunshine или другой GPL-код;
- переносить реализацию из Sunshine с косметическим переименованием;
- использовать GPL-бинарник как скрытую часть GameRemote без отдельного
  процесса и юридической оценки.

Допустимо изучать поведение похожих продуктов и официальную документацию API,
но реализация должна быть собственной.

## 4. Пользовательская цель

Нужен удалённый рабочий стол уровня Parsec, Moonlight или AnyDesk:

- стабильные 60 FPS при достаточном канале;
- минимальная задержка управления;
- Windows 10 и Windows 11;
- LAN и прямые соединения между провайдерами P1, P2 и P3;
- изображение не должно зависать при вводе или запуске игры;
- звук идёт отдельным приоритетным медиапотоком;
- передача файлов не блокирует видео и звук;
- до трёх администраторов на одном ПК;
- все администраторы управляют одним системным курсором;
- один захват экрана и одна кодирующая цепочка на ПК, без трёх независимых
  DXGI/encoder-трактов;
- переход `winlogon -> user desktop` и обратно с контролируемым reconnect;
- сигнализация и авторизация идут через BotStats, медиа не идёт через BotStats;
- отключение Viewer не блокирует и не завершает пользовательскую сессию.

Физический RTT маршрута задаёт нижний предел задержки. Целевые значения после
полной миграции: 25-50 мс в LAN и примерно 50-100 мс между провайдерами при
нормальном маршруте. Показатель 5 мс через разных провайдеров нельзя обещать,
если сам сетевой RTT выше.

## 5. Текущая рабочая архитектура

### 5.1 Сигнализация

1. AdminClient или BotStats открывает `GameRemote.Viewer`.
2. Viewer создаёт SDP offer.
3. BotStats выдаёт одноразовый токен и lease через
   `BotStatsMTS/RemoteDesktopSessionBroker.cs`.
4. Команда идёт клиенту 6D6X.
5. Служба 6D6X запускает `GameRemote.Host` в нужной Windows-сессии.
6. Host возвращает SDP answer через служебный pipe и BotStats.
7. После проверки токена медиа и data channels идут напрямую между Viewer и
   Host.

BotStats не должен проксировать видео, аудио или файлы.

### 5.2 Legacy media engine

Текущее имя движка: `legacy-sipsorcery-v1`.

Host:

- `GameRemote.Host/Session/WebRtcHostSession.cs`: PeerConnection, RTP и data
  channels через SIPSorcery;
- `GameRemote.Host/Capture/DxgiScreenCapture.cs`: DXGI Desktop Duplication;
- `GameRemote.Host/Encode/HardwareMediaFoundationEncoder.cs`: аппаратный H.264
  через Media Foundation;
- `GameRemote.Host/Encode/MediaFoundationEncoder.cs`: software fallback;
- `GameRemote.Host/Encode/D3D11BgraToNv12Converter.cs`: GPU-конвертация;
- `GameRemote.Host/Audio/WasapiLoopbackSource.cs`: loopback-звук;
- `GameRemote.Host/Input/InputInjector.cs`: системный ввод;
- `GameRemote.Host/Input/CursorTracker.cs`: положение и форма курсора;
- `GameRemote.Host/Input/VirtualGamepad.cs`: виртуальный геймпад;
- `GameRemote.Host/Transfer`: буфер обмена и файлы.

Viewer:

- `GameRemote.Viewer/Native/NativeWebRtcSession.cs`: SIPSorcery WebRTC;
- `GameRemote.Viewer/Native/NativeH264Decoder.cs`: Media Foundation decoder;
- `GameRemote.Viewer/Native/NativeVideoSurface.cs`: нативный D3D11 presenter;
- `GameRemote.Viewer/Native/NativeOpusPlayback.cs`: звук;
- `GameRemote.Viewer/Native/RemoteCursorOverlay.cs`: удалённый курсор;
- `GameRemote.Viewer/RemoteDesktopWindow.cs`: управление сессией и fallback;
- WebView2 остаётся резервным Viewer, но не является целевой архитектурой.

### 5.3 Служба 6D6X

`Y:\6D6X6\6D6X6.Service\RemoteDesktopHostManager.cs`:

- запускает отдельный Host/PeerConnection для каждого Viewer, при этом Host
  делят один DXGI/H.264 тракт через `SharedEncodedVideoRelay`;
- передаёт offer через отдельный named pipe;
- выбирает UDP-порты;
- запускает Host через пользовательский агент либо fallback;
- отслеживает Windows desktop и завершение Host;
- разрешает до трёх одновременных административных сессий.

## 6. Результаты аудита legacy-тракта

### 6.1 Ложные разрывы из-за ACK ввода

Раньше Viewer увеличивал `clientInputSequence` до фактической отправки.
Realtime-события мыши и геймпада затем могли быть объединены или отброшены.
Host физически не мог подтвердить отсутствующие номера.

Через 2,5 секунды ACK-watchdog считал это поломкой и закрывал весь WebRTC,
включая исправное видео и звук. Поэтому взятие управления или активное движение
мыши могло замораживать изображение и запускать reconnect.

Исправление в `GameRemote.Viewer/Native/NativeWebRtcSession.cs`:

- sequence назначается только после проверки открытого канала и backpressure;
- sequence записывается после реального `channel.send`;
- объединённые и отброшенные события не создают дырки;
- отсутствие ACK больше не закрывает ICE/SCTP и видеосессию;
- watchdog сбрасывает только наблюдение ACK и пишет диагностику.

Ввод не должен использоваться как индикатор здоровья видеотранспорта.

### 6.2 Ложные разрывы от WTS-событий

Служба раньше останавливала все GameRemote Host при широком наборе событий
Windows Session Change. `RemoteConnect`, `RemoteDisconnect` или событие другой
Windows-сессии могло оборвать здоровую трансляцию.

Исправление в `RemoteDesktopHostManager.NotifyDesktopChanged`:

- у каждого Host сохраняется `Process.SessionId`;
- учитываются только события, реально меняющие visible desktop;
- событие применяется только к Host той же Windows-сессии;
- событие другой сессии игнорируется;
- при отсутствии Host предпочтительный desktop не сохраняется вслепую;
- reconnect остаётся для настоящего перехода `default <-> winlogon`.

### 6.3 Не было версии медиадвижка в протоколе

Без явного поля Viewer мог запросить новый движок, а Host ответить старым. Такой
ответ выглядел бы валидным по токену и SDP.

Добавлено поле `MediaEngine` в request, offer и answer. Broker сохраняет движок
в lease и проверяет совпадение answer. Тихий downgrade запрещён.

### 6.4 Уязвимые зависимости SIPSorcery

`SIPSorcery` обновлён с `10.0.13` до `10.0.16`.

В Host и Viewer закреплены исправленные транзитивные версии:

- `System.Net.Http` 4.3.4;
- `System.Text.RegularExpressions` 4.3.1.

`dotnet list package --vulnerable --include-transitive` для Host и Viewer не
находит уязвимых пакетов.

## 7. История добавления NativeV2 до ABI 2.4

Разделы 7-14 сохраняют причины решений и результаты промежуточных этапов.
Значения ABI, SHA, статус media ownership и формулировки «ещё не реализовано»
в них исторические. Текущее состояние всегда брать из раздела 0 и из
`GameRemote.NativeV2.Managed/NATIVE_V2_ARCHITECTURE.md`.

### 7.1 Проект

`Y:\BotStatsMTS\GameRemote.NativeV2.Managed`

Проект добавлен в `BotStatsMTS.sln` и подключён как ProjectReference к Host и
Viewer.

Основные файлы:

- `NativeV2Runtime.cs`: поиск DLL, загрузка, проверка ABI и capabilities;
- `NativeV2ProductSignaling.cs`: production gate, конверт v2 и проверка
  SHA-256 нативной DLL;
- `NativeV2Engine.cs`: полный managed ABI 2.4 adapter, `SafeHandle`, callback
  lifetime и копирование payload;
- `RemoteDesktopMediaEngineNames.cs`: имена движков;
- `NATIVE_V2_ARCHITECTURE.md`: короткая архитектурная граница;
- `native/include/gameremote_native_v2.h`: стабильный C ABI;
- `native/src/gameremote_native_v2_probe.cpp`: probe-only DLL;
- `native/src/gameremote_native_v2_webrtc.cpp`: официальный libwebrtc engine;
- `native/src/gameremote_native_v2_video_bridge.cpp`: внешняя Annex-B H.264
  encoder/decoder factory;
- `native/src/gameremote_native_v2_audio_bridge.cpp`: PCM source и decoded
  PCM sink поверх штатных Opus/NetEq libwebrtc;
- `native/tests/gameremote_native_v2_loopback.cpp`: нативный loopback;
- `native/tools/build-libwebrtc.ps1`: воспроизводимая GN/Ninja сборка;
- `native/WEBRTC_REVISION.txt`: закреплённый WebRTC commit;
- `native/tools/build-probe.ps1`: сборка probe через MSVC;
- `native/tools/prepare-libwebrtc.ps1`: подготовка официального checkout.
- `native/tools/stage-native-v2.ps1`: ABI probe, атомарный staging DLL и
  manifest;
- `build/GameRemote.NativeV2.Artifact.targets`: явное подключение проверенного
  артефакта в build/publish.

Дополнительные проекты и файлы:

- `GameRemote.NativeV2.Managed.Smoke`: managed end-to-end и stress tests;
- `GameRemote.Viewer/Native/NativeV2H264DecoderBridge.cs`: изолированная
  receive-side граница NativeV2 -> MF/DXVA -> NV12 D3D11;
- `GameRemote.Host/Native/NativeV2HostVideoPipeline.cs`: реальный
  DXGI/GPU/H.264 producer для NativeV2;
- `GameRemote.Viewer/Native/NativeV2ViewerVideoPipeline.cs` и
  `NativeVideoSurface.cs`: latest-frame decode/present без WebView2;
- `GameRemote.Host/Native/NativeV2HostAudioPipeline.cs`: WASAPI loopback ->
  PCM16 48 kHz/10 ms;
- `GameRemote.Viewer/Native/NativeV2ViewerAudioPipeline.cs`: отдельный
  low-latency PCM playout с ограничением задержки;
- `GameRemote.Host/Session/NativeV2HostSessionController.cs`: продуктовая
  Host-сессия;
- `GameRemote.Viewer/Native/NativeV2ProductViewerSession.cs`: продуктовая
  Viewer-сессия;
- `GameRemote.Host --hardware-media-self-test`: реальный D3D11/MF аппаратный
  encoder fixture для проверки не на синтетических байтах.

Закреплённая ревизия WebRTC:

`e12c39e03c3dcab594f73a1e524b1f2c17dfdcb8`

Обновлять pin следует только отдельной осознанной правкой с полной пересборкой
и тестовой матрицей.

### 7.2 Имена движков

- `legacy-sipsorcery-v1`;
- `libwebrtc-v2`.

По умолчанию Viewer запрашивает `libwebrtc-v2`, но только после строгого
probe ABI 2.4, capabilities, manifest и SHA-256.
`GAMEREMOTE_NATIVE_V2_DISABLE=1` аварийно выключает маршрут, а
`GAMEREMOTE_NATIVE_V2_PCS` может ограничить его точным списком ПК. Старые
pilot-переменные маршрутом больше не управляют.

Host и Viewer выполняют `NativeV2Runtime.Probe()`, а signaling envelope v2
содержит SHA-256 загруженной DLL. Разные сборки отклоняются до запуска media.

### 7.3 ABI

Текущая версия ABI: `2.2`, число `(2 << 16) | 2`, то есть `131074`.

Главные exports:

- `gr_native_v2_get_abi_version`;
- `gr_native_v2_get_capabilities`;
- `gr_native_v2_engine_create/destroy/start`;
- remote/local SDP;
- trickle ICE candidate;
- ICE restart;
- quality update;
- D3D11 texture submission;
- external Annex-B H.264 submission;
- PCM16 submission: 48 kHz, mono/stereo, ровно 10 ms;
- data-channel send;
- keyframe request.

Capability flags:

- Probe;
- PeerConnection;
- DataChannels;
- IceRestart;
- D3D11Capture;
- D3D11ZeroCopy;
- HardwareH264Encode;
- HardwareH264Decode;
- WasapiAudio;
- MultiViewerRelay;
- EncodedH264Bridge;
- PcmOpusBridge.

`NativeV2Runtime.SessionReady` становится `true` только при совместимом ABI и
полном обязательном наборе возможностей. Сейчас probe сообщает:

```text
LibraryFound  = true
AbiCompatible = true
SessionReady  = false
Features      = Probe | PeerConnection | DataChannels | IceRestart |
                EncodedH264Bridge | PcmOpusBridge
```

Это ожидаемый и безопасный результат: транспортная DLL является настоящим
partial engine. Изолированные managed Host/Viewer уже выполняют capture,
encode/decode, present и WASAPI, но DLL намеренно ещё не заявляет эти product
capabilities и multi-viewer: rollout остаётся закрытым.

Managed probe дополнительно проверяет наличие каждого обязательного export.
DLL с ABI 2.2 без `gr_native_v2_engine_submit_encoded_h264` или
`gr_native_v2_engine_submit_pcm` отклоняется.

Для закрытого pilot route достаточно реально заявленных transport features.
`SessionReady=false` остаётся дополнительным предохранителем default rollout:
managed Host/Viewer владеют DXGI, аппаратными кодеками и WASAPI, поэтому DLL не
заявляет их как свои capability bits.

### 7.4 Правила владения памятью для будущей реализации

Эти правила необходимо сохранить в реализации DLL:

- все строки и JSON в ABI передаются как UTF-8 bytes + length;
- callback не должен блокировать worker/network/signaling threads;
- указатель payload в event действителен только во время callback;
- texture в video event заимствована на время callback;
- если DLL удерживает texture после `submit_d3d11_texture`, она обязана сделать
  `AddRef` и позже `Release`;
- после возврата `engine_destroy` callbacks для engine больше не допускаются;
- `engine_destroy` должен остановить свои потоки и освободить PeerConnection;
- исключения C++ не пересекают C ABI;
- ошибка возвращается кодом и отдельным error event;
- callbacks могут приходить с разных libwebrtc threads, managed-слой обязан
  маршалить UI-события на Dispatcher.

### 7.5 Что уже доказано тестами

Нативный loopback проверяет:

- offer/answer;
- trickled host candidates;
- открытие трёх data channels;
- двунаправленные данные;
- encoded H.264 RTP;
- PCM16 -> Opus -> RTP -> NetEq -> PCM16 с ненулевым RMS;
- ненулевые rate-control target bitrate/FPS;
- PLI и следующий IDR;
- ICE restart с новым `ice-ufrag`;
- отсутствие callback после `destroy`.

Managed loopback дополнительно проверяет layout ABI, `SafeHandle`, pinned
delegate/context и завершение всех managed event readers. В одном процессе
пройдено 30 последовательных create/destroy циклов. Отдельный тест encoded
video handoff подтверждает, что начальный IDR и следующие 12 delta access units
приходят строго по порядку и без managed drop. Latest-only разрешён только для
уже декодированной D3D11 texture перед показом.

Managed audio test отдельно подтверждает строгую проверку PCM-разметки,
запрет host-only API на Viewer и применение audio bitrate. Реальный WASAPI
smoke дал `submitted=31`, `received=148`, `hostBacklogDrops=0`,
`managedDrops=0`. Viewer audio pump обязан запускаться до `engine.Start()`;
иначе NetEq успевает заполнить очередь ещё во время SDP/ICE.

Реальный аппаратный тест на текущем стенде:

```text
GPU: AMD Radeon RX 6700 XT
Encoder: AMD AMF via Media Foundation (AMDh264Encoder)
Source: D3D11 BGRA -> D3D11 Video Processor -> NV12 -> H.264
Stream: 640x360, 60 FPS, 30 Annex-B access units
Transport: NativeV2/libwebrtc RTP
Decoder: Microsoft H264 Video Decoder MFT, sync-DXVA
Output: NV12 D3D11 texture
Result: повторные полные loopback, 30/30 decoded frames в каждом
```

Совместный live-тест на одном PeerConnection:

```text
Run 1: video captured/encoded/decoded=300/300/313, audio=500/500
Run 2: video captured/encoded/decoded=300/300/309, audio=501/500
Run 3: video captured/encoded/decoded=300/300/314, audio=500/501
Native stages: source/encoder/RTP/decoder/sink=299/299/299/299/299
Drops: managed=0, decoder=0, queue resets=0, audio=0
Recovery: PLI -> fresh IDR, audio продолжает идти
```

Значение decoded может быть немного выше captured в контрольном окне из-за
границ измерительного интервала и уже находящихся в тракте кадров. Критический
инвариант — одинаковый накопительный темп всех пяти нативных границ и отсутствие
потери зависимых H.264 access units.

Product signaling tests дополнительно отклоняют raw SDP, завершённый ICE без
кандидатов, offer/answer substitution, устаревший envelope и несовпадающий
SHA-256 DLL. SDP-first envelope с нулём кандидатов и
`iceGatheringComplete=false` разрешён: smoke доставляет кандидаты отдельным
trickle-шагом, устанавливает соединение, затем меняет `ice-ufrag` и повторяет
offer/answer/candidates для нового поколения в том же PeerConnection.

libwebrtc штатно удаляет AUD NAL type 9 при packetize/depacketize. SPS, PPS,
IDR, delta и filler NAL проверяются побайтно. Это единственная разрешённая
нормализация теста.

Для 640x360 decoder выдаёт coded texture 640x368. Это macroblock alignment,
не изменение видимой геометрии. `NativeV2DecodedFrameInfo` хранит отдельно
visible 640x360 и texture 640x368; presenter обязан crop по visible size.

Отрицательные проверки подтверждают:

- delta frame до первого IDR не попадает в decoder;
- при этом отправляется key-frame request;
- смена разрешения по delta frame не пересоздаёт decoder;
- новое поколение начинается только с IDR;
- dispose не оставляет callback после завершения engine.

## 8. Изменённые файлы NativeV2

BotStatsMTS:

- `CommonModels/RemoteDesktopContracts.cs`;
- `BotStatsMTS/RemoteDesktopSessionBroker.cs`;
- `CodeTests/Program.cs`;
- `GameRemote.Host/GameRemote.Host.csproj`;
- `GameRemote.Host/Program.cs`;
- `GameRemote.Host/Session/WebRtcHostSession.cs`;
- `GameRemote.Viewer/GameRemote.Viewer.csproj`;
- `GameRemote.Viewer/RemoteDesktopWindow.cs`;
- `GameRemote.Viewer/Native/NativeWebRtcSession.cs`;
- `GameRemote.Viewer/Native/NativeV2H264DecoderBridge.cs`;
- `GameRemote.Viewer/Native/NativeV2ProductViewerSession.cs`;
- `GameRemote.Host/Session/NativeV2HostSessionController.cs`;
- `GameRemote.Host/Session/SharedEncodedVideoRelay.cs`;
- `GameRemote.Host/Transfer/FileTransferReceiver.cs`;
- `GameRemote.Host/Native/NativeV2HostVideoPipeline.cs`;
- `GameRemote.Host/Native/NativeV2HostAudioPipeline.cs`;
- `GameRemote.NativeV2.Managed/NativeV2Engine.cs`;
- `GameRemote.NativeV2.Managed/native/include/gameremote_native_v2.h`;
- `GameRemote.NativeV2.Managed/native/src/gameremote_native_v2_webrtc.cpp`;
- `GameRemote.NativeV2.Managed/native/src/gameremote_native_v2_video_bridge.h`;
- `GameRemote.NativeV2.Managed/native/src/gameremote_native_v2_video_bridge.cpp`;
- `GameRemote.NativeV2.Managed/native/src/gameremote_native_v2_windows_media.h`;
- `GameRemote.NativeV2.Managed/native/src/gameremote_native_v2_windows_media.cpp`;
- `GameRemote.NativeV2.Managed/build/GameRemote.NativeV2.Artifact.targets`;
- `GameRemote.NativeV2.Managed/NATIVE_V2_ARCHITECTURE.md`;
- `GameRemote.NativeV2.Managed.Smoke/Program.cs`;
- `BotStatsMTS/BotStatsMTS.csproj` (выравнивание
  `Microsoft.Extensions.DependencyInjection.Abstractions` до `10.0.11` для
  совместимости с Viewer/SIPSorcery);
- `GameRemote.NativeV2.Managed/**`;
- `GameRemote.NativeV2.Managed.Smoke/**`;
- `BotStatsMTS.sln`.

6D6X6:

- `6D6X6.Service/RemoteDesktopHostManager.cs`.

В `CodeTests/Program.cs` добавлена проверка, что answer другого media engine
отклоняется и не выполняет тихий downgrade.

## 9. Сборочная среда и проверенные команды

Среда:

- Windows 10 build 19045;
- Visual Studio Community 2022 17.12;
- установлен MSVC x64;
- .NET SDK 9.0.101;
- проекты собираются под .NET 8, служба 6D6X под .NET Framework 4.8;
- `depot_tools`: `D:\GameRemoteNativeV2\depot_tools`;
- официальный checkout: `D:\GameRemoteNativeV2\webrtc\src`;
- артефакты: `D:\GameRemoteNativeV2\webrtc\artifacts\GameRemote.NativeV2`;
- portable Windows SDK:
  `D:\GameRemoteNativeV2\WindowsKits10-28000\Windows Kits\10`;
- `global.json` отсутствует.

Проверенные команды:

```powershell
cd Y:\BotStatsMTS
dotnet build GameRemote.Viewer\GameRemote.Viewer.csproj -c Release --no-restore
dotnet build GameRemote.Host\GameRemote.Host.csproj -c Release -r win-x64 --no-restore
dotnet build AdminClient\AdminClient.csproj -c Release --no-restore
dotnet build CommonModels\CommonModels.csproj -c Release --no-restore
dotnet build GameRemote.NativeV2.Managed\GameRemote.NativeV2.Managed.csproj `
  -c Release --no-restore
dotnet build GameRemote.NativeV2.Managed.Smoke\GameRemote.NativeV2.Managed.Smoke.csproj `
  -c Release --no-restore

dotnet list GameRemote.Host\GameRemote.Host.csproj package --vulnerable --include-transitive
dotnet list GameRemote.Viewer\GameRemote.Viewer.csproj package --vulnerable --include-transitive

& .\GameRemote.NativeV2.Managed\native\tools\build-probe.ps1 `
  -OutputDirectory "$env:TEMP\GameRemoteNativeV2Probe"

& .\GameRemote.NativeV2.Managed\native\tools\build-libwebrtc.ps1 `
  -CheckoutRoot D:\GameRemoteNativeV2\webrtc `
  -DepotToolsPath D:\GameRemoteNativeV2\depot_tools `
  -WindowsSdkPath 'D:\GameRemoteNativeV2\WindowsKits10-28000\Windows Kits\10'

$env:GAMEREMOTE_H264_SELFTEST_OUTPUT = `
  'D:\GameRemoteNativeV2\hardware-selftest.grh264'
dotnet .\GameRemote.Host\bin\Release\net8.0-windows10.0.17763.0\win-x64\GameRemote.Host.dll `
  --hardware-media-self-test

dotnet .\GameRemote.NativeV2.Managed.Smoke\bin\Release\net8.0-windows10.0.17763.0\GameRemote.NativeV2.Managed.Smoke.dll `
  'D:\GameRemoteNativeV2\webrtc\artifacts\GameRemote.NativeV2\GameRemote.NativeV2.dll' `
  30

dotnet .\GameRemote.NativeV2.Managed.Smoke\bin\Release\net8.0-windows10.0.17763.0\GameRemote.NativeV2.Managed.Smoke.dll `
  'D:\GameRemoteNativeV2\webrtc\artifacts\GameRemote.NativeV2\GameRemote.NativeV2.dll' `
  1 `
  'D:\GameRemoteNativeV2\hardware-selftest.grh264' `
  5 `
  --live-host-video

dotnet .\GameRemote.NativeV2.Managed.Smoke\bin\Release\net8.0-windows10.0.17763.0\GameRemote.NativeV2.Managed.Smoke.dll `
  'D:\GameRemoteNativeV2\webrtc\artifacts\GameRemote.NativeV2\GameRemote.NativeV2.dll' `
  1 `
  --live-host-audio

dotnet .\GameRemote.NativeV2.Managed.Smoke\bin\Release\net8.0-windows10.0.17763.0\GameRemote.NativeV2.Managed.Smoke.dll `
  'D:\GameRemoteNativeV2\webrtc\artifacts\GameRemote.NativeV2\GameRemote.NativeV2.dll' `
  1 `
  'D:\GameRemoteNativeV2\hardware-selftest.grh264' `
  3 `
  --live-host-media

& .\GameRemote.NativeV2.Managed\native\tools\stage-native-v2.ps1 `
  -NativeLibraryPath 'D:\GameRemoteNativeV2\webrtc\artifacts\GameRemote.NativeV2\GameRemote.NativeV2.dll' `
  -OutputDirectory 'D:\GameRemoteNativeV2\staged\GameRemote.NativeV2' `
  -WebRtcSourceRoot 'D:\GameRemoteNativeV2\webrtc\src'

dotnet build .\AdminClient\AdminClient.csproj -c Release --no-restore `
  -p:GameRemoteNativeV2ArtifactPath='D:\GameRemoteNativeV2\staged\GameRemote.NativeV2\GameRemote.NativeV2.dll' `
  -p:GameRemoteNativeV2RequireArtifact=true

dotnet run --project .\GameRemote.NativeV2.Managed.Smoke\GameRemote.NativeV2.Managed.Smoke.csproj `
  -c Release -- `
  'D:\GameRemoteNativeV2\staged\GameRemote.NativeV2\GameRemote.NativeV2.dll' 1

dotnet run --project .\CodeTests\BotStatsMTS.CodeTests.csproj `
  -c Release -p:Platform=x64

cd Y:\6D6X6
dotnet restore 6D6X6.Service\6D6X6.Service.csproj `
  -r win-x64 -p:Platform=x64
dotnet build 6D6X6.Service\6D6X6.Service.csproj `
  -c Release -r win-x64 -p:Platform=x64 --no-restore `
  -p:GameRemoteNativeV2ArtifactPath='D:\GameRemoteNativeV2\staged\GameRemote.NativeV2\GameRemote.NativeV2.dll' `
  -p:GameRemoteNativeV2RequireArtifact=true
dotnet restore 6D6X6\6D6X6.csproj `
  -r win-x64 -p:Platform=x64
dotnet build 6D6X6\6D6X6.csproj `
  -c Release -r win-x64 -p:Platform=x64 --no-restore `
  -p:GameRemoteNativeV2ArtifactPath='D:\GameRemoteNativeV2\staged\GameRemote.NativeV2\GameRemote.NativeV2.dll' `
  -p:GameRemoteNativeV2RequireArtifact=true
```

`RuntimeIdentifier` здесь задаётся явно. Без него локальные stale assets могут
подхватить `win-x86`, хотя служба, CaptureWorker и папочный Host поставляются
как x64.

Результаты:

- Viewer: успешно, 0 ошибок;
- Host: успешно, 0 ошибок, один существующий warning
  `WinRTAotSourceGenerator`;
- AdminClient: успешно, 0 ошибок, существующие nullable warnings;
- CommonModels: успешно;
- 6D6X6.Service: успешно, вместе с публикацией папочного Host;
- полный 6D6X6-клиент: успешно в явной конфигурации `win-x64/x64`;
- строгая x64-сборка 6D6X6 содержит пять NativeV2-файлов в
  `Service\GameRemote.Host\NativeV2`; SHA-256 DLL совпадает со staged artifact;
- native probe: успешно собран MSVC и загружен managed probe;
- официальный libwebrtc transport: успешно собран GN/Ninja;
- native loopback: 20/20 независимых запусков;
- managed create/destroy loopback: 30/30;
- AMD AMF -> libwebrtc -> MF/DXVA -> NV12 D3D11: 5/5, по 30/30 кадров;
- live DXGI -> AMD AMF -> libwebrtc -> MF/DXVA: успешно, PLI и изменение
  bitrate применяются без пересоздания PeerConnection;
- WASAPI -> PCM16 -> Opus -> NetEq -> PCM16: успешно, потери managed audio
  queue отсутствуют;
- три независимых полных `--live-host-media` прогона на одном PeerConnection:
  в каждом контрольном окне native stages сохранили темп
  `299/299/299/299/299` source/encoder/RTP/decoder/sink, видео удержало около
  `300 captured / 300 encoded / 312-313 decoded`, аудио около `500/500`;
  managed/decoder/audio drops и decoder queue resets отсутствуют;
- расширенный live-smoke при ошибке печатает selected route, RTT, available
  bitrate, candidate-pair packets/discards, outbound/inbound video frames/FPS,
  packets/bytes, loss, NACK и PLI. Один первоначальный прогон показал только
  24-27 FPS на приёмнике при исправных capture/encode, поэтому требование
  физического soak не снято; три последующих независимых прогона прошли 60 FPS;
- signaling envelope v2 проверяет SHA-256 DLL на Host и Viewer;
- staged DLL
  `d58dcac48f3a6b8505c300899414e84e17bd3a9590175d54c53e7ec20086fc50`,
  8 714 752 байт; manifest schema 2 закрепляет ABI 2.4, WebRTC revision,
  SHA-256 DLL, лицензии и notices;
- Host, Viewer, AdminClient, BotStats, 6D6X6.Service и полный 6D6X6 содержат
  идентичные по SHA-256 пять NativeV2-файлов: DLL, manifest, revision, license
  и third-party notices;
- third-party notices сформированы для 19 компонентов;
- Host, Viewer, AdminClient и BotStats собраны в Release с обязательным
  `GameRemoteNativeV2RequireArtifact=true`, ошибок нет;
- Host/Viewer NuGet audit: уязвимостей не найдено.
- полный ABI 2.4 loopback дополнительно выполнен непосредственно по DLL из
  `BotStatsMTS\bin\Release\net8.0-windows\NativeV2`, а не только по staged
  каталогу.

Проверено, что в папку Host службы попадает managed adapter. Нативная DLL
попадает в `NativeV2\GameRemote.NativeV2.dll` через единый проверенный
artifact path; строгая Release/publish-сборка завершается ошибкой, если artifact
отсутствует или его SHA-256 не совпадает.
Общий MSBuild target после `CopyFilesToOutputDirectory` проверяет весь комплект
из пяти файлов. BotStats импортирует этот контракт напрямую, поскольку content
из транзитивного `ProjectReference` через Viewer не гарантирован; пустая папка
`NativeV2` теперь завершает строгую сборку ошибкой.

### 9.1 Сборка BotStats

BotStats полностью собран в Release, включая финальный WPF-шаг. Для устранения
`NU1605` его прямая ссылка
`Microsoft.Extensions.DependencyInjection.Abstractions 10.0.8` выровнена до
`10.0.11`, которую уже требует транзитивный граф Viewer/SIPSorcery. Остальные
существующие предупреждения проекта не появились из-за NativeV2.

## 10. История этапов NativeV2 до завершения Path 2

### Шаг 1. Официальный checkout — выполнено

Checkout, `depot_tools`, GN/Ninja и portable Windows SDK уже подготовлены вне
репозитория. Команда повторной подготовки при необходимости:

```powershell
cd Y:\BotStatsMTS
& .\GameRemote.NativeV2.Managed\native\tools\prepare-libwebrtc.ps1 `
  -CheckoutRoot D:\GameRemoteNativeV2\webrtc `
  -DepotToolsPath D:\GameRemoteNativeV2\depot_tools `
  -Sync
```

Checkout большой. Не размещать его внутри `Y:\BotStatsMTS`, не добавлять в Git
и не класть целиком в updater.

### Шаг 2. GN target и C ABI shim — выполнено

Нужен собственный shared-library target, который компилируется внутри
закреплённого WebRTC checkout и экспортирует только
`gameremote_native_v2.h`.

Не экспортировать C++ классы libwebrtc. Managed Host и Viewer не должны видеть
`rtc::`, `webrtc::`, `scoped_refptr` или Chromium STL-ABI.

Внутри shim создать:

- `rtc::Thread` для network;
- `rtc::Thread` для worker;
- `rtc::Thread` для signaling;
- `webrtc::PeerConnectionFactoryInterface`;
- engine state machine;
- observer для PeerConnection;
- observer для data channels;
- очередь неблокирующих C events;
- безопасное завершение потоков.

Loopback offer/answer, trickle ICE, restart ICE, data channels и H.264 уже
реализованы. Product route подключён как default после строгого
artifact/capability probe; Legacy сохранён как аварийный fallback.

### Шаг 3. Data channels — транспорт выполнен

Минимум три логических канала:

1. `control`: ordered/reliable, клавиши, кнопки, команды, clipboard, auth;
2. `input-fast`: unordered/unreliable либо ограниченный retransmit, только
   последнее состояние mouse/gamepad;
3. `files`: ordered/reliable с application-level flow control.

Файловый канал должен иметь низший приоритет и не создавать задержку media или
input. Нельзя считать потерю realtime input причиной закрытия PeerConnection.

Открытие и двунаправленная передача по всем трём каналам подтверждены. Control
и realtime input handlers подключены в закрытом product route. Product files
handler намеренно выключен до отдельного low-priority flow-control теста;
длительные loss/backpressure tests ещё не выполнены.

### Шаг 4. Видео Host — изолированный adapter выполнен

Целевая цепочка:

```text
DXGI Desktop Duplication
  -> latest-frame mailbox size 1
  -> D3D11 texture
  -> GPU color conversion
  -> NVENC / Intel QSV / AMD AMF
  -> libwebrtc encoded image / RTP
```

Требования:

- очередь сырой texture до encoder может быть latest-frame mailbox размером 1;
- encoded H.264 access units после encoder передаются строго по порядку до
  decoder: отдельный P-кадр нельзя удалить без потери dependency chain;
- если требуется сброс encoded backlog, сбрасывается поколение целиком,
  запрашивается IDR и новое поколение начинается только с key frame;
- после decoder старый непросмотренный кадр заменяется свежей D3D11 texture;
- никаких полных BGRA-копий через RAM в штатном аппаратном режиме;
- encoder backend обязательно пишется в лог;
- software fallback остаётся, но явно помечается;
- IDR запрашивается по PLI/FIR, потере decoder state или длительному отсутствию
  изображения, но не на каждый локальный drop;
- при плохом канале сначала снижать bitrate, затем resolution, FPS последним;
- смена quality не должна разрывать PeerConnection.

`NativeV2HostVideoPipeline` связывает существующие DXGI Desktop Duplication,
D3D11 BGRA->NV12, аппаратный Media Foundation encoder и
`NativeV2EncodedH264Source`. Он обрабатывает `VideoRateControl` и
`KeyFrameRequest`, удерживает display active во время теста и не копирует
legacy PeerConnection state machine. Полный live loopback подтверждён на AMD
AMF. Pipeline подключён к `NativeV2HostSessionController` закрытого pilot
route.

### Шаг 5. Видео Viewer — изолированный decode/present выполнен

Целевая цепочка:

```text
libwebrtc RTP/jitter/NACK
  -> hardware H.264 decoder
  -> latest decoded D3D11 texture
  -> NativeVideoSurface/D3D11 present
```

Сохранить aspect ratio и letterbox. Координаты ввода должны преобразовываться
из области фактического изображения, а не полного WPF-окна.

WebView2 допускается только как резерв на время миграции. Целевой Viewer должен
быть полностью нативным.

`NativeV2ViewerVideoPipeline` читает отдельный `VideoEvents` pump, передаёт
H.264 в `NativeV2H264DecoderBridge` и публикует только свежую retained NV12
D3D11 texture в `NativeVideoSurface`. Проверены visible/coded dimensions,
ожидание IDR при смене поколения, bounded decoder queue, aspect-ratio presenter
и реальный nonblank swap-chain. Pipeline подключён к
`NativeV2ProductViewerSession` закрытого pilot route.

### Шаг 6. Аудио — изолированный тракт выполнен

- ABI 2.2 принимает PCM16 48 kHz mono/stereo блоками по 10 ms;
- `PcmAudioSource` отдаёт звук штатному Opus encoder libwebrtc;
- Viewer получает декодированный NetEq PCM через отдельный bounded callback;
- Host захватывает default render endpoint через WASAPI loopback и приводит к
  48 kHz mono без общей очереди с видео;
- Viewer использует отдельный WASAPI event playout, fallback на WaveOut и
  сбрасывает backlog выше 120 ms до целевых 60 ms;
- echo/AGC/noise suppression для desktop loopback отключены;
- mute выставляет реальную громкость output в ноль;
- video quality меняется независимо, audio bitrate задаётся RTP sender;
- синтетический и реальный WASAPI loopback прошли без managed drop.

Классы подключены к единым Host/Viewer product controllers. Совместный live
тест подтверждает, что аудио не блокирует видео и продолжает идти во время
PLI/IDR recovery.

### Шаг 7. Один capture для нескольких Viewer

Сейчас служба запускает отдельный Host на каждого администратора. Целевая
архитектура должна перейти к одному capture/encoder owner на ПК и fan-out до
трёх PeerConnection.

Медленный Viewer не должен задерживать остальных. Для каждого Viewer нужна
своя congestion state и собственная ordered encoded stream boundary. При
backpressure нельзя вырезать одиночные P-кадры; нужно начать новое поколение с
IDR. Если разные каналы требуют разные bitrate/resolution, допускаются
отдельные encoder layers, но не отдельный DXGI capture.

### Шаг 8. Managed adapter и закрытый product controller — выполнено

Реализованный `NativeV2Engine` уже выполняет:

- динамическую загрузку и проверку exports;
- удержание callback delegate и context от GC;
- немедленное копирование callback payload;
- `SafeHandle`-destroy при обычном завершении и исключениях;
- удержание DLL до уничтожения engine;
- завершение managed event reader после destroy.

Product controller выполняет обязательные правила:

- `libwebrtc-v2` включается по умолчанию только после полного ABI/capability/fingerprint probe;
- fallback создаёт отдельную legacy lease/session и не смешивает SDP;
- envelope v2 проверяет тип description и SHA-256 DLL обеих сторон;
- медиа запускается только после проверки токена и открытия control channel.

Managed events уже разделены на четыре независимых потока:

- `Events`: signaling, state, stats, errors и reliable data;
- `VideoEvents`: ordered/lossless encoded H.264 до decoder;
- `AudioEvents`: короткое bounded окно;
- `RealtimeEvents`: latest-only mouse/gamepad state.

Product controller читает их отдельными быстрыми pump и не объединяет обратно
в одну блокирующую UI-очередь.

Если V2 уже начал сигнализацию и упал, создать новый session ID и новый legacy
offer. Нельзя смешивать SDP двух движков в одной lease.

### Шаг 9. Post-SDP trickle и ICE restart — реализовано, полевой soak впереди

Рабочий маршрут сигнализации:

```text
AdminClient/BotStats Viewer
  -> MessageType.RemoteDesktopSessionSignal
  -> BotStats RemoteDesktopSessionBroker
  -> 6D6X6 Service ServerChannel
  -> RemoteDesktopHostManager
  -> существующий Host named pipe
  -> NativeV2HostSessionController
  -> ответ тем же маршрутом
```

Контракт каждого сообщения содержит `sessionId`, `targetPc`,
`viewerInstanceId`, одноразовый token, `MediaEngine=libwebrtc-v2`, ICE
generation, монотонный sequence, kind (`trickle` или `restart-offer`), локальный
и удалённый candidate cursor. Значение `MessageType` добавлено только в конец
enum, поскольку развёрнутые клиенты сериализуют его числом.

Инварианты:

- начальное SDP может уйти без кандидатов только пока gathering не завершён;
- кандидаты поколений не смешиваются;
- restart начинается с cursor `0/0` и нового `generation`;
- один sequence с тем же payload идемпотентно возвращает кэшированный ответ;
- тот же sequence с другим payload, пропуск sequence, старое поколение,
  неправильный token/Viewer/media engine отклоняются;
- каждый ICE-кандидат проверяется как ограниченный по размеру JSON-конверт с
  `sdpMid`, неотрицательным `sdpMLineIndex` и непустым `candidate`; одинаковые
  кандидаты внутри одного пакета отклоняются, а не удаляются молча;
- Viewer, BotStats broker и Host применяют одну и ту же проверку кандидатов;
  отклонённый ответ Host не сдвигает cursor/sequence и может быть безопасно
  повторён с исправленным payload;
- `RestartIce()` создаёт единственный restart-offer; второй обычный offer не
  создаётся;
- Viewer проверяет late candidates каждые 250 мс, начинает restart после двух
  секунд `Disconnected`, делает не более двух попыток с паузой минимум три
  секунды и завершает сессию после 20 секунд без восстановления.

Подтверждено локально:

- SDP-first offer/answer с нулевым начальным набором кандидатов;
- двусторонняя доставка кандидатов после SDP;
- новое ICE-поколение с новым `ice-ufrag` в тех же native engine;
- control data channel работает после restart;
- компактный официальный `PeerConnection::GetStats()` даёт выбранную
  candidate pair, RTT и RTP-счётчики до и после restart того же
  `PeerConnection`;
- broker cursor/sequence/generation validation, отрицательные token/replay
  проверки, повреждённые/повторные ICE-кандидаты и replay ответа после commit;
- три последовательных managed ABI 2.2 loopback прошли.

Это ещё не означает готовность default rollout. Не выполнены физическое
разрушение маршрута на тестовом ПК, длительный LAN/P1-P2/P1-P3 soak и
reboot/desktop-switch матрица.

## 11. Rollout gates

NativeV2 нельзя делать движком по умолчанию, пока не пройдены все ворота:

1. ABI probe и несовместимая версия.
2. PeerConnection loopback 30 минут без утечки handles/threads.
3. Ordered и unreliable data channels под потерями и backpressure.
4. ICE restart без пересоздания UI-окна.
5. H.264 encode/decode loopback на NVIDIA, Intel, AMD и software fallback.
6. Windows 10 и Windows 11.
7. WASAPI audio одновременно с 60 FPS video.
8. Один, два и три Viewer.
9. Передача большого файла при активной игре.
10. `default -> winlogon -> default`.
11. Перезагрузка ПК и автоматическое восстановление Viewer.
12. LAN, P1-P2, P1-P3, одинаковый провайдер и разные NAT/UPnP маршруты.
13. Длительный soak test не менее 2 часов на каждом основном GPU.

Кодовая часть этих шагов уже выполнена: NativeV2 стал default со строгим
probe, а fallback покрывает не только SDP, но и два неудачных цикла recovery
живого медиатракта. Остались сбор полевой телеметрии и матрица из раздела 0.10.

## 12. Обязательная телеметрия

Сетевая часть этого списка уже реализована как схема
`gameremote.native-v2.network-stats.v1`. Нативный poller раз в секунду получает
официальный `RTCStatsReport` на signaling-thread и передаёт только компактный
снимок. Poller останавливается до разрушения `PeerConnection`, callback держит
weak-ссылку, поэтому закрытие сессии не оставляет фонового владельца engine.

Viewer и Host получают:

- фактическую selected candidate pair и число её смен;
- local/remote candidate type, protocol, address и port;
- ICE/DTLS state, RTT и available incoming/outgoing bitrate;
- packets/bytes/retransmit/loss, jitter и среднюю задержку jitter buffer;
- frames/FPS/keyframes/drops и NACK/PLI/FIR отдельно для audio/video и
  inbound/outbound;
- фактическое имя encoder/decoder implementation, когда libwebrtc его
  предоставляет.

Loopback требует ненулевые RTP-счётчики обеих сторон и новый снимок после ICE
restart. Это готовая измерительная основа для физического LAN/P1-P2/P1-P3
soak, но не замена такому soak.

Для каждого видеокадра или агрегированного окна нужны:

- capture timestamp;
- encode queue wait;
- encode duration;
- encoded size и keyframe flag;
- RTP enqueue/send;
- RTT, jitter, packet loss, NACK, PLI/FIR;
- receive/jitter buffer;
- decode queue и decode duration;
- present queue и present timestamp;
- captured/submitted/encoded/sent/received/decoded/presented/dropped;
- причина каждого drop;
- выбранная ICE candidate pair;
- LAN/internet определяется по фактической selected pair, а не по адресу
  BotStats или предположению Viewer.

В UI показывать как минимум:

- целевой FPS / фактический presented FPS;
- bitrate;
- RTT;
- capture-to-present latency, если clocks синхронизированы или есть
  корректная RTT-компенсация;
- encoder/decoder backend;
- выбранный маршрут.

## 13. Тестовая матрица

Проверять не только статичный рабочий стол:

| Сценарий | Что контролировать |
|---|---|
| Статичный desktop | тракт не должен ошибочно считать отсутствие новых кадров зависанием |
| Быстрая мышь | нет ACK-разрыва, нет очереди старых координат |
| Перетаскивание окна | 60 FPS без накопления latency |
| Браузер и музыка | чистый звук, видео не деградирует от аудиобуфера |
| Запуск EA/Steam игры | нет разрыва при смене нагрузки GPU и fullscreen |
| Высокое движение в игре | bitrate adaptation без IDR-петли |
| Alt+Tab и UAC | правильный desktop и координаты |
| Lock/unlock | reconnect только при реальной смене desktop |
| Два/три администратора | один системный курсор, медленный Viewer не тормозит других |
| Передача файла | media/input сохраняют приоритет |
| Потеря пакетов | свежий кадр важнее воспроизведения старой очереди |

## 14. Что нельзя сломать при продолжении

- Host остаётся framework-dependent. Self-contained apphost ранее завершался
  с `0xC0000142 STATUS_DLL_INIT_FAILED` до входа в `Main` на части ПК.
- Логи Host остаются в `C:\6D6X\Logs\GameRemote.Host.log`.
- Служба должна запускать Host именно в интерактивной Windows-сессии.
- Выход из удалённого стола не должен блокировать Windows и не должен выполнять
  logoff пользователя.
- Подтверждение занятого игроком ПК сохраняется.
- До трёх администраторов и общий системный курсор сохраняются.
- Одноразовый токен и проверка media engine сохраняются.
- Полезные legacy-исправления нельзя удалить вместе с SIPSorcery до завершения
  V2 rollout.
- Не запрашивать reconnect из-за одного потерянного ACK, RTP packet или
  временного отсутствия нового кадра на статичном desktop.
- Не возвращать latest-only/coalescing в очередь encoded H.264 до decoder.
  Latest-only допустим только для сырого capture mailbox и decoded texture.

## 15. Точная стартовая задача для следующего исполнителя

Не повторять PeerConnection, H.264 bridge, DXGI/GPU encode/decode, shared
relay, product controllers, PCM/Opus/WASAPI, file channel, fingerprint,
desktop persistence или staging. Они уже реализованы.

Следующая работа начинается с полевой валидации:

1. Прочитать раздел 0 и
   `GameRemote.NativeV2.Managed/NATIVE_V2_ARCHITECTURE.md`.
2. Проверить dirty worktree и не удалять untracked-файлы.
3. Проверить, что в Host и Viewer загружен один SHA из раздела 0.9.
4. Убедиться, что `GAMEREMOTE_NATIVE_V2_DISABLE` не задан; при необходимости
   ограничить полевую проверку одним ПК через `GAMEREMOTE_NATIVE_V2_PCS`.
5. Пройти LAN Windows 10 и Windows 11, записать selected pair, RTT, loss,
   capture/encode/decode/present FPS, backend адаптера и причины recovery.
6. Подключить второго и третьего Viewer. Искусственно замедлить одного и
   доказать, что два других не теряют FPS и latency.
7. Завершить owner Host и доказать автоматический reconnect с избранием нового
   owner без закрытия Viewer window.
8. Выполнить lock/unlock, logoff/logon и reboot. Проверить реальный desktop и
   `%ProgramData%\6D6X6\State\GameRemote.desktop`.
9. Передавать большой файл одновременно с игрой, 60 FPS, звуком и вводом.
   Проверить, что buffered amount ограничен и нет роста media latency.
10. После LAN перейти к P1-P2, P1-P3 и P2-P3. Не считать адрес BotStats
    маршрутом media, смотреть только selected ICE pair.
11. Не менять default с `LegacyV1`, пока вся матрица не подтверждена.

Если тест обнаружит проблему, таблица 0.4 указывает первый проект и файл для
каждой границы. Не смешивать исправления LegacyV1 и NativeV2.

## 16. Состояние artifact и публикации

Архивы и GitHub Release в рамках этого этапа не публиковались. Проверенная DLL
локально staged в `D:\GameRemoteNativeV2\staged\GameRemote.NativeV2` вместе
с manifest, лицензией и notices.

```text
ABI: 2.4
SHA-256: d58dcac48f3a6b8505c300899414e84e17bd3a9590175d54c53e7ec20086fc50
Size: 8714752
WebRTC revision: e12c39e03c3dcab594f73a1e524b1f2c17dfdcb8
```

Перед будущей публикацией обязательны:

- строгая сборка BotStats, AdminClient, 6D6X6.Service и 6D6X6 одним artifact;
- успешный native loopback и managed ABI 2.4 loopback;
- совпадение SHA DLL во всех конечных output/publish папках;
- полный набор manifest/revision/license/notices;
- явное различие LegacyV1 и NativeV2 в release notes;
- строгий default gate и работоспособность аварийного disable/allow-list;
- проверка состава готовых архивов и updater extraction.

Отдельный известный риск общего BotStats dependency graph: NuGet сообщает об
уязвимостях в текущих версиях `AngleSharp`, `MailKit` и
`System.Security.Cryptography.Xml`. Это не вызвано NativeV2 и не изменяет его
маршрутизацию, но требует отдельной security-задачи до широкого внешнего
распространения продукта.
