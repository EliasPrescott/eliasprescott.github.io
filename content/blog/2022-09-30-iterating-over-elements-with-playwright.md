+++
title = "Iterating over Elements with Playwright"
date = 2022-09-30
+++

[Playwright](https://playwright.dev/) is a Microsoft library for automated web testing.
Playwright is designed to handle reactive web pages, which makes it ideal for testing modern web applications. I have worked with Playwright quite a bit recently, so I have found myself writing a lot of helper functions for it. The one helper function I always find myself needing is a way to iterate over elements.

#### Selecting Elements

If you are not familiar with writing Playwright scripts, it typically involves code like this:

```javascript
await page.click('#popupModal .closeBtn');

await page.fill('.someTextBox', 'Example value');
```

From this brief sample, you can see the basics of Playwright's API design. You call functions on a page object, allowing you to perform actions. To perform these actions, Playwright needs to know what element it should perform each action on. For this, the functions often take in a selector string as their first argument.

In the above example, both actions take in plain CSS selectors, which allow you to identify elements based on their tag name, ID, class, and other attributes. Playwright supports other selector formats, but CSS selectors are often the simplest and best way to locate elements.

Locating elements is so important to Playwright, there is even a method for saving a selector as an object. This is the [```page.locator()```](https://playwright.dev/docs/api/class-page#page-locator) method, which returns a [Locator](https://playwright.dev/docs/api/class-locator) object. After saving a Locator, you can perform many of the same actions on it as you could on the page object. Here is a basic example:

```javascript
let importantBtn = page.locator('#importantBtn');

await importantBtn.focus();

await importantBtn.click();
```

The reason I bring all this up is because Locator objects are going to be very important to our topic.

#### Why do we want to Select Multiple Elements?

Repetition of elements is extremely common on web pages. Most web frameworks make it very easy to repeat an element any number of times. This is great for writing web sites, but it can make the sites more difficult to test.

Whenever an element is repeated, all its attributes will also be repeated. This means that all its identifying information will be shared between every instance of the element. So, if I call ```page.click('button.actionBtn')```, and there are five elements that match that selector, which element should Playwright click? Playwright will try to click on the first match by default, but relying on this behavior can be dangerous. Whenever an element is repeated on a page, the number of repetitions is often variable. I may visit a page and see five action buttons, but someone else could visit the same page and see zero buttons. All manner of variables can affect how many elements will show on a web page, so it is vital to write Playwright tests that can handle that.

#### Handling Repetition

Let's get back on track and dive into more code. When asked to perform a single action on a locator that resolves to multiple elements, Playwright will choose the first element by default. If you want to click on an element other than the first match, there is a useful method on the Locator object for that:

```javascript
let btnLocator = page.locator('button.actionBtn');

// nth() uses a zero-based index
let secondButton = btnLocator.nth(1);
```

The ```locator.nth()``` function is useful for selecting individual elements, but it is extremely useful for iterating over elements. Whenever we combine it with ```locator.count()```, we can do things like this:

```javascript
// Go to our example site
await page.goto('https://www.wikipedia.org/');

// Create a locator that resolves to multiple elements
let languageLinkLocator = page.locator('.central-featured div a');

// Get the count of all the elements
let languageLinkCount = await languageLinkLocator.count();

// Iterate through the elements
for (let i = 0; i < languageLinkCount; i++) {
    
    // Get the current element using our index
    let currentLanguageLink = languageLinkLocator.nth(i);

    // Get the current href from the element
    let languageHref = await currentLanguageLink.getAttribute('href');

    // Select the strong element within our current element, and get its inner text
    let languageName = await currentLanguageLink.locator('strong').innerText();

    // Log the href and the language name we scraped
    console.log(languageHref, languageName);
}
```

This is the result I got from running this script:

```
//en.wikipedia.org/ English
//ja.wikipedia.org/ 日本語
//es.wikipedia.org/ Español
//ru.wikipedia.org/ Русский
//fr.wikipedia.org/ Français
//de.wikipedia.org/ Deutsch
//it.wikipedia.org/ Italiano
//zh.wikipedia.org/ 中文
//pt.wikipedia.org/ Português
//ar.wikipedia.org/ العربية
```

At the start of this, I said that I often wrote helper functions that do this, so let's do that now:

```javascript
async function forEachMatch<T>(locator: Locator, action: (currentLocator: Locator) => Promise<T>): Promise<T[]> {
    // Get the count of all the elements
    let locatorCount = await locator.count();

    // Prepare our list of return results
    let results: T[] = [];

    // Iterate through the elements
    for (let i = 0; i < locatorCount; i++) {
        
        // Get the current element using our index
        let currentLocator = locator.nth(i);

        // Run the user-given action for the current element
        let result = await action(currentLocator);

        // Store the result
        results.push(result);
    }

    return results;
}
```

The function signature is a little long, but this function is ideal for quickly scraping data from lists of elements. If you do not want to return anything from the action you call on each element, you could remove the function's return type and change the ```action``` parameter to return void. 

Here is an example usage of our function:

```javascript
test('Scraping Multiple Elements', async ({ page }) => {

    // Go to our example site
    await page.goto('https://www.wikipedia.org/');

    // Create a locator that resolves to multiple elements
    let languageLinkLocator = page.locator('.central-featured div a');

    // Pass in our Locator object along with a function for converting each iterated object into a value
    let languageInformation = await forEachMatch(languageLinkLocator, async languageLink => {
        
        // Inside this action function, we can focus solely on using the languageLink Locator object to scrape data or perform actions
        
        let href = await languageLink.getAttribute('href');

        let name = await languageLink.locator('strong').innerText();

        return [href, name];
    });

    console.log(languageInformation);
});
```

Here is the output:

```json
[
  [ "//en.wikipedia.org/", "English" ],
  [ "//ja.wikipedia.org/", "日本語" ],
  [ "//es.wikipedia.org/", "Español" ],
  [ "//ru.wikipedia.org/", "Русский" ],
  [ "//fr.wikipedia.org/", "Français" ],
  [ "//de.wikipedia.org/", "Deutsch" ],
  [ "//it.wikipedia.org/", "Italiano" ],
  [ "//zh.wikipedia.org/", "中文" ],
  [ "//pt.wikipedia.org/", "Português" ],
  [ "//ar.wikipedia.org/", "العربية" ]
]
```

Hopefully you can see the power of using Playwright to iterate through lists of elements. This is an extremely useful technique when using Playwright for web scraping, but it is also useful for writing more reliable front-end test scripts. Playwright includes all the necessary tools to deal with repeated elements, it just requires a little bit of knowledge and code to put it all together.