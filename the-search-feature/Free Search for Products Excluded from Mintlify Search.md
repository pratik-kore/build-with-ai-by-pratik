# Free Search for Products Excluded from Mintlify Search

Setup guide for the `docs-v2` repository (Mintlify docs, `.mdx` files). Prepared 8 October 2026.

## Goal

Mintlify's built-in search covers only one of our five products. The other products are intentionally left out of search and carry `noindex`. This guide adds a free search box to the `/ai-for-service/` pages without any paid service, any server, or any change to the `noindex` settings.

## How it works

1. A GitHub Action reads every `.mdx` file under `ai-for-service/` and builds one JSON index file.
2. The Action saves that file to a branch called `search-index` in the same repository.
3. A small script, `search.js`, in the docs root runs on every Mintlify page. On `/ai-for-service/` pages it adds a search box at the top, loads the JSON index, and searches it in the visitor's browser.
4. On every push to the branch, the index is rebuilt automatically. No manual steps are needed after setup.

The search box does not appear on other products' pages.

## Prerequisites

- Admin or write access to `Koredotcom/docs-v2` (the repo is public).
- The branch `ai-for-service-dev` exists. Content reaches the live site by merging to `main`.
- GitHub Actions is enabled for the repo.

## Files to add

| File | Path in repo | Branch |
| --- | --- | --- |
| Index builder | `.github/scripts/build-index.cjs` | `ai-for-service-dev` |
| Workflow | `.github/workflows/build-search-index.yml` | `ai-for-service-dev` |
| Search box | `search.js` (docs root, next to `docs.json`) | The branch Mintlify deploys |

Notes on placement:

- The builder script lives inside `.github/` and uses the `.cjs` extension on purpose. Mintlify loads every `.js` file in the docs folder on all pages, so a normal `.js` script elsewhere would be loaded in visitors' browsers.
- `search.js` is the one file that is meant to load on all pages. It checks the page address itself and only shows the box on `/ai-for-service/` pages.

## Step 1: Allow the workflow to write

In the GitHub repo, go to **Settings → Actions → General → Workflow permissions**, select **Read and write permissions**, and save. If the option is greyed out, ask an organization admin.

## Step 2: Add the index builder and workflow

Using Visual Studio (or any Git client):

1. Switch to `ai-for-service-dev` and pull the latest changes.
2. Add `.github/scripts/build-index.cjs` (create the `scripts` folder inside the existing `.github` folder).
3. Add `.github/workflows/build-search-index.yml`.
4. Commit and push.

The workflow starts on its own after the push. Its trigger is:

```yaml
on:
  push:
    branches: [ai-for-service-dev]
    paths:
      - 'ai-for-service/**'
      - '.github/scripts/build-index.cjs'
  workflow_dispatch:
```

It runs `node .github/scripts/build-index.cjs ai-for-service /tmp/out/ai-for-service.json` and publishes the result to the `search-index` branch, which it creates on the first run.

## Step 3: Verify the index

1. Open the repo's **Actions** tab and wait for a green tick on **Build search index**.
2. Open this link in a browser. It should show JSON text:

   `https://raw.githubusercontent.com/Koredotcom/docs-v2/search-index/ai-for-service.json`

Do not continue to Step 4 until this link shows JSON.

## Step 4: Add the search box

1. In `search.js`, confirm the index address in the `PRODUCTS` list:

```js
var PRODUCTS = [
  {
    prefix: '/ai-for-service/',
    index: 'https://raw.githubusercontent.com/Koredotcom/docs-v2/search-index/ai-for-service.json',
  },
];
```

2. Put `search.js` in the docs root on the branch Mintlify deploys, commit, and push.
3. Wait for Mintlify to redeploy, then hard refresh the page (Ctrl+Shift+R).

## Step 5: Test

1. Open any page under `https://docs.kore.ai/ai-for-service/`. A search box with a magnifying glass and the text **Search...** appears at the top of the page.
2. Type a keyword such as `MCP`. Results appear as you type.
3. Click a result, or one of the small section links under it, and confirm it opens the right page and section.
4. Open a page from another product and confirm the box does not appear.

## What users see

- **One result per page.** Each page is listed once, with its title, best-matching section, and a text snippet. Up to 8 pages are shown.
- **Section links.** Other matching sections of the same page appear as small links under the result, and each jumps to that section.
- **Ranking.** Whole-word matches in a page title or heading rank above passing mentions in the text.
- **Highlighting.** Matched words are shown in green, semi-bold text (a lighter green in dark mode).
- **Recent searches.** Clicking into the empty box lists the last 5 searches, with a Clear link. These are stored only in the visitor's own browser.
- **Shortcut.** Ctrl+Shift+K (Cmd+Shift+K on a Mac) moves the cursor into the box. Esc closes the results.

## Keeping the index up to date

Nothing needs to be done by hand. Each push to `ai-for-service-dev` that changes a file under `ai-for-service/` rebuilds the whole index, so new, edited, and deleted pages are all handled. Changes appear in search about 5 minutes after the workflow finishes, because GitHub caches the file for a short time.

To check that the index is current, look at the date of the latest commit to `ai-for-service.json` on the `search-index` branch, or check for a green run in the Actions tab.

If changes are ever pushed straight to `main` without going through `ai-for-service-dev`, add `main` to the `branches` list in the workflow (and copy the workflow file to `main`), otherwise the index will not update.

## Settings you can change in search.js

| Setting | What it does |
| --- | --- |
| `PRODUCTS` | Lists the products that get search: the page address prefix and the index file for each |
| `HIDE_PREVIEW_WIDGET` | When `true`, hides Mintlify's floating Preview Widget icon (bottom right). Set to `false` to show it |
| `MAX_RECENT` | Number of recent searches kept (default 5) |
| `#cs-wrap` CSS rule | Width and position of the search box |

## Adding another product later

1. Copy the workflow file for the new product, changing the branch name, the folder path, and the output file name (for example `product-b.json`).
2. Add one more entry to the `PRODUCTS` list in `search.js` with the product's page prefix and index address.
3. Commit and push. The same automatic updates then apply.

## Troubleshooting

| Symptom | Likely cause and fix |
| --- | --- |
| Workflow fails with a permission error | Step 1 was skipped. Set workflow permissions to read and write |
| Index link in Step 3 shows a 404 | The workflow has not run successfully yet. Check the Actions tab |
| No search box on the page | `search.js` is not on the branch Mintlify deploys, Mintlify has not finished redeploying, or the page cache is old. Hard refresh |
| Box shows "Search is unavailable right now" | The index file is missing or unreachable. Re-check Step 3 |
| Results look out of date | Wait about 5 minutes for GitHub's cache, then check the latest workflow run |
| Box overlaps other header icons | Reduce the width or shift the box left in the `#cs-wrap` CSS rule |

## Limitations

- Text inside code blocks is not indexed, to keep the index small.
- The index file is public, like the pages themselves. Pages are protected from Google by `noindex`, not by access control. If any content must be private, this approach is not suitable.
- The Preview Widget is hidden with CSS only. It is still present in the page for anyone who inspects it.
