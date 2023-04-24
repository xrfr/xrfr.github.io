---
title: Exporting Whatsapp Groups
feed: show
tags: hacking
---

# Problem

Recently, I had the need to get the contact information for everyone in one of my Whatsapp groups.
I tried to use the inbuilt _export chat_ feature, but found that it doesn't give you contacts.
There is a [script](https://gist.github.com/shaneapen/3406477b9f946855d02e3f33ec121975) I found that promises to do the task, but it also seems to fail.
It seems like Whatsapp used to keep phone numbers somewhere in the HTML document when opening the modal showing the group's members, but no longer does.
Now you have to navigate to each person in the group, open their contact page, then click on their name to open a sidebar that has their phone number.

# Solution

Tsk, tsk, Facebook, for trying to keep your users' data hostage.
While time consuming for us to do, since it's a simple sequence of user actions, we can automate it using our own javascript.
We can collect the contact of every group member and make a proper CSV export.


## Preamble

First, let's lay out some convenience functions to select tags on the DOM.
The first two, `document.querySelector` and `document.querySelectorAll` should be familiar, they select DOM elements based on CSS path.
The last one, `document.evaluate` is a way to evaluate *XPath* queries, which lets us do things that we couldn't with just CSS paths, like selecting a tag based on it's text content, as used here.

```js
select = (s, doc) => (doc ? doc : document).querySelector(s);
selectAll = (s, doc) => Array.from((doc ? doc : document).querySelectorAll(s));
getTagByText = (tag, text) => document.evaluate(`//${tag}[contains(text(),'${text}')]`, document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
```

When clicking on DOM elements, it takes time for the page to interpret the click and load the UI.
To get around this, we can use this function `waitFor`, which retries a function until it gets a truthy response, using progressively longer sleep timeouts in between.

```js
sleep = ms => new Promise((resolve, reject) => setTimeout(resolve, ms));
waitFor = (f, args = [], retries = 10) => new Promise((resolve, reject) => {
  let res = f(...args);
  if (res) {
    resolve(res);
  } else if (retries >= 0) {
    resolve(sleep(2000 / retries).then(() => waitFor(f, args, retries - 1)));
  } else {
    reject(new Error(`Failed to get: ${f.name}(${args})`));
  }
});
```

We define some common variables and a function to add a new contact to a `contacts` array.

```js
index = 0;
numMembers = null;
groupName = null;
el = null;
contacts = [];

addContact = (name, number) => {
  contacts.push({name, number});
  let last = contacts[contacts.length - 1];
  console.log(`(${index + 1}/${numMembers}) ${last.name}: ${last.number}`);
  // increment the index for next time
  index++;
}
```

## Simulate User Actions

Now the main content.
This function will go through the sequences of actions to get the contact info for a single group member.
We utilize `await/async` to make each step asynchronous.
There are some special blocks that are only run on the first run to pick up information about the group.

```js
getNextContact = () => sleep(0).then(async () => {
  // on first iteration, pick up group name
  if (!index) {
    groupName = select('div[data-testid="conversation-panel-wrapper"] header div:nth-child(2) span').title;
  }

  // click on group name in header
  (await waitFor(select, ['div[data-testid="conversation-panel-wrapper"] header div:nth-child(2)'])).click();

  // on first iteration, get numMembers
  if (!index) {
    el = await waitFor(select, ['div[data-testid="drawer-right"] section>div:nth-child(1)>div:nth-child(1)>div:nth-child(3)>span button']);
    numMembers = parseInt(el.textContent.split(' ')[0]);
  }

  // click 'View all <...>' to open modal
  (await waitFor(getTagByText, ['div', 'View all'])).click();
```

How to iterate over all group members?
We have to open the members panel to view the entire list, but scraping it is not so simple.
This is because it's implemented as an _infinite list_, and only 20 or so items are shown on the page.

We trick the page into showing us the entire list by finding the correct `div` that the list is using as it's container, and artificially set the height to a crazy high value.
Then we wait until all the users are loaded in the modal.

```js
  // resize the modal to load all the users into divs
  // getting past the infinite list
  el = await waitFor(select, ['div[data-animate-modal-body] div']);
  el.style.height = '50000px';

  // get all members in the group, wait until all are loaded
  let members = await waitFor(
    p => selectAll(p).length == numMembers ? selectAll(p) : false,
    ['div[data-animate-modal-body] div.zoWT4']);
```

If the user is not a known contact, their number and sometimes their name is already given in the list.
We can save some time and collect those without going to their contact page.

```js
  let name = members[index].outerText;
  // if not a known contact, get number and move on
  if (name == 'You') {
    index++;
    return getNextContact();
  } else if (name.startsWith('+')) {
    while (name.startsWith('+')) {
      let children = members[index].parentElement.parentElement.childNodes;
      addContact(select('div:nth-child(2)', children[1]).outerText, name);
      name = members[index].outerText
    }
    return getNextContact();
  }
```

For known contacts, open their contact page, get their information, and return to the group page by clicking the group name in the list of common groups.
This is needed because trying to programmatically click on the groups in the left sidebar doesn't work for some reason.

```js
  // open context menu for user
  el = (await waitFor(() => (
    getTagByText('div', 'Message ') || 
    (members[index].parentElement?.parentElement?.parentElement).click())));

  // open chat with user
  el.click()

  // open info for user
  let sec = await waitFor(() => (
    select('div[data-testid="contact-info-drawer"] section') ||
    select('div[data-testid="conversation-panel-wrapper"] header>div:nth-child(1)').click()));

  // get name and phone number
  let el1 = await waitFor(select, [':scope >div:nth-child(1)>div:nth-child(2)>h2:nth-child(1)', sec]);
  let el2 = await waitFor(select, [':scope >div:nth-child(1)>div:nth-child(2)>div:nth-child(2)', sec]);
  if (!el1) {
    // business accounts
    el1 = select(':scope >div:nth-child(1)>div:nth-child(3)>div:nth-child(1)>div:nth-child(1)', sec);
    el2 = select(':scope >div:nth-child(1)>div:nth-child(3)>div:nth-child(1)>div:nth-child(2)', sec);
  }

  if (el2 && el2.outerText.startsWith('~')) {
    addContact(el2.outerText, el1.outerText);
  } else if (el2) {
    addContact(el1.outerText, el2.outerText);
  } else {
    addContact('', el1.outerText);
  }

  // find group in list of common groups, and click it
  el = await waitFor(select, [`div[data-testid="contact-info-drawer"] section div[data-testid="cell-frame-container"] .zoWT4>span[title="${groupName}"]`])
  el.parentElement.parentElement.parentElement.parentElement.click();
```

Once we get to the end, we simply call the function again until we exhaust all the group members.

```js
}).then(async () => {
  if (index <= numMembers - 1) {
    return getNextContact();
  } else {
    downloadContacts();
    return;
  }
}).catch(err => console.error(err));
```

## Export CSV

Finally, here is the function to create a CSV from the `contacts` array and download it.

```js
downloadContacts = () => {
  let fn = `${groupName}.csv`
  console.log(`Downloading ${fn}`);

  var csv = 'Name, Number\n';  
  contacts.forEach(c => {
    csv += `${c.name}, ${c.number}\n`
  });

  var hiddenElement = document.createElement('a');
  hiddenElement.href = 'data:text/csv;charset=utf-8,' + encodeURI(csv);
  hiddenElement.target = '_blank';
  hiddenElement.download = fn;
  hiddenElement.click();
}
```
