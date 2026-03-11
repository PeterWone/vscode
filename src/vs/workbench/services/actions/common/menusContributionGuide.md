# Menu Contributions Guide

This document explains how to contribute menu items, how those items bind to commands, and how to control placement, visibility, and enabled state.

## 1. Contribution Model

Menu UI is assembled from contributions. The key contribution points are:

- `contributes.commands`: declares command metadata (id, title, optional category, optional enablement).
- `contributes.menus`: places commands into menu locations.
- `contributes.submenus`: declares custom submenu definitions for locations that support submenu items.

At runtime, selecting a menu item executes the command with the same id.

## 2. End-to-End Minimal Example

```json
{
  "contributes": {
    "commands": [
      {
        "command": "sample.print",
        "title": "Print..."
      },
      {
        "command": "sample.printPreview",
        "title": "Print Preview..."
      }
    ],
    "menus": {
      "menuBar/file": [
        { "command": "sample.print", "group": "4z_print@1" },
        { "command": "sample.printPreview", "group": "4z_print@2" }
      ]
    }
  }
}
```

```ts
vscode.commands.registerCommand('sample.print', () => {
  // handle Print...
});

vscode.commands.registerCommand('sample.printPreview', () => {
  // handle Print Preview...
});
```

## 3. Supported Top-Level Menubar Locations

The following top-level menubar contribution keys are supported:

- `menuBar/file`
- `menuBar/edit`
- `menuBar/selection`
- `menuBar/view`
- `menuBar/go`
- `menuBar/terminal`
- `menuBar/help`

Additional menu locations and submenus continue to work as normal.

## 4. Placement, Sections, and Ordering

Use `group` to control where a menu item appears.

Format:

- `"group": "sectionName@order"`

Behavior:

- Different `sectionName` values create separate sections.
- Sections are sorted lexicographically by section name.
- Items within a section are sorted by numeric `@order` when provided.
- If `@order` is omitted, relative order in that section is less explicit.

Example section strategy for File:

- `4_save@1`
- `4_save@2`
- `4z_print@1` (after save section, before `5_*` sections)

If `group` is omitted, the item is still contributed and appended without explicit section/order metadata.

## 5. Visibility vs Disabled State

These are controlled by different fields:

- `menus.when`: controls visibility. If false, the menu item is hidden.
- `commands.enablement`: controls enabled/disabled state. If false, the item remains visible but is greyed out.

Example:

```json
{
  "contributes": {
    "commands": [
      {
        "command": "sample.print",
        "title": "Print...",
        "enablement": "resourceScheme == file"
      }
    ],
    "menus": {
      "menuBar/file": [
        {
          "command": "sample.print",
          "group": "4z_print@1",
          "when": "workbenchState != empty"
        }
      ]
    }
  }
}
```

Behavior of this example:

- `workbenchState == empty`: item hidden.
- Visible but `resourceScheme != file`: item disabled.
- Both conditions satisfied: item visible and enabled.

## 6. Submenus

For locations that support submenus:

1. Declare submenu metadata in `contributes.submenus`.
2. Reference that submenu from `contributes.menus` using a submenu item.

If a location does not support submenus, submenu contributions to that location are rejected.

## 7. Dynamic Contributions and Built-in Items

Contributed items are merged with built-in menu items for that location.

Guidelines:

- Pick stable group prefixes to land in the intended section.
- Use explicit `@order` to avoid fragile ordering.
- Avoid relying on implicit ordering when coexisting with many built-ins.

## 8. Common Patterns

### 8.1 Add one item to a top-level menu

```json
{
  "contributes": {
    "commands": [{ "command": "sample.viewAction", "title": "Sample View Item" }],
    "menus": {
      "menuBar/view": [{ "command": "sample.viewAction", "group": "9_sample@1" }]
    }
  }
}
```

### 8.2 Add items as a dedicated section

```json
{
  "menus": {
    "menuBar/file": [
      { "command": "sample.print", "group": "4z_print@1" },
      { "command": "sample.printPreview", "group": "4z_print@2" }
    ]
  }
}
```

### 8.3 Keep item visible but disable conditionally

```json
{
  "contributes": {
    "commands": [
      {
        "command": "sample.requiresFile",
        "title": "Do File Action",
        "enablement": "resourceScheme == file"
      }
    ],
    "menus": {
      "menuBar/file": [{ "command": "sample.requiresFile", "group": "9_sample@1" }]
    }
  }
}
```

## Localization for Extension-Contributed Menus

To localize contributed menu items, extension authors must localize the strings
that menus render through command and submenu metadata.

Required steps:

1. Replace user-facing strings in `package.json` with `%key%` tokens.
2. Add default values for those keys in `package.nls.json`.
3. Add translated values in `package.nls.<locale>.json` files.
4. Keep keys stable so translations remain compatible across releases.

What must be localized for menu contributions:

- `contributes.commands[].title`
- `contributes.commands[].category` (if used)
- `contributes.commands[].shortTitle` (if used)
- `contributes.submenus[].label` (if used)

Fields such as `contributes.menus[].when` and `contributes.menus[].group` are
not user-facing labels and should not be localized.

Example:

```json
{
  "contributes": {
    "commands": [
      {
        "command": "sample.print",
        "title": "%command.sample.print.title%",
        "category": "%command.sample.category%"
      }
    ],
    "submenus": [
      {
        "id": "sample.printTools",
        "label": "%submenu.sample.printTools.label%"
      }
    ],
    "menus": {
      "menuBar/file": [
        { "command": "sample.print", "group": "4z_print@1" },
        { "submenu": "sample.printTools", "group": "4z_print@2" }
      ]
    }
  }
}
```

`package.nls.json`:

```json
{
  "command.sample.print.title": "Print...",
  "command.sample.category": "Sample",
  "submenu.sample.printTools.label": "Print Tools"
}
```

## 9. Troubleshooting Checklist

- Command id exists in `contributes.commands`.
- The same command id is used in `contributes.menus`.
- Menu location key is valid.
- `menus.when` evaluates to true when expected.
- `commands.enablement` is not unexpectedly false.
- Group string is valid (`section@order`) when ordering matters.
- No typo between command declaration and command handler registration.

## 10. Migration Notes

When migrating from static menu wiring to contributions:

1. Move command metadata to `contributes.commands`.
2. Move placement to `contributes.menus` with explicit groups.
3. Preserve command ids to keep existing handlers/keybindings working.
4. Add `enablement` to commands if disabled state is required.
5. Add `when` only for true visibility gating.

## 11. Built-In Group and Item Reference (Top-Level Menus)

This section lists the built-in group structure and registered items for top-level menus in this branch.

Notes:

- The list reflects static `MenuRegistry.appendMenuItem(...)` registrations found in source.
- Some items are conditional (`when`) and may not appear in every environment.
- Some command ids are constants/references in source (for example `SAVE_FILE_COMMAND_ID`).
- Some menus also receive additional actions through other runtime paths.

### 11.1 `menuBar/file` (`MenubarFileMenu`)

| Group | Item (command/submenu) | Source |
|---|---|---|
| `1_new` | `NEW_UNTITLED_FILE_COMMAND_ID` | `src/vs/workbench/contrib/files/browser/fileActions.contribution.ts` |
| `1_new` | `(inline/unknown)` | `src/vs/workbench/contrib/userDataProfile/browser/userDataProfile.ts` |
| `2_open` | `OpenFileAction.ID` | `src/vs/workbench/browser/actions/workspaceActions.ts` |
| `2_open` | `OpenFileFolderAction.ID` | `src/vs/workbench/browser/actions/workspaceActions.ts` |
| `2_open` | `OpenFolderAction.ID` | `src/vs/workbench/browser/actions/workspaceActions.ts` |
| `2_open` | `OpenFolderViaWorkspaceAction.ID` | `src/vs/workbench/browser/actions/workspaceActions.ts` |
| `2_open` | `OpenWorkspaceAction.ID` | `src/vs/workbench/browser/actions/workspaceActions.ts` |
| `2_open` | `submenu:MenubarRecentMenu` | `src/vs/workbench/browser/actions/windowActions.ts` |
| `3_workspace` | `ADD_ROOT_FOLDER_COMMAND_ID` | `src/vs/workbench/browser/actions/workspaceActions.ts` |
| `3_workspace` | `DuplicateWorkspaceInNewWindowAction.ID` | `src/vs/workbench/browser/actions/workspaceActions.ts` |
| `3_workspace` | `SaveWorkspaceAsAction.ID` | `src/vs/workbench/browser/actions/workspaceActions.ts` |
| `4_save` | `SAVE_FILE_COMMAND_ID` | `src/vs/workbench/contrib/files/browser/fileActions.contribution.ts` |
| `4_save` | `SAVE_FILE_AS_COMMAND_ID` | `src/vs/workbench/contrib/files/browser/fileActions.contribution.ts` |
| `4_save` | `SAVE_ALL_COMMAND_ID` | `src/vs/workbench/contrib/files/browser/fileActions.contribution.ts` |
| `45_share` | `submenu:MenubarShare` | `src/vs/workbench/browser/parts/editor/editor.contribution.ts` |
| `5_autosave` | `ToggleAutoSaveAction.ID` | `src/vs/workbench/contrib/files/browser/fileActions.contribution.ts` |
| `5_autosave` | `submenu:MenubarPreferencesMenu` | `src/vs/workbench/contrib/preferences/browser/preferences.contribution.ts` |
| `6_close` | `CLOSE_EDITOR_COMMAND_ID` | `src/vs/workbench/contrib/files/browser/fileActions.contribution.ts` |
| `6_close` | `REVERT_FILE_COMMAND_ID` | `src/vs/workbench/contrib/files/browser/fileActions.contribution.ts` |
| `6_close` | `CloseWorkspaceAction.ID` | `src/vs/workbench/browser/actions/workspaceActions.ts` |
| `6_close` | `RemoteStatusIndicator.CLOSE_REMOTE_COMMAND_ID` | `src/vs/workbench/contrib/remote/browser/remoteIndicator.ts` |
| `z_ConfirmClose` | `workbench.action.toggleConfirmBeforeClose` | `src/vs/workbench/browser/actions/windowActions.ts` |
| `z_Exit` | `workbench.action.quit` | `src/vs/workbench/electron-browser/desktop.contribution.ts` |

### 11.2 `menuBar/edit` (`MenubarEditMenu`)

| Group | Item (command/submenu) | Source |
|---|---|---|
| `2_ccp` | `submenu:MenubarCopy` | `src/vs/editor/contrib/clipboard/browser/clipboard.ts` |

### 11.3 `menuBar/selection` (`MenubarSelectionMenu`)

| Group | Item (command/submenu) | Source |
|---|---|---|
| `4_config` | `ToggleMultiCursorModifierAction.ID` | `src/vs/workbench/contrib/codeEditor/browser/toggleMultiCursorModifier.ts` |

### 11.4 `menuBar/view` (`MenubarViewMenu`)

| Group | Item (command/submenu) | Source |
|---|---|---|
| `(default)` | `(inline/unknown)` | `src/vs/workbench/services/views/browser/viewsService.ts` |
| `(default)` | `commandId` | `src/vs/workbench/services/views/browser/viewsService.ts` |
| `1_open` | `OpenViewPickerAction.ID` | `src/vs/workbench/contrib/quickaccess/browser/quickAccess.contribution.ts` |
| `1_open` | `ShowAllCommandsAction.ID` | `src/vs/workbench/contrib/quickaccess/browser/quickAccess.contribution.ts` |
| `2_appearance` | `submenu:MenubarAppearanceMenu` | `src/vs/workbench/browser/actions/layoutActions.ts` |
| `2_appearance` | `submenu:MenubarLayoutMenu` | `src/vs/workbench/browser/parts/editor/editor.contribution.ts` |
| `6_editor` | `TOGGLE_WORD_WRAP_ID` | `src/vs/workbench/contrib/codeEditor/browser/toggleWordWrap.ts` |

### 11.5 `menuBar/go` (`MenubarGoMenu`)

| Group | Item (command/submenu) | Source |
|---|---|---|
| `1_history_nav` | `workbench.action.navigateToLastEditLocation` | `src/vs/workbench/browser/parts/editor/editor.contribution.ts` |
| `2_editor_nav` | `submenu:MenubarSwitchEditorMenu` | `src/vs/workbench/browser/parts/editor/editor.contribution.ts` |
| `2_editor_nav` | `submenu:MenubarSwitchGroupMenu` | `src/vs/workbench/browser/parts/editor/editor.contribution.ts` |
| `3_global_nav` | `workbench.action.quickOpen` | `src/vs/workbench/contrib/files/browser/fileActions.contribution.ts` |
| `5_infile_nav` | `editor.action.jumpToBracket` | `src/vs/editor/contrib/bracketMatching/browser/bracketMatching.ts` |
| `5_infile_nav` | `workbench.action.gotoLine` | `src/vs/workbench/contrib/quickaccess/browser/quickAccess.contribution.ts` |
| `7_change_nav` | `editor.action.dirtydiff.next` | `src/vs/workbench/contrib/scm/browser/quickDiffWidget.ts` |
| `7_change_nav` | `editor.action.dirtydiff.previous` | `src/vs/workbench/contrib/scm/browser/quickDiffWidget.ts` |

### 11.6 `menuBar/terminal` (`MenubarTerminalMenu`)

| Group | Item (command/submenu) | Source |
|---|---|---|
| `(default)` | `workbench.action.tasks.build` | `src/vs/workbench/contrib/tasks/browser/task.contribution.ts` |
| `(default)` | `workbench.action.tasks.configureDefaultBuildTask` | `src/vs/workbench/contrib/tasks/browser/task.contribution.ts` |
| `(default)` | `workbench.action.tasks.configureTaskRunner` | `src/vs/workbench/contrib/tasks/browser/task.contribution.ts` |
| `(default)` | `workbench.action.tasks.restartTask` | `src/vs/workbench/contrib/tasks/browser/task.contribution.ts` |
| `(default)` | `workbench.action.tasks.runTask` | `src/vs/workbench/contrib/tasks/browser/task.contribution.ts` |
| `(default)` | `workbench.action.tasks.showTasks` | `src/vs/workbench/contrib/tasks/browser/task.contribution.ts` |
| `(default)` | `workbench.action.tasks.terminate` | `src/vs/workbench/contrib/tasks/browser/task.contribution.ts` |

### 11.7 `menuBar/help` (`MenubarHelpMenu`)

| Group | Item (command/submenu) | Source |
|---|---|---|
| `1_welcome` | `AskVSCodeCopilot.ID` | `src/vs/workbench/browser/actions/helpActions.ts` |
| `1_welcome` | `ShowAllCommandsAction.ID` | `src/vs/workbench/contrib/quickaccess/browser/quickAccess.contribution.ts` |
| `1_welcome` | `workbench.action.showInteractivePlayground` | `src/vs/workbench/contrib/welcomeWalkthrough/browser/walkThrough.contribution.ts` |
| `3_feedback` | `OpenIssueReporterActionId` | `src/vs/workbench/contrib/issue/common/issue.contribution.ts` |
| `5_tools` | `OpenProcessExplorer.ID` | `src/vs/workbench/contrib/processExplorer/browser/processExplorer.contribution.ts` |

### 11.8 `MenubarPreferencesMenu` (submenu used by other menus)

| Group | Item (command/submenu) | Source |
|---|---|---|
| `2_configuration` | `(inline/unknown)` | `src/vs/workbench/contrib/userDataProfile/browser/userDataProfile.ts` |
| `2_configuration` | `(inline/unknown)` | `src/vs/workbench/contrib/themes/browser/themes.contribution.ts` |
| `2_configuration` | `(inline/unknown)` | `src/vs/workbench/contrib/tasks/browser/task.contribution.ts` |
| `2_configuration` | `(inline/unknown)` | `src/vs/workbench/contrib/preferences/browser/preferences.contribution.ts` |
| `2_configuration` | `VIEWLET_ID` | `src/vs/workbench/contrib/extensions/browser/extensions.contribution.ts` |

## 12. How Built-In and Contributed Items Merge in Practice

When you contribute into a menu that already has built-ins:

1. Your item enters the same sorting pipeline as built-in menu items.
2. Group name controls section placement relative to built-in sections.
3. Order value (`@n`) controls ordering within a section.
4. A different group name from neighboring built-in groups creates a new section (with separators).

Example in `menuBar/file`:

- Built-ins use `4_save` and then `5_autosave`.
- If you contribute `4z_print@1`, it appears after `4_save` and before `5_autosave`.
- If you contribute `4_save@99`, it appears in the save section itself, after lower `4_save@*` entries.
- If you contribute `9_sample@1`, it appears much later, usually near end sections.

## 13. Predefined Groups vs Custom Groups

Using predefined/built-in groups:

- Pros: natural placement with familiar commands and fewer extra separators.
- Pros: predictable user mental model (for example print near save workflow).
- Cons: higher coupling to internal group naming conventions.

Using custom groups:

- Pros: clear ownership and easy isolation of your contributed section.
- Pros: lower risk of interleaving with built-ins unexpectedly.
- Cons: may create extra visual sections and feel less native if overused.

Practical guidance:

- Use built-in groups when your action semantically belongs there.
- Use custom groups for distinct feature buckets.
- Always set explicit `@order` for deterministic placement.
