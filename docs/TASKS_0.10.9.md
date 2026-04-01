# RustConn 0.10.9 — Задачі

> На основі: `AUDIT_0.10.9.md`, `AUDIT_0.10.9_REVIEW.md`, `HIG_REVIEW_0.10.9.md`, `SYSADMIN_REVIEW_0.10.9.md`
> Дата: 2026-03-31

---

## P0 — Блокери релізу

### ✅ T-1: Tar archive path traversal — defense-in-depth
- **Файл:** `rustconn-core/src/cli_download.rs` (`extract_tar`, `extract_tar_gz`, `extract_tar_xz`)
- **Що:** Замінити `archive.unpack(dest)` на ручну ітерацію записів з валідацією шляхів (аналогічно `enclosed_name()` для zip). Пінити `tar >= 0.4.45` в `Cargo.toml` (CVE-2026-33056).
- **Джерело:** Audit 1.1, Rust Review NEW-1
- **Статус:** Виконано в 0.10.9 (Security: "Tar archive path traversal (defense-in-depth)")

### ✅ T-2: Виправити changelog — `--device=all` vs `--device=serial`
- **Файли:** `CHANGELOG.md`, `debian/changelog`, `packaging/obs/debian.changelog`, `packaging/obs/rustconn.changes`, `packaging/obs/rustconn.spec`, `metainfo.xml`
- **Що:** Додати корекцію в 0.10.9 changelog: "Corrected: Flatpak uses `--device=all` (required for serial), not `--device=serial` as previously stated in v0.9.11 notes."
- **Джерело:** Sysadmin S-12
- **Статус:** Виконано — корекція додана в CHANGELOG.md, debian/changelog, packaging/obs/debian.changelog, packaging/obs/rustconn.changes, packaging/obs/rustconn.spec, metainfo.xml

---

## P1 — Безпека (критичні)

### ✅ T-3: Bitwarden session key — env var замість CLI arg
- **Файл:** `rustconn-core/src/secret/bitwarden.rs` (`build_command`)
- **Що:** Замінити `cmd.arg("--session").arg(key)` на `cmd.env("BW_SESSION", key)`.
- **Джерело:** Audit 1.3
- **Статус:** Виконано в 0.10.9 (Security: "Bitwarden session key no longer exposed in process list")

### ✅ T-4: 1Password password — stdin замість CLI arg
- **Файл:** `rustconn-core/src/secret/onepassword.rs` (`store`, `run_command`)
- **Що:** Передавати `password=<value>` через stdin pipe замість CLI-аргументу.
- **Джерело:** Audit 1.4
- **Статус:** Виконано в 0.10.9 (Security: "1Password credentials no longer exposed in process list")

### ✅ T-5: File permissions 0600 для KDBX export та всіх export файлів
- **Файли:** `rustconn-core/src/secret/kdbx.rs` (`export_xml`), `rustconn-core/src/export/mod.rs` (`write_export_file`)
- **Що:** Після створення файлу встановити `std::os::unix::fs::PermissionsExt::from_mode(0o600)`.
- **Джерело:** Audit 1.5, 1.6
- **Статус:** Виконано в 0.10.9 (Security: "Export file permissions hardened")

### ✅ T-6: RDP certificate — дефолт `false`, умовний `/cert:ignore`
- **Файли:** `rustconn-core/src/rdp_client/config.rs`, `rustconn-core/src/protocol/freerdp.rs`
- **Що:** Змінити `ignore_certificate: true` на `false`. Зробити `/cert:ignore` умовним. Додати `/cert:tofu` як дефолт.
- **Джерело:** Audit 1.2, HIG H3
- **Статус:** Виконано в 0.10.9 (Security: "RDP certificate validation")

### ✅ T-7: Bitwarden `lock_vault()` — додати `clear_session_key()`
- **Файл:** `rustconn-core/src/secret/bitwarden.rs`
- **Що:** В `lock_vault()` додати виклик `clear_session_key()` поруч з `clear_verified()`.
- **Джерело:** Audit 1.11
- **Статус:** Виконано в 0.10.9 (Security: "Bitwarden session key cleared on vault lock")

### ❌ T-8: vault_ops — замінити `Result<(), String>` на `SecretResult`
- **Файл:** `rustconn/src/vault_ops.rs`
- **Що:** Замінити `Result<(), String>` / `Result<Option<Credentials>, String>` на `SecretResult<()>` / `SecretResult<Option<Credentials>>` у ~10 публічних функціях. Видалити `.map_err(|e| format!("{e}"))`.
- **Джерело:** Audit 2.1, HIG H5
- **Статус:** Не виконано — vault_ops.rs досі використовує `Result<Option<String>, String>`

---

## P2 — Безпека (середні) та якість

### ✅ T-9: VNC custom args — blocklist небезпечних аргументів
- **Файл:** `rustconn-core/src/protocol/vnc.rs`
- **Що:** Додати blocklist: `-via`, `-passwd`, `-PasswordFile`, `-SecurityTypes`, `-ProxyServer`. Аналогічно до RDP blocklist.
- **Джерело:** Audit 1.7
- **Статус:** Виконано в 0.10.9 (Security: "VNC custom args blocklist")

### ✅ T-10: FreeRDP extra_args — застосувати blocklist з rdp.rs
- **Файл:** `rustconn-core/src/protocol/freerdp.rs`
- **Що:** Фільтрувати `extra_args` через той самий blocklist що `custom_args` в `rdp.rs`.
- **Джерело:** Audit 1.8
- **Статус:** Виконано в 0.10.9 (Security: "FreeRDP extra args blocklist")

### ✅ T-11: Pass backend — валідація connection_id
- **Файл:** `rustconn-core/src/secret/pass.rs`
- **Що:** Перевіряти що `connection_id` не містить `..`, `/`, `\`. Або sanitize через `Path::components()`.
- **Джерело:** Audit 1.9
- **Статус:** Виконано в 0.10.9 (Security: "Pass backend path traversal prevention")

### ✅ T-12: Log sanitization — розширити патерни
- **Файл:** `rustconn-core/src/session/logger.rs`
- **Що:** Додати: `passphrase:`, `client_secret:`, `authorization:`, `ghp_[a-zA-Z0-9]+`, `glpat-`, `eyJ[a-zA-Z0-9]` (JWT).
- **Джерело:** Audit 1.12, Sysadmin review
- **Статус:** Виконано в 0.10.9 (Security: "Log sanitization expanded")

### ✅ T-13: Asbru regex — LazyLock
- **Файл:** `rustconn-core/src/import/asbru.rs`
- **Що:** Замінити `Regex::new()` на `static ASBRU_GV_REGEX: LazyLock<Regex>`.
- **Джерело:** Audit 2.5
- **Статус:** Виконано в 0.10.9 (Improved: "Asbru import regex cached")

### ❌ T-14: RDP certificate UI — AdwPreferencesGroup
- **Файли:** `rustconn/src/dialogs/connection/rdp.rs` (або відповідний)
- **Що:** Додати "Security" group в RDP connection dialog: verify certificate toggle, CA cert path. Аналогічно до SPICE `create_security_group()`.
- **Джерело:** HIG H3
- **Статус:** Не виконано — Security group є тільки в SPICE діалозі, в RDP відсутній

### ❌ T-15: Bulk delete — міграція на AdwAlertDialog
- **Файл:** `rustconn/src/window/operations.rs` (`create_bulk_delete_dialog`)
- **Що:** Замінити `adw::Window` на `AdwAlertDialog` з extra child widget для списку. Додати `set_close_response("cancel")`, `ResponseAppearance::Destructive`.
- **Джерело:** HIG H6
- **Статус:** Не виконано — `create_bulk_delete_dialog` досі використовує `adw::Window`

### ❌ T-16: Error toasts → AdwAlertDialog для actionable failures
- **Файли:** `rustconn/src/window/operations.rs`, `connection_dialogs.rs`
- **Що:** Замінити `ToastType::Error` на `AdwAlertDialog` для помилок що потребують дії користувача. Залишити toasts для інформаційних повідомлень.
- **Джерело:** HIG H2
- **Статус:** Не виконано

### ❌ T-17: Тести для vault_ops
- **Файл:** `rustconn/src/vault_ops.rs` або `rustconn-core/tests/`
- **Що:** Додати unit-тести для `dispatch_vault_op`, `generate_store_key`, `select_backend_for_load`, `rename_vault_credential` з mock backends.
- **Джерело:** Audit 2.6
- **Статус:** Не виконано — тести для vault_ops не знайдено

### ❌ T-18: Flatpak `home/.aws` → read-only
- **Файли:** `packaging/flatpak/*.yml`, `packaging/flathub/*.yml`
- **Що:** Змінити `home/.aws` на `home/.aws:ro`. Для SSO token cache використати sandbox copy.
- **Джерело:** Sysadmin S-2
- **Статус:** Не виконано — `home/.aws:ro` не знайдено в маніфестах

### ✅ T-19: Backup archive — документувати `.machine-key`
- **Файл:** `docs/USER_GUIDE.md` або in-app help
- **Що:** Документувати що `.machine-key` не включається в backup і потрібен для розшифровки credentials на іншій машині.
- **Джерело:** Sysadmin S-6
- **Статус:** Виконано — додано попередження в секцію Backup & Restore в USER_GUIDE.md

### ✅ T-20: Download checksum — попередження при SkipLatest
- **Файл:** `rustconn-core/src/cli_download.rs`
- **Що:** Логувати `tracing::warn!` при `ChecksumPolicy::SkipLatest`. Розглянути signature verification.
- **Джерело:** Audit 1.10, Sysadmin review
- **Статус:** Виконано — `tracing::warn!` присутній у всіх 3 match arms для `SkipLatest`

---

## P3 — Оптимізації та технічний борг

### ❌ T-21: `poll_for_result` → `glib::MainContext::channel()`
- **Файл:** `rustconn/src/utils.rs`
- **Що:** Замінити 16ms polling timer на event-driven `glib::MainContext::channel()`.
- **Джерело:** Audit 3.5
- **Статус:** Не виконано — `glib::MainContext::channel()` не використовується в кодовій базі

### ✅ T-22: Wayland subsurface — cfg-gate мертвий код
- **Файл:** `rustconn/src/wayland_surface.rs`
- **Що:** Винести `ShmBuffer` та subsurface код за `#[cfg(feature = "wayland-native")]`. Додати feature в `Cargo.toml`.
- **Джерело:** Audit 4.1
- **Статус:** Виконано — `#[cfg(feature = "wayland-native")]` присутній в wayland_surface.rs та display.rs

### ✅ T-23: Консолідація PixelBuffer
- **Файли:** `embedded_rdp/buffer.rs`, `embedded_spice.rs`, `embedded_vnc_types.rs`, `wayland_surface.rs`
- **Що:** Створити єдиний `PixelBuffer` у спільному модулі.
- **Джерело:** Audit 4.2
- **Статус:** Частково виконано в 0.10.8 — `CairoBackedBuffer` винесено в `cairo_buffer.rs`, використовується RDP, VNC, SPICE. Старі fallback типи (`PixelBuffer`, `SpicePixelBuffer`, `VncPixelBuffer`) залишились — див. T-31.

### ❌ T-31: Видалити застарілі fallback PixelBuffer типи
- **Файли:** `rustconn/src/embedded_rdp/buffer.rs` (`PixelBuffer`), `rustconn/src/embedded_spice.rs` (`SpicePixelBuffer`), `rustconn/src/embedded_vnc_types.rs` (`VncPixelBuffer`), `rustconn/src/embedded_rdp/drawing.rs`, `rustconn/src/embedded_vnc.rs`
- **Що:** Видалити старі per-protocol pixel buffer структури та їх fallback rendering paths (`to_vec()` copy). Всі embedded viewers мають використовувати виключно `CairoBackedBuffer` з `cairo_buffer.rs`. Прибрати `should_render_fallback` логіку в drawing.rs, embedded_spice.rs, embedded_vnc.rs.
- **Джерело:** Продовження T-23 (Audit 4.2)
- **Статус:** Не виконано

### ❌ T-24: RecordingReader — streaming
- **Файл:** `rustconn-core/src/session/recording.rs`
- **Що:** Замінити `fs::read()` на `BufReader` зі streaming по chunks.
- **Джерело:** Audit 3.4
- **Статус:** Не виконано — `RecordingReader::open()` досі використовує `fs::read(data_path)`

### ✅ T-25: Framebuffer fallback — guard та warning
- **Файли:** `rustconn/src/embedded_rdp/drawing.rs`, `embedded_spice.rs`
- **Що:** Додати `tracing::warn!` якщо fallback `to_vec()` path активується в embedded mode.
- **Джерело:** Audit 3.2
- **Статус:** Виконано — `tracing::warn!` (через `Once`) додано в RDP, SPICE, VNC fallback paths

### ✅ T-26: Clippy suppressions — scoped до rustconn crate
- **Файли:** `Cargo.toml`, `rustconn/Cargo.toml`
- **Що:** Перенести GTK-специфічні suppressions (`redundant_clone`, `needless_borrow`) з workspace в `rustconn/Cargo.toml`. Залишити `rustconn-core` під строгішим лінтингом.
- **Джерело:** Rust Review NEW-3
- **Статус:** Виконано — 8 GTK-специфічних suppressions перенесено в rustconn/Cargo.toml, workspace Cargo.toml залишає rustconn-core під строгішим лінтингом

### ✅ T-27: SSH agent socket — документувати Flatpak обмеження
- **Файл:** `docs/USER_GUIDE.md` або `docs/INSTALL.md`
- **Що:** Документувати що в Flatpak: custom socket paths обмежені, 1Password agent socket не змонтований, sandbox-internal agent не доступний host процесам.
- **Джерело:** Sysadmin S-4
- **Статус:** Виконано — додано детальний опис обмежень в секцію Flatpak troubleshooting в USER_GUIDE.md

### ✅ T-28: `show_error_dialog` — orphaned window fix
- **Файл:** `rustconn/src/app.rs`
- **Що:** Замінити створення тимчасового `adw::ApplicationWindow` на `dialog.present(None::<&gtk4::Widget>)`.
- **Джерело:** HIG H1
- **Статус:** Виконано — використовує `app.active_window()` замість тимчасового вікна

### ✅ T-29: Untranslated strings в snippet.rs
- **Файл:** `rustconn/src/dialogs/snippet.rs`
- **Що:** Обгорнути "Snippet name is required" та "Command is required" в `i18n()`.
- **Джерело:** HIG H7
- **Статус:** Виконано — всі 4 входження обгорнуті в `i18n()`

### ❌ T-30: Export security warning dialog
- **Файл:** `rustconn/src/dialogs/export.rs`
- **Що:** Показувати `AdwAlertDialog` з попередженням при експорті форматів що можуть містити credentials.
- **Джерело:** HIG H5 enhanced
- **Статус:** Не виконано — діалог попередження відсутній в export.rs
