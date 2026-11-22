# Personal recipe app implementation brief

Build a responsive web app for a household recipe collection. It should be easy to update recipes from a desktop, follow a timed workflow on an iPhone in the kitchen, and look up what worked on previous cooks. Make the same site accessible away from home.

## Access and deployment

- Let anyone browse, search, use the random picker, read saved cook history, and run cooking timers without signing in.
- Use GitHub sign-in for shared writes. Anyone with write access to the configured repository can create and edit recipes and cook logs and upload photos in the app. Check authorization on the server for every mutation, including when a collaborator's access has been revoked. Give read-only visitors a clear path to sign in when they choose to save.
- Use a public GitHub repository as the durable source of truth for recipes, cook logs, and photos. These files and prior committed versions are publicly readable. Put a brief public-visibility notice next to the save controls for notes and photos.
- Recommended architecture: one HTTPS web deployment with a responsive TypeScript frontend and same-origin serverless handlers for GitHub sign-in, reads, and writes. Use a GitHub App installed on the target repo with repository Contents read/write permission. Perform writes with the signed-in user's GitHub App user access token so GitHub applies both user and app permissions. Use a server-validated session with Secure, HttpOnly, SameSite cookies, OAuth state, and CSRF protection for writes. Keep tokens and secrets out of client code, browser storage, logs, and Git.
- Keep hosting provider and UI framework configurable implementation choices. Use a branch that allows the required data commits. Read live repository data with bounded caching and revalidation. A successful save must become available to the other device on refresh without waiting for a site rebuild. Avoid a separate database, object store, or dedicated always-on machine.

## Visual direction

- Use a dark theme built on slate blue-gray surfaces with a muted lavender or cool blue accent. Starting palette: page #202432, panels #2A3041, primary text #F1EEF5, and accent #C4B2F4. Verify text, focus rings, and timer states against their actual backgrounds. Use labels or icons alongside color for overdue and paused states.
- Take inspiration from Silksong’s ornate, delicate atmosphere. Create original curling, lace-like linework for page corners, section dividers, or the Discover header. Keep the flourishes subtle, away from text and controls, and hidden from assistive technology. Use original artwork and typography.
- Let the collection and recipe pages carry most of the ornament. Keep cooking mode quiet and legible on a phone at arm’s length, with large timer numerals, generous touch targets, and a clear overdue state.

## Recipe collection and editor

- Provide a Recipes view with search and links to recipe detail, and a Discover tab with random selection and recently cooked recipes.
- Search case-insensitively across visible recipe text, including title, category labels, ingredient section names and lines, step and substep titles, mise en place, actions, variation notes, and the source website URL. Include any additional recipe text fields that are rendered. Match text inside collapsed sections as well. Search leaves cook notes and cook logs out of the index. Use simple substring matching for the first version.
- Allow an optional source website URL. Edit it in the recipe form and display an “Original recipe” link on the recipe detail when present. Validate that the URL uses HTTP or HTTPS on both client and server.
- Allow optional freeform variation notes, with the same light Markdown editing and preview as recipe instructions. Edit them in the recipe form and show them under “Variations” on the recipe detail when present. Use this field for reusable ideas such as substitutions, alternate seasonings, and different cooking methods.
- Let editors paste ingredients directly into multiline fields, one ingredient per line. Preserve the written quantity, units, and text. Support a single unsectioned list or multiple ordered, named sections such as Dough, Filling, and Glaze. Let editors add, rename, reorder, and remove sections. Preserve the order of ingredient lines and ignore blank rows.
- Let editors assign multiple reusable category tags. Trim whitespace and match tags case-insensitively to prevent duplicate spellings that differ only by case. Preserve a readable display label.
- Model the workflow as ordered steps, each with zero or more ordered substeps. Enforce a maximum of one substep level in both the editor and server validation. Allow insertion, removal, and reordering at the appropriate level.
- Provide an Edit workflow action from the recipe detail so editors can revise steps, substeps, and durations without opening the ingredient and category form. Keep workflow data in the recipe record for now. Apply edits to the targeted fields with the existing revision check. Start future cooks from the latest workflow and preserve the workflow in any active local session.
- Give every step and substep a title, optional expected duration, freeform mise en place, and freeform actions. Mise en place stays independent from the master ingredient list. Use plain text input with light Markdown editing and preview for mise en place and actions. Render sanitized Markdown, preserve meaningful line breaks, and disable raw HTML.
- Render independent expand/collapse controls. A collapsed row shows the title and, during cooking, any active timer indicator. Expanded rows show mise en place before actions. Use accessible disclosure controls and a comfortable touch layout.

## Cooking mode

- Start with a local draft tied to a recipe and a stable cook ID. Capture the workflow's step IDs, titles, and expected durations when cooking starts so a recipe edit elsewhere cannot rewrite the record in progress.
- Allow independent stopwatches on any step or substep, including untimed steps and simultaneous parent/child timers. Support Start, Pause, Resume, and Stop. Paused intervals count toward neither actual duration nor the expected-duration threshold. Treat parent timers as independent measurements. Display active timers even if their workflow rows are collapsed.
- Use timestamps plus accumulated running time to calculate elapsed time. Persist every timer state transition so reopening or reloading the cook can reconstruct elapsed time. Use a visual tick only to update the display.
- When active elapsed time reaches the optional expected duration, alert once per stopwatch with a visible indication and a brief sound where browser audio is available. Keep counting until Stop. Initialize audio from a user gesture. If the page becomes visible after a threshold passed, show the overdue state and deliver the alert if it has yet to fire. Stopwatches work without expected durations.
- Request a screen wake lock during cooking when supported. Reacquire it when the visible page returns if needed and release it when leaving cooking mode. Foreground alerts are the requirement. Locked-screen and background alerts are outside this release.
- Let the cook review and correct recorded actual durations before saving. Keep the captured expected durations. If any watch is still running or paused at finish, present that state and let the user stop it or leave that timing out. Omit unused timers instead of recording zero.

## Cook history and offline behavior

- Provide a separate “Log a cook” action on each recipe for an after-the-fact entry with freeform notes and no timings. Pre-fill the cook date and let the user correct it for a past cook. Store the date/time consistently and display it in local time.
- On each recipe, list saved cooks newest first. A cook has a date/time, freeform notes, photos, and optional timing rows. Each timing row shows the step or substep name, expected duration if one was set, and actual duration. Preserve these captured labels and expected durations through recipe edits, including reorders and deletes of steps. After saving, allow edits to notes, actual durations, and photos while retaining the historical labels and expected durations.
- Persist active sessions and pending logs in IndexedDB, including draft notes, timing state, and any pending cook-photo files. Cache the app shell and the in-progress recipe content so a cook already started can resume after an offline reload. Timers and note entry must continue through a connection drop.
- Finish locally when offline or signed out. Show whether the cook is a draft, saved locally and pending, or synced. Keep the pending entry after a reload and offer a visible retry when back online, prompting for GitHub sign-in if needed. If the app can retry in the foreground with valid authorization, it may do so. Clear the pending state only once the server confirms the commit or confirms the identical write already landed.
- Active cook state is device-local. Finished, synced logs appear on both devices. Keep histories independent from active local drafts.

## Photos and discovery

- Attach photos to whichever screen accepted the upload. Recipe uploads appear in the recipe’s photo gallery. Cook-log uploads appear in both the recipe’s photo gallery and that specific cook log. In the recipe gallery, link cook photos back to their source cook. Derive the gallery from recipe-owned photos plus photos from synced cooks. Store each image once, and update both views when a cook photo is added or removed. Manage cook photos from their source cook log. Support the desktop file picker and iPhone photo library, including a clear conversion or error path for HEIC if received. Show previews and allow removal before save and when editing.
- Default to compressed web-friendly images. Correct orientation, strip metadata such as GPS, and validate decoded type, dimensions, and size server-side. Set and document upload limits that fit the host and GitHub request limits. For example, cap the longest edge at 1,600 px and target 500 KB per image while preserving reasonable quality. Make failure actionable and keep unsaved notes and photos on the device.
- In Discover, sample uniformly from recipes that have every selected category. An empty selection uses the whole collection. Recently cooked recipes remain eligible. Handle an empty match set and link the result directly to its recipe.
- Show the five most recently cooked distinct recipes. Rank each recipe by its latest saved cook date. An unsynced cook stays local until published.

## Repository format and write behavior

Use readable, versioned JSON records and stable IDs that survive renames and reorders. A workable layout:

```text
recipes/<recipe-id>.json
cooks/<recipe-id>/<cook-id>.json
photos/recipes/<recipe-id>/<photo-id>.jpg
photos/cooks/<recipe-id>/<cook-id>/<photo-id>.jpg
```

- Store recipe title, category tags, ordered ingredient sections, workflow, photo references, optional source URL, and optional variation notes in each recipe record. Existing recipes without the optional fields remain valid. Store recipe ID, cook ID, cooked-at timestamp, notes, optional timing rows, and photo references in each cook record. Use integer seconds for durations and an absent value for an unspecified expected duration. Use stable step IDs internally and a captured name in each history row. A substep may display its captured parent title for context.
- Read related files against a consistent Git revision. Derive search and recent lists from the records, with an optional short-lived server cache. Avoid a shared mutable index file that must change with every save.
- Restrict mutations to typed operations and server-generated data paths in the configured repository and branch. Validate IDs, payload shape, photo ownership, and size. Reject arbitrary file paths and extra fields. Keep one commit per logical save, including record and photo changes, using GitHub's createCommitOnBranch with expectedHeadOid or an equivalent atomic compare-and-write.
- Compare the saved record's revision with the version edited by the user. If only unrelated files changed, retry against the new branch head. If the same record changed, preserve the draft and show a conflict so the user can reload and reconcile. Use a stable cook ID and the same payload on retries to deduplicate network timeouts and double submissions. Treat an identical committed record and matching photo assets as success. Preserve the local draft on validation, permission, network, and conflict failures.

## Build order and acceptance checks

1. Implement schemas, fixtures, GitHub-backed reads, and responsive recipe browsing, search, and discovery. Verify that text in a collapsed substep is searchable, multiple categories use AND matching, and duplicate cooks produce one recent recipe entry.
2. Add GitHub sign-in and in-app recipe editing with sectioned paste and photo upload. Verify that anonymous and signed-in read-only visitors cannot write, collaborators can save, unauthorized paths are rejected, and a saved edit is visible from another device without a redeploy. Verify the source link opens the original website, variation notes appear in recipe search, and cook notes remain excluded.
3. Build cooking mode and local draft persistence. On iPhone Safari and a desktop browser, run overlapping parent and substep timers, pause one past its expected duration, resume it, and confirm a single foreground alert at the threshold. Verify accurate durations after an offline reload and that collapsing a step does not affect timing.
4. Add cook history, notes-only logs, photo attachments, and synchronization. Test anonymous cooking followed by sign-in, an offline cook with notes followed by reconnect, a repeated save after a response is lost, two editors updating the same record, and editing a recipe without changing its old timing records. Confirm saved cook edits and photos show up on the other device. Verify a cook photo appears on both the recipe and the source cook log, disappears from both when removed, and is stored once. Verify a workflow-only edit keeps the ingredient list and older timing records intact.
5. Deliver a short setup README covering local development, GitHub App registration and callback, expected environment variables, branch setup, deployment, public visibility and Git history, and backup/restore from a clone. Document browser alert behavior and verify the main flows on a real iPhone when available.

## Technical references

- [GitHub App permissions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app)
- [GitHub App user access tokens](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-user-access-token-for-a-github-app)
- [GitHub GraphQL commit operations](https://docs.github.com/en/enterprise-cloud@latest/graphql/reference/commits)
- [Screen Wake Lock API](https://developer.mozilla.org/en-US/docs/Web/API/Screen_Wake_Lock_API)
