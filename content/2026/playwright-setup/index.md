---
layout: post
title: "Playwright Startup Guide"
description: "Initial steps of Playwright setup"
date: 2026-09-22
type: guide
tags: ["automation", "web", "playwright"]
featured: false
ai: none
---

## Intro

I have always belived that playright (yeah I also thought it was right) is a automation software. I will say do this like clicking buttons and it will follow my lead. Now I am seeing it's just a npm package, well more than a package i believe. It takes my instructions in programming language not mouse click and play by my side. Often I am so impressed that I should have learn it before stating development. You know having tools before starting to build a car !

## Playing

However, simple playwright test suite improvises in three stages:

- ARRANGE: Setup the testing environment, perform pre-action jobs
- ACT: Perform required action
- ASSERT: Verify outcome and logging

```javascript
test("description of what you are testing", async ({ page }) => {
  // 1. ARRANGE: Navigate to a URL
  await page.goto("https://develop.skest.info");

  // 2. ACT: Perform actions (click, type, select)
  await page.locator('input[type="email"]').fill("user@example.com");
  await page.locator('button[type="submit"]').click();

  // 3. ASSERT: Verify the expected outcome
  await expect(page.locator(".welcome-message")).toBeVisible();
});
```

## Write Tools

To perform a proper action we have basic apis such as tracking elements in dom by html elements, tags, placeholders etc.

| Method               | Best Used For                            | Example                                         |
| -------------------- | ---------------------------------------- | ----------------------------------------------- |
| `getByRole()`        | Buttons, links, headings, checkboxes     | `page.getByRole('button', { name: 'Submit' })`  |
| `getByLabel()`       | Form inputs with explicit `<label>` tags | `page.getByLabel('Password')`                   |
| `getByPlaceholder()` | Inputs identified by placeholder text    | `page.getByPlaceholder('Enter your search...')` |
| `getByText()`        | Non-interactive text (paragraphs, spans) | `page.getByText('Welcome back!')`               |
| `getByTestId()`      | Custom test attributes (`data-testid`)   | `page.getByTestId('profile-card')`              |

## Essentials

Functionally our playwright automatically has sleep functionality builtin in the actions. So doing continuous action with `await` does feels like human clicking buttons filling up forms etc.

```js
await page.getByRole("button", { name: "Save" }).click(); // Click
await page.getByLabel("Username").fill("akib"); // Type text
await page.getByLabel("Subscribe").check(); // Check checkbox
await page.getByRole("combobox").selectOption("Option Value"); // Dropdown select
```

The Assersion is performeed via expectation and result which is automatically evaluated. Some profound assertions are:

```js
// Visibility & State
await expect(page.getByText("Dashboard")).toBeVisible();
await expect(page.getByRole("button")).toBeDisabled();

// Text & Values
await expect(page.locator("h1")).toHaveText("Welcome back, Akib");
await expect(page.getByLabel("Username")).toHaveValue("akib");

// Page URL & Title
await expect(page).toHaveURL("https://develop.skest.info/dashboard");
```

## Play or Write

Here comes the best feature we should be talked about in the first place. When you do not want to write test case for every button, you do this:

```bash
npx playwright codegen https://develop.skest.info
```

Now what happens?

- A Browser window opens
- You click around
- The inspector tracks browser `Javascript`

## Commands

Run tests in interactive UI Mode:

```Bash
npx playwright test --ui
```

Run tests with a visible browser window (headed):

```Bash
npx playwright test --headed
```

View the latest HTML test execution report:

```Bash
npx playwright show-report
```
