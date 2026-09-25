# React Notes

## 1 - Components - reusable

A javascript function not returns markup it return JSX. JSX is also optional

## 2

### Variable

```javascript
const planet = 'world'
<div> Hello {planet} </div>
```

### Dynamic Attributes

```javascript
const src = "/react.svg"
<img src = {src} />
```

### Dynamic Styles

```javascript
const background = "red"
<div style ={{background}} /div>
```

## 3 - JS functions can return only ONE thing

```javascript
function App(){
  return (
    <>
      </Header>
      <Main/>
    </>
```

These empty component are called React Fragments

## 4 - Props

### Sending Value

```jsx
<Greetings text = {'Yo'}/>
```

### Using Value

```javascript
function Greetings(props){
                return <h1> {props.text}</h1> 
              }
```

## 5

U can pass anything as the props even other components as children props. Great for composition. Great for layout components

```jsx
<Parent>
  <Child/>
</Parent>
```

## 6

```jsx
<Component key={'1'} />

{ items.map((item,index) => (<div key = {index}> {item} </div>))}
```

use index if no unique key

## 7 - Rendering

DOM - Document Object Model (looks like tree)

Virtual DOM

Rendering Process

```text
StateChanged?(Update VDOM) -> Diffs (Identify what changed) -> Reconciliation with DOM
```

## 8 - Event Handling (Handling User Interactions)

```jsx
<button onClick = {handleClick} />
<input OnChange = {handleChange} />
<form onSubmit = {handleSubmit} />
```

```javascript
function RedAlert(){
  const handleClick = () => {
    alert('Alert!!!')
  }
  return (
    <button onClick={handleClick}> Click Me </button>
  )
}
```

## 9 - State as SnapShot

We have to use Special functions

```text
useState()
useReducer()
```

```javascript
function Likes()
{
  const [likes, setLikes] = useState(0)

  const handleClick = () => {
    setClicks(likes+1)
  }

  return ( <button onClick={handleClick}> Likes: {likes} </button>)
}
```

## 10 - Controlled Components

```javascript
function ControlledInput(){
  const [value, setValue] = useState('')
  return (
    <input value={value}
            onChange={(e) => setValue(e.target.value)} />
  )
}
```

## 11 - 5 Types of Hook

1- StateHook  
   ** useState()
   useReducer()

2- Context Hooks
   useContext()

3- Ref Hooks
   ** useRef()

4- Effects Hooks
   ** useEffect()

5- Performance
   useMemo()
   useCallback()

## 12 - Purity

Same Input should return same Output

Only return JSX, Dont Change stuff that existed before rendering

To prevent changing any variable while rendering we can use Strict Mode

```jsx
<StrictMode>
  <App/>
</StrictMode>
```

## 13- Effects are those where code reaches outside of React apps

Request side effects made in event Handler

```javascript
function handleSubmit(e)
{
  e.preventDefault()
  post('/api/register', {email, password})
}
```

## 14- If u can not run your effects in event handlers then u can run them using useEffect()

```javascript
useEffects(() => {
  fetchData().then(data => {})
}, [])
```

## 15 - If u want to get out from react and directly work with DOM element. To refrence an actual DOM element use Ref

```javascript
const ref = useRef()
<input ref={ref}/>
ref.current.focus()
```

## 16 - Context

If u want to pass the data through components (Like jump to where the data needs to go through)

### a) Create your context ->

```javascript
const AppContext = createContext
```

### b) Wrap

```jsx
<AppContext.Provider>
  <App/>
</AppContext.Provider>
```

### c) Put data on the Provider

```jsx
<AppContext.Provider value = "Hello">
```

### d) access the data

```javascript
function Title(){
    const text = useContext(AppContext)
    return <h1> {text} </h1>
  }
```

When the Context value changes, every component consuming it re-renders. Fast changing data in Context = performance problem.
For frequent global state updates, reach for Zustand or Redux instead.

## 17 - Portals

Like contexts but for components

Portals let u move react components into any HTML component u select.

Helpful for displaying modal, dropdowns, tooltips(Useful for the components where they can not be displayed properly due to their parents component style)

```jsx
<div>
  <p> I am in the parent div </p>
  {createPortal(
        <p> I am in the document body </p>
        document.body
  )}
</div>
```

## 18 - Suspense Component to show waiting or lazy loading

```javascript
const Component = lazy(
  () => import('./Component')
)

<Suspense fallback = {<Loading/>}>
  <Component/>
</Suspense>
```

## 19 - ErrorBoundry

```javascript
import {ErrorBoundry} from 'react-error-boundry'

function Fallback({erro}){
  return (
    <div role="alert">
      <p> No user provided </p>
      <pre> {error.message} </pre>
    </div>
  );
}

<ErorrBoundry FallBackComponent = {FallBack}>
  <App/>
</ErrorBoundry>
```

---

# Hooks

## 1 - UseState

### a) Manage Form Input

```javascript
const [value, setValue] = useState("")

const handleChange = (e) => {
  setValue(e.target.value)
}

<input type = "text" value ={value} onChange = {handleChange} /> 
```

### b) Show or hide components

```javascript
const[isVisible, setIsVisible] = useState(false)

<button onClick={() => setIsVisible(!isVisible)}>
  Toggle
</button>

<>
  {isVisible && <div> Content to Show/Hide </div>}
</>
```

### c) Dynamic Styles

```javascript
const[isActive, setIsActive] = useState(false)

<button className = {isActive ? 'active': 'inactive'} 
        onClick= {() => setIsActive(!Active)}
>
Click Me
</button>
```

### d) Counters

```javascript
const[count, setCount] = useState(0)

const increment = () => setCount(count+1)
const decrement = () => setCount(count-1)

<div> 
  <button onClick={decrement}> - </button>
  <button onClick={increment}> + </button>
</div>
```

## 2- Reducer

### a)

```javascript
const reducer = (state, action) => {
  switch(action){
    case 'increment':
      return state + 1
  }
}

const[count, dispatch] = useRedcuer(reducer, 0)

<button onClick={ () => dispatch('increment')} > Increment </button>
```

### b)

```javascript
const initialState = {
  email: '',
  password: '',
};

const reducer = (state, action) => {
  switch (action.type) {
    case 'SET_EMAIL':
      return {
        ...state,
        email: action.payload,
      };

    case 'SET_PASS':
      return {
        ...state,
        password: action.payload,
      };

    default:
      return state;
  }
};

const [state, dispatch] = useReducer(reducer, initialState);

<form>
  <input
    type="email"
    onChange={(e) => {
      dispatch({
        type: 'SET_EMAIL',
        payload: e.target.value,
      });
    }}
  />

  <input
    type="password"
    onChange={(e) => {
      dispatch({
        type: 'SET_PASS',
        payload: e.target.value,
      });
    }}
  />
</form>
```

### c)

```javascript
const gameReducer = (state, action) => {
  switch (action.type){
    case 'move':
      return {...state, position: state.position + action.distance}
    case 'score':
      return {...state, score: state.score + action.points}
    default:
      return state
  }
}

const[state, dispatch] = useReducer(gameReducer, {position:0, score:0})'

<>
  <p>Position: {state.position}, Score:{state.score}</p>
  <button onClick={() => dispatch({type:'move', distance:1})}> Move </button>
  <button onClick={() => dispatch({type:'score', points:10})}> Score </button>
</>
```

## 3 - useEffects

### a) Example 1
```javascript
const[count,setCount] = useState(0)

useEffects(() => {
document.title = "You clicked ${count} times"
}, [count])

<button onClick={() => setCount(count+1)}>
  Click me
</button>
```
Two kind of side effects Event-Based, Render-Based
We shall not use useEffect() in any of them.

a) Event-Base - U can make your code simpler by doing it as EventHandler
```javascript
<button onClick = {saveData}>
  Save
</button>
```
b) Use reactQuery or frameWork tool

### b) Example 2
Ideal for syncing your React code with browser APIs

```javascript
const ref useRef(null)

useEffect(() => {
  if(isPlaying){
    ref.current.play()
  } else {
    ref.current.pause()
  }
}, [isPlaying])

<video ref = {ref} src={src} loop playsInline />

```

### c) Example 3

```javascript

import { useEffect, useState } from 'react';

interface DemoProps {}

export default function Demo({}: DemoProps) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // The code that we want to run
    console.log('The count is:', count);

    // Optional return function
    return () => {
      console.log("I am being cleaned up!");
    }
  }, []); // The dependency array



  return (
    <div className='tutorial'>
      <h1>Count: {count}</h1>
      <button onClick={() => setCount(count - 1)}> Decrement </button>
      <button onClick={() => setCount(count + 1)}> Increment </button>
    </div>
  );
}

```

## 4 - useMemo

```javascript
import {useRef} from 'react';
import {initialItems} from './utils';

interface DemoProps {}

function Demo({}:DemoProps)
{
  const[count, setCount] = useState(0);
  const[items] = useState(initialItems);
  
  const selectedItem = useMemo (() => items.find((item) => item.isSelected), [items]);

  return (
    <div className='tutorial'>
      <h1> Count {count}</h1>
      <h1> Selected Item: {selectedItem?.id}</h1>
      <button onClick={() => setCount(count+1)}> Increment </button>
    </div>
  )
}

export default Demo;
```

## 5 - callBack
```javascript
import { useState } from 'react';

import { shuffle } from '@/utils';

import Search from './Search';

const allUsers = [
  'john',
  'alex',
  'george',
  'simon',
  'james',
];

interface DemoProps {}

export default function Demo({}: DemoProps) {
  const [users, setUsers] = useState(allUsers);

  const handleSearch = useCallBack((text: string) => {
    console.log(users[0]);
    const filteredUsers = allUsers.filter((user) =>
      user.includes(text),
    );
    setUsers(filteredUsers);
  }, [user]):

  return (
    <div className='tutorial'>
      <div className='align-center mb-2 flex'>
        <button onClick={() => setUsers(shuffle(allUsers))}>
          Shuffle
        </button>

        <Search onChange={handleSearch} />
      </div>

      <ul>
        {users.map((user) => (
          <li key={user}>{user}</li>
        ))}
      </ul>
    </div>
  );
}
```

## 6 - Custom Hooks
```javascript
import { useEffect, useState } from 'react';

function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(false);

  useEffect(() => {
    (async () => {
      try {
        setLoading(true);
        setError(false);

        const response = await fetch(url);

        if (!response.ok) {
          throw new Error('Failed to fetch data');
        }

        const data = await response.json();

        setData(data);
        setLoading(false);
      } catch (error) {
        setError(true);
        setLoading(false);
      }
    })();
  }, [url]);

  return { data, loading, error };
}

export default function Users() {
  const { data, loading, error } = useFetch(
    'https://jsonplaceholder.typicode.com/users'
  );

  if (loading) {
    return <p>Loading...</p>;
  }

  if (error) {
    return <p>Something went wrong</p>;
  }

  return (
    <div>
      <h1>Users</h1>

      {data?.map((user) => (
        <p key={user.id}>{user.name}</p>
      ))}
    </div>
  );
}

```

createBrowserRouter
