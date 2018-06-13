use `zone.js` to handle `native async/await` issue.
`native async/await` will use `native promise`, so `zone.js` can not control it's lifecycle.
Code like below will not work.

```ts
async function syncTest() {
  console.log('async Function');
}

async function asyncTest() {
  return new Promise((res, rej) => {
    setTimeout(res, 100);
  });
}

async function run(asyncMethod) {
  const zone = Zone.current.fork({
    name: 'zone'
  });
  zone.run(async () => {
    console.log('before run', Zone.current.name);
    await asyncMethod();
    console.log('after run', Zone.current.name);
  });
  console.log('outside', Zone.current.name);
}

run(syncTest);
run(asyncTest);
```

after await, the `Zone` will lost.

in this repo, I created a patched `zone.js` to handle this issue, it will not resolve all cases of `async/await`.
It will only work in some limited conditions.

1. the awaited method is sync operation, such as the `syncTest` above.
2. the awaited method is async operation, and inside the async operation, it will use `global.Promise or window.Promise`.

And the following case will still not work.

```ts
zone1.run(async() => {
  console.log('before run', Zone.current.name);
  await syncTest();
  console.log('after run', Zone.current.name);
});
zone2.run(async() => {
  console.log('before run', Zone.current.name);
  await syncTest();
  console.log('after run', Zone.current.name);
});
```
If we have two zone, and `await` sync operation, finally, with this solution, zone1 after run will also go into zone2.

But this solution maybe able to resolve some specified issue, such as we know we only use one zone.

Here is a running sample, https://stackblitz.com/edit/js-hdcq2u?file=index.js