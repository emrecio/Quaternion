# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Quaternion is a cross-platform desktop IM client for the [Matrix](https://matrix.org) protocol, built with Qt 6 (C++23) and QML. It is part of the Quotient project. The main branch is `dev`.

## Build Commands

```bash
# Clone with libQuotient submodule
git clone --recursive https://github.com/quotient-im/Quaternion.git

# If already cloned, initialize the submodule
git submodule init && git submodule update

# Configure and build (out-of-source)
mkdir build_dir && cd build_dir
cmake .. -DCMAKE_BUILD_TYPE=Debug
cmake --build . --target all

# macOS with Homebrew dependencies
cmake .. -DCMAKE_PREFIX_PATH="$(brew --prefix qt);$(brew --prefix qtkeychain);$(brew --prefix libolm);$(brew --prefix openssl)"
cmake --build . --target all

# macOS DMG packaging
cmake --build . --target image
```

There are no unit tests in this repository. CI validates by running `quaternion --version` with `-platform offscreen`.

## Dependencies

- **Qt 6.4+**: Widgets, Quick, Qml, Gui, Network, QuickControls2, QuickWidgets, Multimedia, LinguistTools
- **libQuotient 0.9.2+**: Matrix protocol library, used as a git submodule under `lib/` (default) or as an external installation. Controlled by `-DUSE_INTREE_LIBQMC=ON|OFF`
- **Qt Keychain**: Secure token storage
- **libolm 3.2.5+**: E2EE support
- **OpenSSL 3.x**
- **CMake 3.16+**, C++23-capable compiler (GCC 13+, Clang 16+, Apple Clang 15+, MSVC 2022)

## Architecture

The application is a single-executable Qt Widgets app with a QML-based timeline:

### Entry Point and Main Window
- `client/main.cpp` — Application bootstrap: sets up Qt, loads translations, configures proxy/encryption, creates `MainWindow`
- `client/mainwindow.{h,cpp}` — Central `QMainWindow` subclass that owns all top-level UI components and manages Matrix connections via `Quotient::AccountRegistry`. Implements `Quotient::UriResolverBase` for handling matrix: URIs

### Chat UI (Widgets + QML hybrid)
- `client/chatroomwidget.{h,cpp}` — Composite widget combining the timeline, message input (`ChatEdit`), and attachment handling. Processes slash commands (`/join`, `/me`, `/md`, etc.) and dispatches messages via `QuaternionRoom::sendMessage()`
- `client/timelinewidget.{h,cpp}` — `QQuickWidget` that hosts the QML timeline. Bridges C++ models to QML, handles read receipts and viewport tracking
- `client/qml/Timeline.qml` — Main QML timeline view using a `ListView` with the `MessageEventModel`
- `client/qml/TimelineItem.qml` — Individual message rendering (text, images, files, reactions)

### Room Model Layer
- `client/quaternionroom.{h,cpp}` — Extends `Quotient::Room` with Quaternion-specific features: event highlighting, viewport persistence, history loading (`ensureHistory()`), message sending with HTML filtering
- `client/models/messageeventmodel.{h,cpp}` — Qt model exposing room timeline events to the QML timeline
- `client/models/roomlistmodel.{h,cpp}` — Qt model for the room list sidebar
- `client/models/orderbytag.{h,cpp}` and `abstractroomordering.{h,cpp}` — Room list ordering/grouping by Matrix tags

### HTML Processing
- `client/htmlfilter.{h,cpp}` — Bidirectional conversion between Matrix-flavoured HTML and Qt-compatible HTML. Three entry points: `toMatrixHtml()` (outgoing), `fromMatrixHtml()` (incoming), `fromLocalHtml()` (paste/drag-drop)

### Dock Panels
- `client/roomlistdock.{h,cpp}` — Room list sidebar with filtering and tag-based grouping
- `client/userlistdock.{h,cpp}` — Member list sidebar for the current room

### Message Input
- `client/chatedit.{h,cpp}` — Rich text editor with tab-completion for usernames/rooms/commands
- `client/kchatedit.{h,cpp}` — Base text edit widget with message history (up/down arrow recall)

### Dialogs
- `client/logindialog.{h,cpp}` — Login/SSO authentication
- `client/roomdialogs.{h,cpp}` — Room creation and settings dialogs
- `client/profiledialog.{h,cpp}` — User profile editing
- `client/verificationdialog.{h,cpp}` — E2EE device verification

## Code Style

- Formatting: `.clang-format` in repo root (ClangFormat 18+, based on WebKit style, 100-char column limit, 4-space indent)
- C++ standard: C++23 (`cxx_std_23`)
- Qt string literals: use `u"..."_s` (from `Qt::StringLiterals`), not `QStringLiteral()`
- Namespaced includes for libQuotient: `#include <Quotient/room.h>` (enforced by `QUOTIENT_FORCE_NAMESPACED_INCLUDES`)
- Logging: use Qt logging categories prefixed with `quaternion.` (see `client/logging_categories.h`). libQuotient uses `quotient.` prefix
- License headers: SPDX format (`SPDX-FileCopyrightText` / `SPDX-License-Identifier: GPL-3.0-or-later`)
- Translations: managed externally via [Lokalise.co](https://lokalise.co); `quaternion_en.ts` is updated by building the `trbase` target
