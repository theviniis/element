# Close Button Minimizes to System Tray

## Context

Element already ships a full system tray: [systemtray.cpp](src/ui/systemtray.cpp) (`SystemTray`, JUCE `SystemTrayIconComponent`), the hide/show command `Commands::toggleUserInterface` ([guiservice.cpp](src/services/guiservice.cpp) case at L1020), and two persisted toggles (`systrayKey`, `startHiddenKey`) with Preferences rows.

The missing piece: the main window's **close** button always quits. `MainWindow::closeButtonPressed()` ([mainwindow.cpp:120](src/ui/mainwindow.cpp)) calls `systemRequestedQuit()` unconditionally, while `minimiseButtonPressed()` already hides to the tray when the tray is enabled.

Decisions made with the user:
- New preference, **default off**: "Close button minimizes to tray", only enabled when "Show system tray" is on (same gating as "Start hidden in system tray").
- Close **silently hides**; quitting stays available via tray menu → Exit and the normal app menu/Cmd+Q.
- Decision logic lives in a `GuiService` helper (so `MainWindow` doesn't have to include `src/ui/systemtray.hpp`, which pulls in the legacy `ElementApp.h`).
- No new unit tests: the logic is a short conjunction over UI state that can't be exercised headless. Verification is build + manual runs.

JUCE note: `DocumentWindow::userTriedToCloseWindow()` forwards to `closeButtonPressed()` (juce_DocumentWindow.cpp:433), so one override covers both the native title-bar close and OS-level close requests.

## Implementation

### 1. Settings key
[settings.hpp](include/element/settings.hpp) / [settings.cpp](src/settings.cpp), next to the tray keys:
- `static const char* closeToTrayKey = "closeToTray";`
- `bool isCloseToTrayEnabled() const` → `getBool (closeToTrayKey, false)`
- `void setCloseToTrayEnabled (bool)` → `set (closeToTrayKey, enabled)`
- Doxygen comment mirroring `isStartHiddenEnabled()`: only meaningful when the system tray is enabled and available.

### 2. Decision helper
[include/element/ui.hpp](include/element/ui.hpp) + [guiservice.cpp](src/services/guiservice.cpp), beside `refreshSystemTray()`:

```cpp
/** True if closing the main window should hide it to the system tray
    instead of quitting the application. */
bool shouldHideMainWindowOnClose() const;
```

Returns true only when **all** hold:
- `settings.isCloseToTrayEnabled()`
- `settings.isSystrayEnabled()`
- `SystemTray::isAvailable()` (standalone run mode, not Wine)
- `SystemTray::getInstance() != nullptr && isOnDesktop()` (icon was actually created)

### 3. Close button
[mainwindow.cpp:120](src/ui/mainwindow.cpp):

```cpp
void MainWindow::closeButtonPressed()
{
    auto& gui = *world.services().find<GuiService>();
    if (gui.shouldHideMainWindowOnClose())
    {
        gui.commands().invokeDirectly (Commands::toggleUserInterface, true);
        return;
    }
    JUCEApplication::getInstance()->systemRequestedQuit();
}
```

Quit paths are untouched: menu Quit, tray Exit and Cmd+Q go through `systemRequestedQuit()` directly ([guiservice.cpp:966](src/services/guiservice.cpp), [winsparkle.cc](src/winsparkle.cc)).

### 4. Preferences row
[preferences.cpp](src/ui/preferences.cpp) `GeneralSettingsPage`:
- New `BoolSettingRow closeToTray;` next to `startHidden` (L587-588).
- `init (*this, "Close button minimizes to tray", settings.isCloseToTrayEnabled(), ...)` writing `setCloseToTrayEnabled` (L427-431).
- `closeToTray.setEnabled (systray.get())` at init, and inside the `systray` callback alongside `startHidden` (L423).
- `layoutSetting (r, closeToTray);` after `startHidden` (L526).

## Edge cases

- Preference on but no tray (plugin run mode, Wine, tray disabled) → close quits as today; the app is never hidden without a way back.
- Disabling "Show system tray" disables the new row; the stored preference is kept for when it is re-enabled.
- Known limitation (parity with the existing minimise button): JUCE cannot detect a missing tray *host* on X11, so on a desktop without a StatusNotifier/Embed host the icon may be invisible — same risk the minimise button already has, and opt-in via a default-off preference.

## Verification

- `cmake --build build && ctest --test-dir build --output-on-failure` (whole suite stays green).
- Manual: Preferences → enable "Show system tray" + "Close button minimizes to tray"; close the window → app keeps running with a tray icon; tray click restores it; tray → Exit quits; Cmd+Q quits; uncheck the preference → close quits again; uncheck "Show system tray" → close quits.
- `util/format.py` pass over touched files.
