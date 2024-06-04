https://stackoverflow.com/questions/66151248/do-react-hooks-really-have-to-start-with-use

React hook definition according to React docs:

> A custom Hook is a JavaScript function whose name starts with ”use” and that may call other Hooks.

A perfect definition of a custom hook would be (notice the removal of "use" prefix and "may"):

> A custom Hook is a JavaScript function that calls other Hooks.

So we could distinguish between helper functions and custom hooks.

But, we can't tell if certain functions are actually using hooks (we do know in runtime). Thats why we use static code analyzing tools (like eslint) where we analyze text (lexical) and not meaning (semantics).

>*This convention guarantees that you can always look at a component and know where its state, Effects, and other React features might “hide”. (Reusing Logic with Custom Hooks)*

>*Should all functions called during rendering start with the use prefix? No. Functions that don’t call Hooks don’t need to be Hooks. (Reusing Logic with Custom Hooks)*

>*... this convention is very important. Without it, we wouldn’t be able to automatically check for violations of Rules of Hooks because we couldn’t tell if a certain function contains calls to Hooks inside of it. (Legacy docs)*

Hence:
```javascript
// #1 a function
// CAN'T BREAK ANYTHING

function double(nb) {
  return nb * 2;
}

// #2 Still a function, does not use hooks
// CAN'T BREAK ANYTHING
function useDouble(nb) {
  return nb * 2;
}

// #3 a custom hook because hooks are used 
// CAN BREAK, RULES OF HOOKS
function useDouble(nb) {
  const [state, setState] = useState(nb);
  const doubleState = (n) => setState(n*2);
  return [state,doubleState];
}
```

>Is naming hooks usexxxxx just a convention.

Yes, to help static analyzer to warn for errors.

>Can it really break something somehow?

Example #2 can't break your application since it just a "helper function" that not violating Rules of Hooks, although there will be a warning.

>Could you show an example?
```javascript
// #2 from above
function useDouble(nb) { return nb * 2; }

// <WithUseDouble/>
function WithUseDouble() {
  // A warning violating Rules of Hooks 
  // but useDouble is actually a "helper function" with "wrong" naming
  // WON'T break anything
  if (true) {
    return <h1>8 times 2 equals {useDouble(8)} (with useDouble hook)</h1>
  }
  
  return null;
}
```