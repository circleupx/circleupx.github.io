---
title: Accessibility testing in Playwright
tags: ["Testing", "Playwright", "a11y"]
author: "Yunier"
date: "2020-03-13"
description: "Guide on how to handle accessibility using Playwright"
---

I don't believe I've mention this before here, but I am a huge fan of [Hey](https://hey.com/). By far the **best** email service I have ever used. What makes Hey even cooler is the team behind Hey sharing they engineering approach to different problems. Be that through various [tweets](https://twitter.com/dhh/status/1275901955995385856) or blog post like [Scaling the hottest app in tech on AWS and Kubernetes](https://acloudguru.com/blog/engineering/scaling-the-hottest-app-in-tech-on-aws-and-kubernetes) which outline how they use k8s. Recently, they shared how to tackle ay11 under [hey accessibility is a lot of work](https://world.hey.com/michael/hey-accessibility-is-a-lot-of-work-785ec5cf). One thing that stood out was to me was their usage of [axe-core](https://github.com/dequelabs/axe-core). Axe-core is an accessibility engine for automated Web UI testing. Which reminded me of [playwright](https://playwright.dev/), so I started to wonder if the two could be combined, turns out they can be. Let's explore how to do that.

I'm going to reuse the project I created in my [last](https://www.yunier.dev/2021-02-28-testing-webapps-with-playwright/) playwright post, it already has few test and has been configure to use other packages like Mocha and Chai. To get started, axe-core needs to be installed, you can use the following command.

```shell
npm install axe-core
```

With axe now installed I can start building some test. I'll add new file under the test folder, I'll call it a11y.ts. I'll start by importing the required packages need for our test.

```javascript
import { Browser, Page } from 'playwright/types/types';
import { chromium } from 'playwright';
import { describe } from 'mocha';
import * as fs from 'fs';
import { AxePlugin, AxeResults } from 'axe-core';
import { expect } from 'chai';
```

Next, I'll need to configure playwright's browser and page object to run before and after each test. That can be done like this.

```javascript
let browser: Browser;
before(async () => {
    browser = await chromium.launch();
});

after(async () => {
    await browser.close();
});

let page: Page;
beforeEach(async () => {
    page = await browser.newPage();
});

afterEach(async () => {
    await page.close();
});
```

Base on the [axe docs](https://github.com/dequelabs/axe-core/blob/develop/doc/developer-guide.md), I need to include **axe.min.js** in any fixture or test system. That file can be loaded from the node_modules folder using node's own file system module.

```javascript
// Loads axe.min.js
const file: string = fs.readFileSync(require.resolve('axe-core/axe.min.js'), 'utf8');
```

The tricky part comes next, you see, you need to understand a very important aspect of playwright. That is that playwright has two execution contexts, one for playwright and one for the browser. These two execution context will never share state with each other, it is not possible to access a [window](https://developer.mozilla.org/en-US/docs/Web/API/Window) or [document](https://developer.mozilla.org/en-US/docs/Web/API/Document) directly in playwright's exection context. So, how can we get axe.min.js, which was loaded under playwright context injected into the browser context. The answer can be found under the [evaluate](https://playwright.dev/docs/api/class-page/#pageevaluatepagefunction-arg) function of playwright's [page](https://playwright.dev/docs/api/class-page) object. You see, the evaluate function can run a JavaScript function in the context of the web page and bring results back to the playwright environment. It can also take something that exist on the playwright environment and send it to the browser context. 

For example, if you wanted to get the value of a document's href that is running under the browser context loaded into a playwright context, you would do so like this.

```javascript
// After evaluation, href will have the same value as document.location.href 
const href = await page.evaluate(() => document.location.href);
```

For more information see the [playwright docs](https://playwright.dev/docs/core-concepts#execution-contexts-playwright-and-browser) on execution context.

The evaluate method can take a string or a function as an optional argument. I can use file system to read axe.min.js into memory as a string, then pass that as an argument on the evaluate method. This is how I can get axe.min.js to be included.

```javascript
const file: string = fs.readFileSync(require.resolve('axe-core/axe.min.js'), 'utf8');
await page.evaluate((minifiedAxe: string) => window.eval(minifiedAxe), file);
```

Now comes the next trick, and that is that I need to run axe under the context of the browser, see [#1638](https://github.com/microsoft/playwright/issues/1638#issuecomment-634233277). That means we have to use the evaluate method again. The problem here is that when the test are executed, axe will run under the playwright context, and will not be available under the browser context. The solution to this problem is to extend the [window](https://developer.mozilla.org/en-US/docs/Web/API/Window) interface to include axe, see [How to declare a new property on the Window object with Typescript](https://ourcodeworld.com/articles/read/337/how-to-declare-a-new-property-on-the-window-object-with-typescript) for more details.

I'll use the following code to include axe under the window interface.
```javascript
declare global {
    interface Window {
        axe: AxePlugin
    }
}
```

Putting everything we have talked about so far together yields the following test.

```javascript
describe('loading google.com successfully', function () {
    context('accesibility evaluation', function () {
        it('should pass accessibility test', async () => {
            await page.goto('https://www.google.com');
            const file: string = fs.readFileSync(require.resolve('axe-core/axe.min.js'), 'utf8');
            await page.evaluate((minifiedAxe: string) => window.eval(minifiedAxe), file);
            const evaluationResult: AxeResults = await page.evaluate(() => window.axe.run(window.document))
            expect(evaluationResult.violations).to.be.empty;
        });
    });
});
```

Let's review what the test above is doing.

1. Playwright's page object is used to nagivate to google.com.
2. Node's file system is used to load axe.min.js.
3. axe.min.js is injected onto the browser context using evaluate method.
4. axe.run is executed by providing it with document context.
5. Assert that no a11y violations were found on google.com.

The a11y.ts file should now look like this.

```javascript
import { Browser, Page } from 'playwright/types/types';
import { chromium } from 'playwright';
import { describe } from 'mocha';
import * as fs from 'fs';
import { AxePlugin, AxeResults } from 'axe-core';
import { expect } from 'chai';

declare global {
    interface Window {
        axe: AxePlugin
    }
}

let browser: Browser;
before(async () => {
    browser = await chromium.launch();
});

after(async () => {
    await browser.close();
});

let page: Page;
beforeEach(async () => {
    page = await browser.newPage();
});

afterEach(async () => {
    await page.close();
});

describe('loading google.com successfully', function () {
    context('accesibility evaluation', function () {
        it('should pass accessibility test', async () => {
            await page.goto('https://www.google.com');
            const file: string = fs.readFileSync(require.resolve('axe-core/axe.min.js'), 'utf8');
            await page.evaluate((minifiedAxe: string) => window.eval(minifiedAxe), file);
            const evaluationResult: AxeResults = await page.evaluate(() => window.axe.run(window.document))
            expect(evaluationResult.violations).to.be.empty;
        });
    });
});
```

As always, to execute the a11y test, plus the test that already existed on the project run **'npm test'**. If you run that command you will notice the following output.

```text
$ npm test

> e2eplaywright@1.0.0 test C:\Users\Yunier\Documents\playwright-demo
> mocha -r ts-node/register 'test/*.ts' --recursive --reporter mocha-junit-reporter --timeout 60000 --exit

npm ERR! Test failed.  See above for more details.
```

The test failed, I can use console.log to spit out the violation. Running the test again yields the following result.s

```json
{
	"id": "aria-required-attr",
	"impact": "critical",
	"tags": ["cat.aria", "wcag2a", "wcag412"],
	"description": "Ensures elements with ARIA roles have all required ARIA attributes",
	"help": "Required ARIA attributes must be provided",
	"helpUrl": "https://dequeuniversity.com/rules/axe/4.1/aria-required-attr?application=axeAPI",
	"nodes": [{
		"any": [{
			"id": "aria-required-attr",
			"data": ["aria-expanded"],
			"relatedNodes": [],
			"impact": "critical",
			"message": "Required ARIA attribute not present: aria-expanded"
		}],
		"all": [],
		"none": [],
		"impact": "critical",
		"html": "<input class=\"gLFyf gsfi\" jsaction=\"paste:puy29d;\" maxlength=\"2048\" name=\"q\" type=\"text\" aria-autocomplete=\"both\" aria-haspopup=\"false\" autocapitalize=\"off\" autocomplete=\"off\" autocorrect=\"off\" autofocus=\"\" role=\"combobox\" spellcheck=\"false\" title=\"Search\" value=\"\" aria-label=\"Search\" data-ved=\"0ahUKEwiV8IqSubHvAhXHjFkKHaP-DIQQ39UDCAY\">",
		"target": [".gLFyf"],
		"failureSummary": "Fix any of the following:\n  Required ARIA attribute not present: aria-expanded"
	}]
}
```

Awesome. I can now do a11y testing while using playwright. You should know, that the example above is the most basic a11y test you can create. Axe is a very powerful library that offers many configurations. You can take the code above and expand it by passing different configurations to the run method. Or you can take a different approach, that is to leverage an existing library that does most of the heavy lifting for you, a library like [axe-playwright](https://github.com/abhinaba-ghosh/axe-playwright).

**Credits:**
- [Hey, accessibility is a lot of work!](https://world.hey.com/michael/hey-accessibility-is-a-lot-of-work-785ec5cf)
- [Accessibility Testing[Question] #1638](https://github.com/microsoft/playwright/issues/1638)
- [Writing unit tests in TypeScript](https://medium.com/@RupaniChirag/writing-unit-tests-in-typescript-d4719b8a0a40)
- [How to declare a new property on the Window object with Typescript](https://ourcodeworld.com/articles/read/337/how-to-declare-a-new-property-on-the-window-object-with-typescript)