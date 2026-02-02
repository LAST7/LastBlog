---
title: WXT 1 - 跨浏览器的插件侧板控制
date: 2026-01-30 17:08:26
categories: 笔记
tags:
    - browser
    - wxt
excerpt: WXT 框架下浏览器插件对 Firefox 及 chromium 系浏览器的侧边栏控制差异
---

## webextension-polyfill

- 查阅 wxt 相关 [README](https://wxt.dev/guide/essentials/extension-apis.html#using-webextension-polyfill)

    ```bash
    pnpm i @wxt-dev/webextension-polyfill webextension-polyfill
    ```

- 以及在 `wxt.config.ts` 中加入：

    ```ts
    export default defineConfig({
        //...
        modules: ["@wxt-dev/webextension-polyfill"]
        //...
    });
    ```

- 此外，还需要 `@types/webextension-polyfill` 以及 `@types/firefox-webext-browser` 来扩展 `browser` 的 Firefox 成员。

    ```bash
    pnpm i -D @types/firefox-webext-browser
    ```

## 括号作为行开头

![semi-1.png](https://s2.loli.net/2026/01/31/SrWPQwFyUzN8JLx.png)

![semi-2.png](https://s2.loli.net/2026/01/31/n5kgCxzQI6uaVKr.png)

- 这就是 JavaScript ASI 吗，给我整笑了。

---

## 默认打开 Side Panel

### Firefox

- 经测试，`manifest.json` 中的 `sidebar_action` 或者 `side_panel` 事实上没有任何作用。

- 也可能是 WXT 抹平了这其中的一些坑，没有细究。

---

- 对于 Firefox，点击扩展图标触发 listener 函数的机制和 `default_popup` 是 **冲突** 的，这意味着如果浏览器发现 manifest 中的 `action` 一项中存在 `default_popup`，那么点击扩展图标只会打开 popup 窗口而不会触发 `background.ts` 中挂载的 listener 函数。

- 因此，想要默认打开侧板，需要删除 `entrypoints` 中的 `popup` 来防止 wxt 自动在 `manifest.json` 中生成 `default_popup`。
    - 也可以通过一种奇怪的方式来让 WXT 忽略 `popup`:

        > https://github.com/wxt-dev/wxt/issues/1433

- 但是不能完全删去 `action`，因为这样一来`background.ts` 就无法访问 `browser.browserAction` 或者 `browser.action`。

    > https://github.com/wxt-dev/wxt/issues/669
    > https://wxt.dev/guide/essentials/config/manifest.html#action-without-popup
    - 可以在 `wxt.config.ts` 中的 `manifest` 部分添加一个空的 `action`，如下所示：

        ```ts
        export default defineConfig({
            // ...
            manifest: {
                // ...
                action: {}
                // ...
            }
            // ...
        });
        ```

- 然后在 `background.ts` 中注册监听函数：

    ```ts
    if (import.meta.env.MANIFEST_VERSION === 3) {
        browser.action.onClicked.addListener(sidePanelManager.open);
    } else {
        browser.browserAction.onClicked.addListener(sidePanelManager.open);
    }
    ```

### Chrome

- 同样的，需要 `manifest.json` 中包含 `action`，且需要删除 `popup`：

    > https://developer.chrome.com/docs/extensions/reference/api/sidePanel#open-action-icon

- 经测试，当 `PanelBehavior` 中的 `openPanelOnActionClick` 被设为 `true` 的时候，点击扩展图标同样不会触发 `background.ts` 中挂载的 listener 函数。

- `openPanelOnActionClick` 默认设置为 `false`，保险起见再手动设置一遍：

    ```ts
    // Chrome specific: Disable the default declarative behavior to ensure
    // the onClicked event is dispatched to our listener.
    if (import.meta.env.CHROME && typeof chrome.sidePanel?.setPanelBehavior === "function") {
        chrome.sidePanel.setPanelBehavior({ openPanelOnActionClick: false }).catch((error) => {
            console.error("[Chrome] Error resetting panel behavior: ", error);
        });
    }
    ```

### 完整代码

{% folding purple::@/utils/sidepanel.ts %}

```ts
import { Tabs } from "webextension-polyfill";

/**
 * Defines the contract for side panel operations across different browsers.
 * This abstraction allows the rest of the extension to trigger UI components
 * without platform-specific branching.
 */
export interface SidePanelManager {
    /**
     * Toggles or opens the side panel depending on the browser's capabilities.
     * @param tab The current active tab.
     */
    open: (tab: Tabs.Tab) => Promise<void>;
}

/**
 * Chrome implementation using the MV3 sidePanel API.
 * @requires "sidePanel" permission in manifest.json
 */
async function ChromeOpenSidePanel(tab: Tabs.Tab) {
    // Ensure the API exists
    if (typeof chrome.sidePanel?.open !== "function") {
        console.error(
            "[Chrome] sidePanel.open is not available. Check 'sidePanel' permission and ensure Chrome version >= 116"
        );
        return;
    }

    // Ensure a valid `windowId`
    const windowId = tab.windowId ?? chrome.windows.WINDOW_ID_CURRENT;

    try {
        await chrome.sidePanel.open({ windowId: tab.windowId });
        console.log("[Chrome] Side panel opened via action click.");
    } catch (error) {
        console.error("[Chrome] Failed to open side panel: ", error);
    }
}

/**
 * Firefox implementation using the sidebarAction API.
 * @requires "sidebar_action" definition in manifest.json
 */
async function FirefoxOpenSidePanel(tab: Tabs.Tab) {
    try {
        await browser.sidebarAction.open();
        console.log("[Firefox] Side panel opened via action click.");
    } catch (error) {
        console.error("[Firefox] Failed to open side panel: ", error);
    }
}

const panelImpl: Record<string, SidePanelManager> = {
    chrome: { open: ChromeOpenSidePanel },
    firefox: { open: FirefoxOpenSidePanel }
};

/**
 * A platform-aware manager instance.
 * WXT optimizes this at build-time, removing the unused platform implementation.
 */
export const sidePanelManager: SidePanelManager =
    panelImpl[import.meta.env.BROWSER] ?? panelImpl["chrome"];

/**
 * Configures the extension icon to open the side panel as the default behavior.
 *
 * @description
 * This function registers a listener to the extension's action icon.
 * - **Chrome**: Triggers `chrome.sidePanel.open`. Requires `openPanelOnActionClick: false`.
 * - **Firefox**: Triggers `sidebarAction.open`.
 *
 * @architecture
 * Uses `browser.action` for MV3 and fallbacks to `browser.browserAction` for MV2 (Firefox).
 * The imperative approach is used here to allow potential pre-open side effects.
 */
export function openSidePanelAsDefault() {
    // TODO: When in need of multiple listener functions, define a main listener
    // function and call the others inside it to maintain order
    // FIXME: async function registered as callback causing problem?
    if (import.meta.env.MANIFEST_VERSION === 3) {
        browser.action.onClicked.addListener(sidePanelManager.open);
    } else {
        browser.browserAction.onClicked.addListener(sidePanelManager.open);
    }

    // Chrome specific: Disable the default declarative behavior to ensure
    // the onClicked event is dispatched to our listener.
    if (import.meta.env.CHROME && typeof chrome.sidePanel?.setPanelBehavior === "function") {
        chrome.sidePanel.setPanelBehavior({ openPanelOnActionClick: false }).catch((error) => {
            console.error("[Chrome] Error resetting panel behavior: ", error);
        });
    }
}
```

{% endfolding %}
