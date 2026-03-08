# Audit Tasks

Результат аудиту кодової бази RustConn (березень 2026).

Загальний стан: кодова база в відмінному стані. Crate boundaries дотримані,
`unsafe_code = "forbid"`, credentials через `SecretString`, i18n через `i18n()`,
zero `todo!()`/`unimplemented!()`, zero `println!` в core. Критичних проблем не знайдено.

---

## TASK-01 — Zeroize Bitwarden master password в `unlock_vault`

**Пріоритет:** Low (defense-in-depth)
**Зусилля:** ~5 хвилин, 1 рядок

**Файл:** `rustconn-core/src/secret/bitwarden.rs`, функція `unlock_vault` (рядок 747)

**Проблема:** `password.expose_secret().to_string()` створює plain `String` на heap,
яка не зануляється при drop. При core dump пароль потрапить у файл.

**Рішення:**
```rust
use zeroize::Zeroizing;

pub async fn unlock_vault(password: &SecretString) -> SecretResult<SecretString> {
    let pw = Zeroizing::new(password.expose_secret().to_string());
    tokio::task::spawn_blocking(move || unlock_vault_sync(&pw))
        .await
        .map_err(|e| SecretError::ConnectionFailed(format!("Unlock task panicked: {e}")))?
}
```

`zeroize` вже є в workspace dependencies. `Zeroizing<String>` deref до `&str`.

---

## TASK-02 — Cleanup SSH_ASKPASS скрипта при drop

**Пріоритет:** Low (гарна практика)
**Зусилля:** ~15 хвилин

**Файл:** `rustconn-core/src/monitoring/ssh_exec.rs`

**Проблема:** Тимчасовий askpass скрипт створюється в `/tmp` і ніколи не видаляється.
Скрипт не містить пароль (тільки `echo "$_RC_MON_PW"`), але накопичується.

**Рішення:** Обгорнути шлях у newtype з `Drop`:
```rust
struct AskpassScript(std::path::PathBuf);

impl Drop for AskpassScript {
    fn drop(&mut self) {
        if let Err(e) = std::fs::remove_file(&self.0) {
            tracing::debug!(path = %self.0.display(), error = %e,
                "Failed to clean up askpass script");
        }
    }
}
```
Використати `Option<Arc<AskpassScript>>` замість `Option<PathBuf>` в closure captures.

---

## TASK-03 — Увімкнути `spice-embedded` за замовчуванням

**Пріоритет:** Medium (розбіжність між кодом і конфігурацією)
**Зусилля:** ~10 хвилин

**Проблема:** SPICE embedded клієнт повністю реалізований (`SpiceClient` з event loop,
input forwarding, clipboard, fallback на `remote-viewer`), але feature вимкнений
за замовчуванням в обох крейтах.

**Файли:**

1. `rustconn-core/Cargo.toml` — додати `"spice-embedded"` в default:
```toml
default = ["vnc-embedded", "rdp-embedded", "spice-embedded"]
```

2. `rustconn/Cargo.toml` — додати `"spice-embedded"` в default:
```toml
default = ["tray", "vnc-embedded", "rdp-embedded", "rdp-audio", "wayland-native", "spice-embedded"]
```

3. `product.md` — оновити таблицю Protocol Strategy:
```
| SPICE | spice-client | `remote-viewer` | `spice-embedded` |
```

Після зміни: `cargo build --all-targets` та `cargo clippy --all-targets`
для перевірки що все компілюється з новим default.

---

## Відхилені рекомендації

| Рекомендація | Причина відхилення |
|---|---|
| Async migration (`block_on_async` → `spawn_async`) | Великий рефакторинг (~30 call sites), ризик регресій. Блокування відбувається при старті (1 раз) або при підключенні (очікувано). Не виправдовує ризик. |
| fd-based password passing для SSH monitoring | Складна реалізація. `/proc/pid/environ` доступний тільки same UID — якщо зловмисник має same UID, він вже може ptrace. |
| `--device=all` документація | Коментар вже є в маніфесті: "serial ports for picocom require --device=all; no granular option exists". |
| Credential cache periodic cleanup | Expired entries видаляються при доступі (lazy). Memory overhead нерелевантний для desktop додатку. |
| Видалення X11 fallback з Flatpak | Потрібен для LTS систем (Ubuntu 22.04 до 2027). Стандартна практика для GTK4 Flatpak. |
