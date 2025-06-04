# Terminal User Interface (TUI) Documentation

## 1. Overview

This document provides an overview of the Terminal User Interface (TUI) for the OpenCode application. The TUI is built using the Go programming language and relies heavily on the following libraries:

- **Bubble Tea**: A powerful framework for building terminal applications based on The Elm Architecture. It provides the core structure for managing state, events, and rendering.
- **Lipgloss**: A library for declarative, style-driven terminal layout and styling. It's used for defining the appearance of UI elements.

The TUI aims to provide an interactive and user-friendly command-line experience for interacting with the AI agent, managing sessions, viewing logs, and configuring the application.

## 2. Directory Structure

The TUI code is organized within the `internal/tui/` directory. Key subdirectories include:

- **`components/`**: Contains reusable UI building blocks, categorized further by functionality (e.g., `chat/`, `dialog/`, `logs/`, `core/`).
- **`image/`**: Utilities related to image handling for the TUI (e.g., previews in file picker).
- **`layout/`**: Defines different layout structures (e.g., `Container`, `SplitPaneLayout`, `PlaceOverlay`) used to arrange components on the screen.
- **`page/`**: Implements the different "pages" or main views of the application (e.g., `ChatPage`, `LogsPage`).
- **`styles/`**: Contains global styling helpers, icon definitions, and markdown rendering configurations.
- **`theme/`**: Manages UI themes, allowing for different color schemes and visual appearances.
- **`tui.go`**: The main entry point and orchestrator for the TUI, housing the primary application model.
- **`util/`**: General utility functions and components for the TUI.

## 3. Core Concepts

### App Model (`tui.go`)

The main application logic resides in `internal/tui/tui.go`. It defines the `appModel` struct, which is the root of the Bubble Tea application.

- **State Management**: `appModel` holds the overall state of the TUI, including the current page, active dialogs, selected session, and references to various UI components.
- **Event Loop**: The `Update` method in `appModel` is the central event handler. It processes messages such as user input (key presses), window resize events, and custom application events.
- **Page Management**: It manages transitions between different pages (e.g., from Chat to Logs) using a `PageChangeMsg`.
- **Dialog Orchestration**: It controls the visibility and interaction with various dialog components (e.g., Help, Quit, Command Palette).
- **Global Keybindings**: Defines and handles global keyboard shortcuts.

### Pages (`internal/tui/page/`)

Pages represent the primary views of the application. Each page is typically a Bubble Tea model itself.

- **`ChatPage` (`chat.go`)**: The main interface for users to interact with the AI agent. It includes areas for displaying the conversation, typing messages, and a sidebar for session-related information.
- **`LogsPage` (`logs.go`)**: Displays application logs, allowing users to view log entries and their details.
- **`page.go`**: Defines common types like `PageID` and `PageChangeMsg` for page management.

### Layouts (`internal/tui/layout/`)

Layout components are responsible for arranging other UI elements on the screen.

- **`Container` (`container.go`)**: A wrapper around other components that allows for adding padding and borders. It helps in styling and isolating components.
- **`SplitPaneLayout` (`split.go`)**: Divides the screen into multiple resizable panels (left/right, top/bottom). Used, for example, by the `ChatPage` to arrange the message list, editor, and sidebar.
- **`PlaceOverlay` (`overlay.go`)**: A function to render one UI element (foreground) on top of another (background). This is essential for displaying dialogs and pop-ups without disrupting the underlying page content.
- **`layout.go`**: Defines common interfaces like `Sizeable` (for components that can be resized) and `Bindings` (for components that expose keybindings).

### Components (`internal/tui/components/`)

Components are reusable UI building blocks that encapsulate specific functionalities and appearances.

- **`chat/`**: Components specific to the chat interface:
    - `editor.go`: The message input field.
    - `list.go`: The scrollable list of chat messages.
    - `message.go`: Logic for rendering individual chat messages (user, assistant, tool calls).
    - `sidebar.go`: The sidebar displaying session info and modified files.
- **`core/`**: Core UI elements:
    - `status.go`: The status bar at the bottom of the screen, showing help hints, model info, LSP diagnostics, and temporary messages.
- **`dialog/`**: Various dialog components for specific interactions:
    - `help.go`: Displays keyboard shortcuts.
    - `quit.go`: Confirmation dialog for quitting the application.
    - `session.go`: Allows switching between chat sessions.
    - `commands.go`: A command palette for executing predefined actions.
    - `theme.go`: Allows changing the UI theme.
    - `models.go`: Allows selecting the LLM model.
    - `filepicker.go`: A dialog for browsing and selecting files (e.g., for attachments), including image previews.
    - `permission.go`: Asks for user permission for potentially sensitive actions.
    - `init.go`: Dialog for initial project setup/configuration.
    - `arguments.go`: Dialog for inputting arguments for custom commands.
    - `complete.go`: Dialog for path/command completions.
- **`logs/`**: Components for the logs page:
    - `table.go`: Displays logs in a tabular format.
    - `details.go`: Shows the detailed content of a selected log entry.
- **`util/`**: General utility components:
    - `simple-list.go`: A generic, reusable list component.

## 4. Styling and Theming

The TUI's appearance is highly customizable through styles and themes.

### Styles (`internal/tui/styles/`)

- **`styles.go`**: Provides helper functions (e.g., `BaseStyle()`, `Padded()`, `Border()`) that create `lipgloss.Style` objects using colors from the currently active theme. This promotes consistency in styling across the application.
- **`icons.go`**: Defines string constants for various icons used in the UI.
- **`markdown.go`**: Configures the `glamour` library for rendering markdown content. It defines an `ansi.StyleConfig` that maps markdown elements (headings, code blocks, links, etc.) to theme colors, ensuring that markdown is styled consistently with the rest of the UI. It also handles syntax highlighting within code blocks using theme colors.
- **`background.go`**: Includes `ForceReplaceBackgroundWithLipgloss` to ensure consistent background theming, especially when dealing with components that might try to set their own ANSI background colors.

### Themes (`internal/tui/theme/`)

- **`theme.go`**:
    - Defines the `Theme` interface, which lists all color methods a theme must implement (e.g., `Primary()`, `Text()`, `Background()`, `MarkdownHeading()`). All colors are `lipgloss.AdaptiveColor` to support both light and dark terminal backgrounds.
    - Provides a `BaseTheme` struct that concrete themes can embed to get a default implementation.
- **`manager.go`**:
    - Implements a `ThemeManager` (`globalManager`) to handle theme registration, selection, and retrieval.
    - `RegisterTheme()`: Adds a new theme (e.g., Catppuccin, Dracula, defined in their respective files like `catppuccin.go`) to the manager.
    - `SetTheme()`: Changes the active theme and updates the application's configuration file.
    - `CurrentTheme()`: Returns the currently active theme instance, which components use to get their colors.
    - `AvailableThemes()`: Provides a list of all registered theme names for selection in the `ThemeDialog`.
- **Theme Files (e.g., `catppuccin.go`, `dracula.go`, `opencode.go`)**: Each file defines a specific theme by creating a struct that embeds `BaseTheme` and sets its color fields to the specific hex values for that theme. These themes are then registered with the `ThemeManager`.

## 5. Key UI Elements and Interactions

### Chat Interface

The primary interaction point is the chat interface, composed of:

- **Message List (`messagesCmp`)**: Displays the conversation history with the AI, including user messages, AI responses, and tool calls/results. It supports scrolling and dynamically updates as new messages arrive.
- **Message Editor (`editorCmp`)**: A text input area for users to compose their messages. It supports multi-line input, attachments, and opening in an external editor.
- **Sidebar (`sidebarCmp`)**: When a session is active, this panel shows the session title, LSP configurations, and a list of files modified during the session with diff statistics.

### Dialogs

Dialogs are used for various interactions and settings:

- **Help (`HelpCmp`)**: Activated by `Ctrl+?`, shows relevant keybindings.
- **Quit (`QuitDialog`)**: Activated by `Ctrl+C`, asks for confirmation before exiting.
- **Command Palette (`CommandDialog`)**: Activated by `Ctrl+K`, allows searching and executing various commands (e.g., "Initialize Project", "Compact Session", custom commands).
- **Session Switcher (`SessionDialog`)**: Activated by `Ctrl+S`, lists existing sessions to switch to.
- **Model Selector (`ModelDialog`)**: Activated by `Ctrl+O`, allows changing the LLM used by the agent.
- **Theme Selector (`ThemeDialog`)**: Activated by `Ctrl+T`, allows changing the TUI's visual theme.
- **File Picker (`FilepickerCmp`)**: Activated by `Ctrl+F` (or from the editor for attachments), provides a file browser with image previews.
- **Permission Dialog (`PermissionDialogCmp`)**: Shown when an agent action requires user approval.
- **Init Dialog (`InitDialogCmp`)**: Shown on first run or when the project is not initialized.
- **Arguments Dialog (`MultiArgumentsDialogCmp`)**: Activated by certain commands, allows the user to input multiple arguments required by that command.

### Status Bar (`statusCmp`)

The status bar at the bottom of the screen provides at-a-glance information:

- Help hint (`Ctrl+?`).
- Current LLM model name.
- LSP diagnostics (errors, warnings).
- Token usage and cost for the current session (if active).
- Temporary informational or error messages.

## 6. Development Guide

### How to Add a New Component

1.  **Define the Model**: Create a new Go file in the appropriate subdirectory of `internal/tui/components/`. Define a struct for your component's state.
2.  **Implement `tea.Model`**: Implement the `Init()`, `Update(tea.Msg) (tea.Model, tea.Cmd)`, and `View() string` methods.
3.  **Styling**: Use helper functions from `internal/tui/styles/` and colors from `theme.CurrentTheme()` for styling.
4.  **Integration**:
    *   If it's a general component, it might be used by pages or other components.
    *   If it's a dialog, integrate it into `internal/tui/tui.go`'s `appModel` (add state, keybindings to show/hide, and update/view logic).

### How to Add a New Page

1.  **Define the Page Model**: Create a new Go file in `internal/tui/page/`. Define a struct for the page's state and implement `tea.Model`.
2.  **Page ID**: Define a new `PageID` constant in `internal/tui/page/page.go`.
3.  **Register the Page**: In `internal/tui/tui.go`, add the new page to the `pages` map in `appModel`.
4.  **Navigation**: Implement logic to navigate to this page, usually by sending a `PageChangeMsg` from another part of the application (e.g., via a keybinding or command).
5.  **Layout**: Use layout components from `internal/tui/layout/` to structure the page's content.

### How to Add a New Theme

1.  **Create Theme File**: Add a new Go file in `internal/tui/theme/` (e.g., `mytheme.go`).
2.  **Define Theme Struct**:
    ```go
    package theme

    import "github.com/charmbracelet/lipgloss"

    type MyTheme struct {
        BaseTheme
    }

    func newMyTheme() Theme {
        return &MyTheme{
            BaseTheme: BaseTheme{
                PrimaryColor:       lipgloss.AdaptiveColor{Light: "#RRGGBB", Dark: "#RRGGBB"},
                // ... define all other colors required by the Theme interface
            },
        }
    }

    func init() {
        RegisterTheme("my-theme-name", newMyTheme())
    }
    ```
3.  **Implement Colors**: Fill in all the `lipgloss.AdaptiveColor` fields in `BaseTheme` with the hex color codes for your theme, providing both `Light` and `Dark` variants.
4.  **Register**: In the `init()` function of your theme file, call `RegisterTheme("your-theme-name", newYourTheme())`.
5.  The new theme will then be available in the `ThemeDialog`.
```
