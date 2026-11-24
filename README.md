# UserPopoutImagePaste

A Vencord custom plugin that lets you press **Ctrl+V** with an image in a user profile popout to automatically open their DM and send the image.

## Installation

1. **Install Vencord** on your Discord Desktop client if you haven't already:
   - Follow the [Vencord installation guide](https://docs.vencord.dev/installing/)

2. **Add the plugin to your Vencord instance**:
   - Navigate to your Vencord `src/userplugins` folder
   - Copy the `userPopoutImagePaste` folder (or the `index.ts` file) into it:
     ```bash
     cp -r /path/to/this/plugin/src/userplugins/userPopoutImagePaste /path/to/your/Vencord/src/userplugins/
     ```
   - Or, if you're using the single-file version:
     ```bash
     cp /path/to/this/plugin/plugin.txt /path/to/your/Vencord/src/userplugins/
     ```

3. **Restart Discord** completely (kill the process from Task Manager if needed)

4. **Enable the plugin**:
   - Go to **User Settings** → **Plugins** (under "Personalize Server")
   - Find **"UserPopoutImagePaste"** in the list
   - Toggle it **ON**

## How it works

- Open any user's profile popout (right-click their avatar → "Profile")
- **Copy** an image to your clipboard (Ctrl+C on an image, or screenshot to clipboard)
- **Press Ctrl+V** in the popout
- The plugin will:
  1. Identify which user popout you're in
  2. Open that user's DM channel
  3. Send the image through Discord's native upload pipeline
  4. You'll see a toast notification when it's done

## How it works under the hood

- Reads the user ID from the popout's React fiber tree (falls back to avatar URL parsing)
- Uses Discord's `openPrivateChannel({ recipientIds })` API to open the DM
- Uses Discord's `CloudUpload` + `MessageActions.sendMessage(..., { attachmentsToUpload })` pipeline
- No webpack patches — safe from Discord updates breaking it

## Requirements

- [Vencord](https://vencord.download/) installed on Discord Desktop
- Discord Desktop client (not browser/web version)

## Development

If you want to modify this plugin:
- Edit `src/userplugins/userPopoutImagePaste/index.ts`
- Rebuild Vencord: `pnpm buildStandalone`
- Restart Discord to see changes

## License

[GPL-3.0-or-later](https://github.com/Vendicated/Vencord/blob/master/LICENSE)
