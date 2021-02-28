---
title: Testing WebApps with Playwright
layout: post
tags: ["Testing", "Playwright"]
readtime: true
---

A few weeks ago I was looking for an end-to-end testing framework. An alternative to [Selenium](https://www.selenium.dev/), and all the other end-to-end frameworks. I came across a project called [Playwright](https://playwright.dev/). Playwright is a new end-to-end framewrok created and maintained by Microsoft, it allows you to test web applications on different browsers. Some the major feature it provides are as follows.

1. Playwright has full API coverage for all modern browsers, including Google Chrome and Microsoft Edge (with Chromium), Apple Safari (with WebKit) and Mozilla Firefox.
2. Supports multiple languages like Node.js, Python, c# and Java.
3. First-party Docker image and GitHub Actions to deploy tests to your preferred CI/CD provider.
4. Use device emulation to test your responsive web apps in mobile web browsers.
5. Provides APIs to monitor and modify network traffic, both HTTP and HTTPS. Any requests that page does, including XHRs and fetch requests, can be tracked, modified and handled.

While those feature are all great and useful, they don't measure up to what I consider to be the best feature of Playwright. That is being able to create and execute test as easily as unit tests. You can also leverage tools like [qa wolf](https://www.qawolf.com/) and [headless-recorder](https://github.com/checkly/headless-recorder), these tools record any action you take on the browser, those actions are then converted into Playwright scripts.

For today's post, I am going to build a small playwright project, I will use this project to demonstrate how to create some basic tests using playwright. To follow along you will need to have [Node.js](https://nodejs.org/en/) installed, version 10.17 or greater. 

Open your favorite cli tool, [windows terminal](https://github.com/Microsoft/Terminal) in my case. Run the following npm command to install playwright

```text
npm i -D playwright
```

With playwright now installed, create a new file on the same directory you opened your terminal. Call the file google.ts, copy and paste the following code.

```javascript
const {chromium} = require('playwright');

(async () => {
    const browser = await chromium.launch();
    const page = await browser.newPage();
    await page.goto('https://www.google.com');
    await page.screenshot({path: 'google.png'});
    await browser.close();
})();
```

The first line of code imports the playwright library, then an asynchronous context is created, this is where playwright is executed. The second line of code launches an instace of chrome, from that instance a new page is created, a go to url is provided so that it can be visited, once the page is loaded, a screenshot is saved to the directory playwright is running under. Finally, the browser instance is disposed.

To execute the code, run the following command on the cli.

```
node google.ts
```

This is the most basic example on how to use playwright. Time to take things to the next level. Clean the working directory, delete all files. Create a new node.js project by running the following command.

```text
npm init
```

Provided the required information such as project name, author, description and so on. A package.json fille will be created with the information you provided.

```json
{
  "name": "e2eplaywright",
  "version": "1.0.0",
  "description": "Sample playwright repository",
  "main": "index.js",
  "scripts": {
    "test": "mocha"
  },
  "keywords": [
    "playwright"
  ],
  "author": "yunier",
  "license": "ISC"
}

```

Time to establish the folder structure. Whenever I work with a Node.js library, I like to follow the project structure outlined in [this](https://stackoverflow.com/questions/5178334/folder-structure-for-a-node-js-project) stack overflow post. 

Fow now I will only add the test folder using the following command.

```text
mkdir test
```

It is time to install a few additional node packages. Since I wiped the directory clean, I will need to reinstall playwright, I will also need a testing library, Playwright supports [Mocha](https://mochajs.org/), [Jest](https://github.com/playwright-community/jest-playwright) and [Ava](https://github.com/avajs/ava). For this repo I will use Mocha. I will also need an assertion library, you can use whichever library you want on your project, [chaijs](https://github.com/chaijs/chai) is my preferred framework. I work a lot with postman for API testing, postman uses chaijs to do assertions so I'm most familiar with chaijs and I know the syntax by heart. Other packages I will need are ts-node, typescript and mocha-junit-reporter. 

Run the following command to install the required packages

```
npm install playwright ts-node typescript eslint mocha chai @types/chai @types/faker @types/mocha mocha-junit-reporter
```

Your package.json should look somewhat like this.

```json
{
  "name": "e2eplaywright",
  "version": "1.0.0",
  "description": "Sample playwright repository",
  "scripts": {
    "test": "mocha"
  },
  "keywords": [
    "playwright"
  ],
  "author": "yunier",
  "license": "ISC",
  "dependencies": {
    "@types/chai": "^4.2.15",
    "@types/faker": "^5.1.7",
    "@types/mocha": "^8.2.1",
    "chai": "^4.3.0",
    "eslint": "^7.21.0",
    "mocha": "^8.3.0",
    "mocha-junit-reporter": "^2.0.0",
    "playwright": "^1.9.1",
    "ts-node": "^9.1.1",
    "typescript": "^4.2.2"
  }
}
```

I do want to make a slight modification. I will repalce the scripts field with the following script commands to let npm know how to handle the command "npm test" and "npm lint".

```json
{
    "scripts" : {
        "lint" : "eslint .",
        "test" : "mocha -r ts-node/register 'test/*.ts' --recursive --reporter mocha-junit-reporter --timeout 60000 --exit"
    }
}
```

Your package.json should now look like this.

```json
{
  "name": "e2eplaywright",
  "version": "1.0.0",
  "description": "Sample playwright repository",
  "scripts": {
    "lint": "eslint .",
    "test": "mocha -r ts-node/register 'test/*.ts' --recursive --reporter mocha-junit-reporter --timeout 60000 --exit"
  },
  "keywords": [
    "playwright"
  ],
  "author": "yunier",
  "license": "ISC",
  "dependencies": {
    "@types/chai": "^4.2.15",
    "@types/faker": "^5.1.7",
    "@types/mocha": "^8.2.1",
    "chai": "^4.3.0",
    "eslint": "^7.21.0",
    "mocha": "^8.3.0",
    "mocha-junit-reporter": "^2.0.0",
    "playwright": "^1.9.1",
    "ts-node": "^9.1.1",
    "typescript": "^4.2.2"
  }
}
```

Perfect, our project is ready, I can start writing some test. I'm going to start small by adding two test, one that goes to google.com and confirms the page title is 'Google' and another that assert that when google.com is loaded succesfully, a 200 OK Http status should be the response status code.

Add a new file under the test folder, call it google.ts and open it up on your favorite editor, I recommend vscode. Copy and paste the following code.

```javascript
import { Browser, Page } from 'playwright/types/types';
import { expect } from 'chai';
import { chromium } from 'playwright';
import { describe } from 'mocha';

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
    context('page title', function () {
        it('Should be equal to Google', async () => {
            await page.goto('https://www.google.com');
            const pageTitle = await page.title();
            expect(pageTitle).to.equal('Google')
        });
    });
});

describe('loading google.com successfully', function () {
    context('response status code', function () {
        it('Should be 200 OK', async () => {
            const response = await page.goto('https://www.google.com');
            expect(response?.status()).to.equal(200);
        });
    });
});
```

The code above imports the required node packages, then playwright's browser and page objects are configured to run before and after each test, then finally at the end, you have the two test. Like I said before one test will assert the title of the page and the other test will aserrt the response code. To execute these tests run the following command.

```text
npm test
```

The command should output the following result. 

```text
> e2eplaywright@1.0.0 test C:\Users\Yunier\Documents\playwright
> mocha -r ts-node/register 'test/*.ts' --recursive --reporter mocha-junit-reporter --timeout 60000 --exit
```

Playwright run headless by default, if you want to see it execute the tests on the browser, then set the headless property to false on the launch() method.

```javascript
let browser: Browser;
before(async () => {
    browser = await chromium.launch({headless: false});
});
```

For a complete list of available launch options, see the [playwright api docs](https://playwright.dev/docs/api/class-playwright).

Awesome. A foundation for building end-to-end test with playwright has been established. More test can now be added. If you are building tests that deal with data input, then I suggest installing [faker.js](https://github.com/marak/Faker.js/), it is another node libary used to create fake data such as people, address, number, text and so on. Learn more about playwright by reading the [docs](https://playwright.dev/docs/intro).

