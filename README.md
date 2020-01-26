# bs-rx
Bucklescript bindings for [rxjs v7(alpha)](https://github.com/ReactiveX/rxjs)  
Most functionality is available, while ajax / fetch / websocket apis are not yet done. Refer to [documentation](https://ambientlight.github.io/bs-rx) for existing coverage.

## Installation and Usage

```
npm install @ambientlight/bs-rx
```

Then add `@ambientlight/bs-rx` into `bs-dependencies` in your project `bsconfig.json`.


```reason
Rx.range(~start=1, ~count=200, ())
|> Rx.Operators.filter((x, _idx) => x mod 2 == 1)
|> Rx.Operators.map((x, _idx) => x + x)
|> Rx.Observable.subscribe(
  ~next=x=>Js.log(x)
)
```

## Examples

#### Map and flatten each letter to an Observable ticking every 1 second

```reason
Rx.of_([|"a", "b", "c"|])
|> Rx.Operators.mergeMap(`Observable((x, _idx) => 
  Rx.interval(~period=1000, ())
  |> Rx.Operators.map((i, _idx) => string_of_int(i) ++ x)), 
  ())
|> Rx.Observable.subscribe(~next=x=>Js.log(x));
```

#### Custom operator

Create an observable that never completes and repeats when browser is back online.

```reason
let repeatWhenOnline = source => 
  source
  |> Rx.Operators.takeUntil(Rx.fromEvent(~target=Webapi.Dom.window, ~eventName="offline"))
  |> Rx.Operators.repeatWhen(_notifier => Rx.fromEvent(~target=Webapi.Dom.window, ~eventName="online"));

let obs = Rx.of1("I'm online")
|> repeatWhenOnline
|> Rx.Observable.subscribe(~next=x=>Js.log(x));
```

Also, have a look at [OperatorTests](https://github.com/ambientlight/bs-rx/blob/master/__tests__/OperatorTests.re) for more usage examples.

## Testing

You may find marble testing handy to test your rxjs logic. Marble string syntax allow your to specify rxjs events(such as emissions, subscription points) over virtual time that progresses by frames(denoted by `-`). You can use it to express the expected behavior of your observable sequences as strings and compare them with `Rx.Observable.t('a)` instances you are testing. You need to initialize `TestScheduler.t` with a function that can perform deep comparison (such as `BsMocha.Assert.deep_equal`), then put your marble tests inside `ts |> TestScheduler.run(_r => ...)`. Asynchronous operators usually take `~scheduler` parameter, pass `TestScheduler.t` instance to them. The next example illustrates it, also you may want to refer to rxjs [marble diagrams documentation](https://rxjs-dev.firebaseapp.com/guide/testing/marble-testing).

```reason
test("timeInterval: should record the time interval between source elements", () => {
  let ts = TestScheduler.create(~assertDeepEqual=BsMocha.Assert.deep_equal);
  ts |> TestScheduler.run(_r => {
    // subscribe in 6th frame, 4 emissions: b, c, d, e
    let e1 = ts |> hot("--a--^b-c-----d--e--|");
    let e1subs =          [|"^--------------!"|];
    let expected =          "-w-x-----y--z--|";
    // expected values in w, x, y, z emissions
    let values = { "w": 1, "x": 2, "y": 6, "z": 3 };

    let result = e1
    |> HotObservable.asObservable
    |> Rx.Operators.timeInterval(~scheduler=ts|.TestScheduler.asScheduler, ())
    |> Rx.Operators.map((x, _idx) => x |. Rx.TimeInterval.intervalGet);

    ts |> expectObservable(result) |> toBeObservable(expected, ~values);
    ts |> expectSubscriptions(e1 |> HotObservable.subscriptions) |> toBeSubscriptions(e1subs);
    Expect.expect(true) 
  })
  |> Expect.toBe(true)
});
```

## Contributing

Any contribution is greatly appreciated. Feel free to reach out in issues for any questions or problem you ran into. Implementational inheritance is used to model inheritance used in rxjs, you may want to refer to [Implementation Inheritance](https://github.com/reasonml-community/bs-webapi-incubator#implementation-inheritance). 

```
git clone https://github.com/ambientlight/bs-rx.git
cd bs-rx
npm install
npm run build
npm run test
```

You can also build docs via bsdocs. If you have cloned this repo, the pushes to master should spin the github actions workflow that rebuild the github pages docs with workflow available at [deploy_docs.yml](https://github.com/ambientlight/bs-rx/blob/master/.github/workflows/deploy_docs.yml). (You will need to set `GH_PAGES_TOKEN` for github pages deployment to work).

If you want to generate docs in local make sure you have opam installed with ocaml version matching the ocaml version used in your `bs-platform` (`4.02.3+buckle-master` for bs-platform@5.2.1).

```
opam switch 4.02.3+buckle-master
```

For osx, you can use the npm installation of bsdoc, but for linux-based distros, you would need to build bsdoc from source for now.