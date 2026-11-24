/*
 * Vencord, a Discord client mod
 * Copyright (c) 2026 Vendicated and contributors
 * SPDX-License-Identifier: GPL-3.0-or-later
 */

import { Devs } from "@utils/constants";
import { Logger } from "@utils/Logger";
import definePlugin from "@utils/types";
import { CloudUpload as TCloudUpload } from "@vencord/discord-types";
import { CloudUploadPlatform } from "@vencord/discord-types/enums";
import { findByPropsLazy, findLazy } from "@webpack";
import { ChannelStore, MessageActions, showToast, UserStore } from "@webpack/common";

const CloudUpload: typeof TCloudUpload = findLazy(m => m.prototype?.trackUploadFinished);

// The module exposing the modern DM API: openPrivateChannel({recipientIds}), getOrEnsurePrivateChannel(userId)
const ChannelActionCreators = findByPropsLazy("openPrivateChannel", "getOrEnsurePrivateChannel");

const LOGGER = new Logger("UserPopoutImagePaste");

function getUserFromFiber(el: Element): string | null {
    const fiberKey = Object.keys(el).find(k => k.startsWith("__reactFiber$"));
    if (!fiberKey) return null;

    let fiber: any = (el as any)[fiberKey];

    // Walk up the component tree looking for a `user` / `userId` prop
    for (let i = 0; fiber && i < 100; i++, fiber = fiber.return) {
        const props = fiber?.memoizedProps ?? fiber?.pendingProps;
        if (props) {
            if (typeof props.userId === "string" && /^\d{5,}$/.test(props.userId)) return props.userId;
            if (typeof props.user?.id === "string" && /^\d{5,}$/.test(props.user.id)) return props.user.id;
        }
    }

    return null;
}

function getUserIdFromDom(popout: Element): string | null {
    // Discord stores the snowflake in avatar / banner urls
    for (const attr of ["src", "href"]) {
        for (const el of popout.querySelectorAll(`[${attr}*="/avatars/"], [${attr}*="/users/"], [${attr}*="/banners/"]`)) {
            const url = el.getAttribute(attr) ?? "";
            const match = url.match(/\/(\d{17,20})\//);
            if (match) return match[1];
        }
    }
    return null;
}

function getActivePopout(target: EventTarget | null): Element | null {
    // The element the paste happened in, if inside a profile popout
    if (target instanceof Element) {
        const inside = target.closest(".user-profile-popout, [class*='userPopout']");
        if (inside) return inside;
    }

    // Fallback: any currently open popout in this window
    return document.querySelector(".user-profile-popout, [class*='userPopout']");
}

function findPopoutUserId(popout: Element): string | null {
    return getUserFromFiber(popout) ?? getUserIdFromDom(popout);
}

function handlePaste(e: ClipboardEvent) {
    const files = Array.from(e.clipboardData?.files ?? []).filter(f => f.type.startsWith("image/"));
    if (!files.length) return;

    const popout = getActivePopout(e.target);
    if (!popout) return;

    const userId = findPopoutUserId(popout);
    if (!userId) {
        LOGGER.warn("Found a popout but could not determine the user id. Popout:", popout);
        showToast({ text: "Could not determine which user this popout belongs to", variant: "critical" });
        return;
    }

    if (userId === UserStore.getCurrentUser().id) {
        showToast({ text: "Cannot send images to yourself", variant: "critical" });
        return;
    }

    // We handle this paste ourselves
    e.preventDefault();
    e.stopImmediatePropagation();

    showToast({ text: "Opening DM and sending image..." });

    switchToDMAndSend(userId, files);
}

async function switchToDMAndSend(userId: string, files: File[]) {
    try {
        // Navigates to the DM with the user and resolves with the channel id
        const channelId: string = await ChannelActionCreators.openPrivateChannel({
            recipientIds: [userId],
            location: "UserPopoutImagePaste"
        });

        if (!channelId || !ChannelStore.hasChannel(channelId)) {
            showToast({ text: "Failed to open DM channel", variant: "critical" });
            return;
        }

        // Build CloudUploads, then let Discord's own sendMessage upload + send them
        const uploads = files.map(file => new CloudUpload(
            {
                file,
                isThumbnail: false,
                platform: CloudUploadPlatform.WEB
            },
            channelId
        ));

        MessageActions.sendMessage(
            channelId,
            {
                content: "",
                tts: false,
                invalidEmojis: [],
                validNonShortcutEmojis: []
            },
            true, // wait for channel ready, we just switched to it
            {
                location: "UserPopoutImagePaste",
                attachmentsToUpload: uploads
            }
        );

        showToast({
            text: files.length === 1 ? "Image sent to DM!" : `${files.length} images sent to DM!`,
            variant: "success"
        });
    } catch (err) {
        LOGGER.error("Failed to send image to DM", err);
        showToast({ text: "Failed to send image to DM", variant: "critical" });
    }
}

export default definePlugin({
    name: "UserPopoutImagePaste",
    description: "Press Ctrl+V with an image copied while a user popout is open to automatically open their DM and send the image",
    tags: ["Chat", "Utility", "Media"],
    authors: [Devs.Ven],
    enabledByDefault: true,

    start() {
        document.addEventListener("paste", handlePaste, { capture: true });
    },

    stop() {
        document.removeEventListener("paste", handlePaste, { capture: true });
    }
});
