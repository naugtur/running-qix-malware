# You too can run malware from NPM (I mean without consequences)

Phishing NPM package authors continues, unsurprisingly. 
The stakes are not high enough to switch from phishing to anything more advanced (like https://xkcd.com/538/) but seeing article blurbs say "Supply chain Attack" next to "These packages generally receive 2-3 billion downloads per week." might finally be enough to make an impression, one hopes.

This is not a detailed analysis of the attack, there's plenty of that already. If you're looking for one, visit our friends at https://socket.dev/blog/npm-author-qix-compromised-in-major-supply-chain-attack


Instead, let's look at how you could have a compromised dependency like that get into your app and be stopped.

One of the compromised packages was `is-arrayish` and I'll use that as an example going forward.

## What the malware does

So if an app uses `is-arrayish` in the browser, it will override `fetch`, `XMLHttpRequest` and `window.ethereum.request` and whenever it finds a transaction being sent, it'll replace the target address with one of the malware author's addresses that looks most alike.

I won't go into this either, but you can take a look at the summary of "donations" some other friends linked to here: https://intel.arkm.com/explorer/entity/61fbc095-f19b-479d-a037-5469aba332ab

Pretty low impact for an attack this big. Some of it seems to be people mocking the malware author with worthless transfers.


## Let's see it in action

Say we have an app. 
The app allows the user to send a meaningless transaction to themselves. Don't expect it to make sense.
It also uses `is-arrayish` because otherwise we'd have nothing to demo.

```js
const isArrayish = require("is-arrayish");

const button = document.createElement("button");
button.textContent = "Send ETH Transaction";
document.body.appendChild(button);

button.addEventListener("click", async () => {
  const accounts = await window.ethereum.request({
    method: "eth_requestAccounts",
  });
  if (!isArrayish(accounts)) {
    throw new Error("Accounts response must be array-like");
  }
  const myAddr = accounts[0];

  const txHash = await window.ethereum.request({
    method: "eth_sendTransaction",
    params: [
      {
        value: "0x5af3107a4000",
        from: myAddr,
        to: myAddr,
      },
    ],
  });
  console.log("Transaction sent:", txHash);
});
```

This is what it looks like:

![](./img/scr1.png)

Now **after you update `is-arrayish` to 0.3.3** and rebuild the project, you might notice a slight difference.

![](./img/scr2.png)


## Enter LavaMoat

If your project was set up with LavaMoat, you'd be using a policy to decide which package is allowed access to what. [More about policies in the guide](https://lavamoat.github.io/guides/policy/)

With LavaMoat, all `is-arrayish` can do is fail:

![](./img/scr3.png)

```
TypeError: Cannot define property fetch, object is not extensible
```

*BTW, If the malware was written a little better to avoid detection and fail silently, the functionality of the app would be fully restored.*

## LavaMoat Webpack Plugin

To protect the project, [@lavamoat/webpack](https://www.npmjs.com/package/@lavamoat/webpack) was used.

In short, what it does is: it puts modules from every dependency in a separate lexical global context that we call `Compartment` and only allows access to globals that the policy lists. It also controls which packages can import which other packages. 

If the project dependency gets updated to contain malicious code, the policy will not allow it to access any globals or imports it didn't use before. 

Read more in the [official guide](https://lavamoat.github.io/guides/webpack/)

If you prefer watching videos, there's some here [https://naugtur.pl/flix](https://naugtur.pl/flix). 

