# GameRemote NativeV2 media plane

Подробная карта проектов, runtime-вызовов, сборки и полевых проверок находится
в `..\GAMEREMOTE_NATIVE_V2_HANDOFF_RU.md`, прежде всего в разделе 0.

## Решение

`GameRemote.NativeV2.dll` является собственным C ABI-адаптером над
официальным BSD-лицензированным `libwebrtc`. Ревизия закреплена в
`native\WEBRTC_REVISION.txt`. Код Sunshine и другие GPL-компоненты в тракт не
входят.

Managed Host и Viewer не используют C++ типы WebRTC напрямую. Через границу
проходят только versioned ABI-структуры, UTF-8/byte payload, callback и
удерживаемые D3D11 texture leases. Это изолирует приложение от нестабильного
Chromium C++ ABI.

## Текущее состояние

Версия ABI: `2.4` (`0x00020004`).

Нативная DLL заявляет и реализует:

- `Probe`;
- `PeerConnection`;
- `DataChannels`;
- `IceRestart`;
- `D3D11Capture`;
- `D3D11ZeroCopy`;
- `HardwareH264Encode`;
- `HardwareH264Decode`;
- `MultiViewerRelay` как поддерживаемую product-возможность;
- `EncodedH264Bridge`;
- `PcmOpusBridge`.

`WasapiAudio` пока намеренно не заявлен capability bit: WASAPI capture/playout
остаётся в изолированных managed pipelines, а PCM/Opus/RTP/NetEq проходит
через libwebrtc внутри DLL.

Реализованы и проверены:

- lifecycle libwebrtc engine и корректное завершение callback;
- Viewer-offer / Host-answer, trickle ICE и ICE restart;
- строгий versioned SDP/ICE envelope с SHA-256 fingerprint DLL;
- data channels `control`, `input-fast`, `files`;
- `files` как ordered/reliable `webrtc::Priority::kLow`;
- buffered-amount API для bounded backpressure;
- DXGI Desktop Duplication внутри DLL;
- D3D11 Video Processor BGRA texture -> NV12 texture без managed readback;
- hardware H.264 MFT encode внутри DLL;
- Annex-B H.264 через libwebrtc RTP/pacing/NACK/PLI;
- hardware H.264 MFT/DXVA decode внутри DLL;
- callback декодированной NV12 D3D11 texture в Viewer;
- latest-frame D3D11 presentation;
- один capture/encoder на ПК с раздачей до трёх Host-сеансов;
- PCM16 source/sink через штатные Opus, RTP и NetEq libwebrtc;
- RTCStats выбранной ICE-пары, RTT, bitrate, RTP, jitter, NACK/PLI/FIR;
- strict artifact contract в конечных приложениях;
- native loopback и полный managed ABI 2.4 loopback.

`NativeV2Runtime.SessionReady` становится истинным только при полном наборе
обязательных ABI 2.4 capability bits и корректном artifact. Полный
fingerprinted ABI 2.4 является основным product route по умолчанию. Для
эксплуатационного ограничения доступны:

```text
GAMEREMOTE_NATIVE_V2_DISABLE=1
GAMEREMOTE_NATIVE_V2_PCS=X5;X16
```

Первый параметр аварийно отключает NativeV2, второй необязательно ограничивает
production route точным списком ПК. При отказе probe/start/recovery Viewer
переходит на LegacyV1 как на аварийный совместимый маршрут.

## Видеопуть Host

```text
DXGI Desktop Duplication (GameRemote.NativeV2.dll)
  -> BGRA ID3D11Texture2D
  -> D3D11 Video Processor
  -> NV12 ID3D11Texture2D
  -> hardware Media Foundation H.264 MFT
  -> Annex-B access unit
  -> SharedEncodedVideoRelay owner
  -> NativeV2 external encoded source каждого PeerConnection
  -> libwebrtc RTP / pacing / congestion control / NACK / PLI
```

Managed `NativeV2HostVideoPipeline` только запускает/останавливает нативный
граф, применяет rate control, обрабатывает события и связывает его с shared
relay. Он больше не захватывает BGRA и не кодирует H.264 самостоятельно.

## Видеопуть Viewer

```text
libwebrtc RTP / jitter / depacketization
  -> полный ordered Annex-B access unit
  -> WindowsH264Decoder внутри GameRemote.NativeV2.dll
  -> NV12 ID3D11Texture2D
  -> retained NativeV2TextureLease
  -> latest-frame mailbox
  -> NativeVideoSurface / D3D11 swap chain
```

Decoder queue ограничена тремя access units. При переполнении нельзя удалить
один произвольный P-frame. DLL очищает зависимую цепочку, ждёт IDR и не чаще
одного раза за 500 мс просит key frame через RTCP PLI и control recovery.

Encoded H.264 до decoder всегда ordered. Latest-only разрешён только после
decode на presentation boundary.

## Общий capture/encoder

Каждому администратору по-прежнему принадлежит отдельный Host процесс и
PeerConnection. Это сохраняет отдельные DTLS/SRTP, token, input и file
channels. Дорогая media source общая:

1. `SharedEncodedVideoRelay.Create` использует owner lock по ПК/desktop/monitor.
2. Owner вызывает `NativeV2Engine.StartHostMedia` и публикует Annex-B.
3. Subscriber получает тот же поток по локальному named pipe.
4. Максимум участников: 3.
5. Очередь каждого subscriber: 2 кадра.
6. Медленный subscriber очищает только собственную зависимую GOP-цепочку.
7. Subscriber ждёт новый IDR и не тормозит owner/других Viewer.
8. PLI subscriber передаётся owner.
9. При смерти owner Viewer выполняет reconnect, после чего новый Host выбирает
   владельца.

Это один encode, но не один общий PeerConnection: повторное SRTP/RTP
шифрование для каждого администратора остаётся изолированным.

## Аудиопуть

```text
WASAPI loopback (managed Host)
  -> PCM16 48 kHz, 10 ms
  -> NativeV2 PcmAudioSource
  -> libwebrtc Opus / RTP
  -> libwebrtc NetEq
  -> NativeV2 PcmAudioSink
  -> bounded managed queue
  -> WASAPI event playout (managed Viewer)
```

Аудио имеет собственную очередь и не ждёт video/file pipeline.

## Data channels и приоритеты

- `control`: ordered/reliable, команды, auth, cursor, clipboard, recovery;
- `input-fast`: realtime/latest-only для мыши и геймпада;
- `files`: ordered/reliable, `webrtc::Priority::kLow`.

Файловый канал не может бесконтрольно накапливать данные:

- Viewer и Host читают native buffered amount;
- Host download держит не более 512 КБ ожидающих file bytes;
- upload сначала пишет `.part`;
- после завершения проверяются размер и SHA-256;
- только затем выполняется атомарный переход в итоговый путь;
- отмена и ошибка очищают временный файл.

## Signaling и безопасность

- Viewer создаёт offer, Host создаёт answer; обратные роли отклоняются ABI;
- BotStats broker переносит envelope и не разбирает media payload;
- envelope содержит SDP, candidates, gathering state и fingerprint DLL;
- пустой список candidates допустим только до gathering complete;
- late candidates передаются post-SDP trickle;
- signal request содержит session, viewer, token, generation, sequence и
  candidate cursors;
- replay разрешён только для идентичного SHA-256 payload;
- несовпадение lease/token/viewer/fingerprint отклоняется до media start;
- ICE restart выполняется в существующем PeerConnection;
- BotStats никогда не проксирует video/audio/input/files.

## Desktop и lifecycle

`6D6X6.Service/RemoteDesktopHostManager.cs` владеет Host-процессами и хранит
авторитетный desktop в:

```text
%ProgramData%\6D6X6\State\GameRemote.desktop
```

- без интерактивного пользователя выбирается `winsta0\winlogon`;
- после logon/unlock выбирается `winsta0\default`;
- lock/logoff переключает состояние на `winlogon`;
- события другой Windows session не останавливают текущие Host;
- при смене видимого desktop останавливаются только затронутые Host;
- Viewer сохраняет окно и автоматически согласует новый Host;
- `HostRuntime.Generation` не позволяет старому loop удалить заменившийся Host;
- после reboot служба, user-agent `HELLO` и Viewer reconnect восстанавливают
  маршрут без блокировки или выхода пользователя.

## Владение памятью и очереди

- Native callback не блокирует libwebrtc thread.
- Byte payload действителен только во время callback; managed adapter копирует
  его немедленно.
- Encoded H.264 не coalesce до decoder.
- Decoded textures coalesce latest-only перед present.
- D3D11 pointer в callback borrowed; удержание требует `AddRef` и lease.
- Вытесненный texture lease освобождается немедленно.
- Audio, control, input, files и video имеют независимые очереди.
- После `engine_destroy` новые callback запрещены.

## Первопричина прежней потери FPS

Старый `EventDispatcher` заменял ещё не прочитанные encoded H.264 access units
последним кадром. Это разрушало dependency chain до decoder и создавало
видимость исправного capture/encoder при низком presented FPS.

Текущее правило:

- ordered/lossless до decoder;
- bounded dependency-aware reset только всей GOP-цепочки;
- latest-only только после decode;
- IDR запрашивается при PLI/FIR, decoder loss или полном reset, но не при
  каждом локальном замедлении UI.

## Artifact

```text
Staging: D:\GameRemoteNativeV2\staged\GameRemote.NativeV2
ABI: 2.4
WebRTC revision: e12c39e03c3dcab594f73a1e524b1f2c17dfdcb8
DLL size: 8713216
DLL SHA-256: 05913a99ebbdff449be95aa0e7165978f9a9eb09bcf912013814e9a3e7970207
Manifest schema: 2
```

Обязательный набор:

- `GameRemote.NativeV2.dll`;
- `GameRemote.NativeV2.manifest.json`;
- `WEBRTC_REVISION.txt`;
- `LICENSE.libwebrtc.txt`;
- `THIRD_PARTY_NOTICES.libwebrtc.md`.

MSBuild-контракт копирует эти файлы с `CopyToOutputDirectory=Always` и
`CopyToPublishDirectory=Always`. После копирования SHA-256 выходной DLL
обязательно сравнивается с выбранным staging artifact. Это защищает от
ситуации, когда более новый timestamp старого ABI заставляет
`PreserveNewest` оставить несовместимый бинарник в `bin` или publish.

## Где продолжать работу

Не искать NativeV2 media bugs в legacy SIPSorcery/WebView2 классах.

- C ABI: `native/include/gameremote_native_v2.h`;
- PeerConnection/RTP/data/stats: `native/src/gameremote_native_v2_webrtc.cpp`;
- DXGI/D3D11/MF: `native/src/gameremote_native_v2_windows_media.cpp`;
- managed ABI: `NativeV2Runtime.cs`, `NativeV2Engine.cs`;
- Host adapter: `../GameRemote.Host/Native/NativeV2HostVideoPipeline.cs`;
- shared source: `../GameRemote.Host/Session/SharedEncodedVideoRelay.cs`;
- Host session: `../GameRemote.Host/Session/NativeV2HostSessionController.cs`;
- Viewer session: `../GameRemote.Viewer/Native/NativeV2ProductViewerSession.cs`;
- Viewer present: `../GameRemote.Viewer/Native/NativeV2ViewerVideoPipeline.cs`;
- полный runtime call graph: `../GAMEREMOTE_NATIVE_V2_HANDOFF_RU.md`.

## Остаток до общего rollout

Это не новый архитектурный этап, а полевая валидация:

1. Windows 10/11 и NVENC/QSV/AMF.
2. LAN и P1/P2/P3.
3. Два-три одновременных Viewer, включая медленного.
4. Kill owner Host и автоматическое переизбрание через reconnect.
5. Lock/unlock/logon/logoff/winlogon/reboot.
6. Игра, звук, ввод и большой file transfer одновременно.
7. Длительный soak по RAM, handles, queues, loss, PLI и presented FPS.

NativeV2 уже является default. Legacy остаётся аварийным fallback на время
полевой матрицы и может быть принудительно выбран через
`GAMEREMOTE_NATIVE_V2_DISABLE=1`.
