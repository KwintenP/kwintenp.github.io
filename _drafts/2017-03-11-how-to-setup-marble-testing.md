---
layout: post
cover: false
title: How to setup marble testing
date:   2017-02-15
subclass: 'post'
categories: 'casper'
published: false
disqus: true
---

In an earlier blogpost, I showed you guys how to do client side filtering with streams (<a href="http://blog.kwintenp.com/client-side-filtering-with-streams/" target="_blank">here</a>). I tried to show you how you could use marble diagrams to draw out how the data will flow in your streams. Turns out that drawing your marble diagrams up front can help you a lot in testing your code as well. Using the marble diagram testing provided by RxJS, we can easily test the code we've written in the previous post. Let's see how.

### Setting up the marble diagram testing

The steps to set this up are really easy. First we need to copy two files from the RxJS source code into our own codebase. This is the `marble-testing.ts` and `test-helper.ts` file which you can find <a href="https://github.com/ReactiveX/rxjs/tree/master/spec/helpers" target="_blank">here</a>.
The next thing you need to do is import these files in a test where you want to use the marble testing.

```typescript
import "./helpers/test-helper.ts";
// I'll come back to these imports later
import { hot, cold, expectObservable, expectSubscriptions } from './helpers/marble-testing';
```

That's it, you are ready to start testing!

### Example

The marble diagram for the example looks like this:

![marble-diagram](https://www.dropbox.com/s/zhj0xvz6d5e84m4/Screenshot%202017-03-04%2016.12.24.png?raw=1)

We have a stream containing the characters and one containing a value to filter the characters based on the gender. We use the `combineLatest` operator to create a new stream which hold the filtered characters. The code to create this stream based on the two input streams looks like this:

```typescript
public createFilterCharacters(
        filter$: Observable<string>,
        characters$: Observable<StarWarsCharacter[]>) {
  return characters$.combineLatest(
    filter$, (characters: StarWarsCharacter[], filter: string) => {
      if (filter === 'All') {
        return characters;
      }
      return characters.filter(
            (character: StarWarsCharacter) =>
              character.gender.toLowerCase() === filter.toLowerCase()
      );
  });
}
```

#### Testing without marble diagrams
Trying to test this code without using marble diagram testing is quite verbose. First of all, we would need to create two streams ourselves to mock the character and gender filter streams. Then we would need to feed them to the method and take back the resulting stream. In our test, we would have to subscribe ourselves to this stream to check if the resulting next events are the ones we expect in the order we expect them. 
Let's take a look at the code:

```typescript
 it('on createFilterCharacters without marble testing', () => {
    // create a characters$ stream
    const characters$ = Observable.of([obiWan, c3po, leia]);
    // create a gender$ stream which is used to filter
    const gender$ = new BehaviorSubject<string>('All');


    let times = 0;
    // Feed the two streams to the method and subscribe to the result
    component.createFilterCharacters(gender$, characters$).subscribe(
      (val) => {
        // Based on the number of values that have passed here
        // check the value to see if it is what we expect
        if (times === 0) {
          expect(val).toEqual([obiWan, c3po, leia]);
          times++;
        } else if (times === 1) {
          expect(val).toEqual([obiWan]);
          times++;
        } else if (times === 2) {
          expect(val).toEqual([c3po]);
          times++;
        } else if (times === 3) {
          expect(val).toEqual([leia]);
          times++;
        }
      }
    );

    // pass new values to the gender subject to emulate the
    // gender filter change
    gender$.next("Male");
    gender$.next("N/A");
    gender$.next("Female");
  });
```

#### Testing with marble testing
We can write this a lot easier using marble diagram testing. To do this, we need to define ASCII marble diagrams and create observables from them. We can define teh character stream like this:

```typescript
// Here we create an ASCII marble diagram that 
// represents our characters stream. Since this
// is a backend call in real life, this will 
// first take some time before a value is ready.
// We represent this by using the '-'. It will take
// 4 ticks or '-' before the result arrives. We
// define the result with a c here and close with a
// |. This denotes that the stream completes.
const charactersASCII = "----c|";
// We define an object that represents the values
// in the stream above. We used the c to denote 
// a 'next' event and we use the same c in the 
// object below to show the value.
const charactersValues = {c: [obiWan, c3po, leia]};

// The ASCII and the values above aren't streams
// of course. And our method is expecting a stream.
// Using the 'cold' helper method from the 
// marble-testing, we can create a stream from
// the ASCII and the values.
const characters$ = cold(charactersASCII, charactersValues);
```

 Let's take a look at the code.

```typescript
describe('component: ClientSideFilterComponent', () => {
  it('on createFilterCharacters', () => {
    // we define a few values where the key will be used later
    // on to denote an observable value
    const values = {a: 1, b: 2, c: 3, d: 4};
    // using this cold method we imported above we can create
    // cold stream likes this
    const a = cold(' a-----b-----c----|', values)
    const asub = ( '^-----------------!')
    const b = cold('---------d----------|', values)
    const bsub = '^-------------------!'
    const expected = '-a-----b-d---c------|'

    expectObservable(a.merge(b).take(5)).toBe(expected, values);
    expectSubscriptions(a.subscriptions).toBe(asub);
    expectSubscriptions(b.subscriptions).toBe(bsub);
  });
});

```