---
title: React hooks - I
date: '2019-03-19'
---

Hooks are probably the most talked about feature in the react community in recent times. They made their
way into react [16.8](https://github.com/facebook/react/releases/tag/v16.8.0). So I thought it would be fun to learn and write something a bit non-trivial with 
them, for instance fetching data from a server API and displaying them.

But what are hooks? They are just special functions which let you use react features. 

```javascript
// Traditional class approach
import React, { Component } from "react";
import axios from "axios";

class App extends Component {
  state = {
    flights: null,
    error: null
  };
  componentDidMount() {
    axios
      .get("https://api.spacexdata.com/v2/launches")
      .then(response => {
        this.setState({ flights: response.data });
      })
      .catch(error => {
        // Not to be done in production ...
        console.log(error)
      });
  }

  renderList = flight => (
    <li key={flight.flight_number}>
      <p>Flight Number: {flight.flight_number}</p>
      <p>Mission name: {flight.mission_name}</p>
    </li>
  );

  render() {
    const { flights, error } = this.state;

    if (error) {
      return <div> Error! </div>;
    }

    if (!flights) {
      return <div> Fetching... </div>;
    }

    return <ul>{flights.map(flight => this.renderList(flight))}</ul>;
  }
}

export default App;
```

In the traditional approach, you would use a class, and define the state inside it, and use life cycle
methods to fetch data. Hence, classes were the preferred way to make components which fetch data, as you could store
them in as a state variable. 

However using hooks, we can write functional components which can fetch, and load data, without using any
life cycle methods (Bye Bye componentDidMount ...). Let's start the refactoring.
* Make the component a functional component
```js
const App = () => {} 
```
* Remove the life cycle hooks!
* Remove components from the import list, and import useState and useEffect from react.
```js
import React, { useState, useEffect } from "react";
```
* [useState](https://reactjs.org/docs/hooks-state.html) is a react hook which let's us use state from functional components (Cannot call functional
components stateless anymore!)
* Invoking useState, returns an array with two elements. The first element is the state, and the second
is a function, which can be used to change it. Let's define an initial state, and invoke useState with the
initial state passed as an argument.
```js
const initState = { flights: null };
const [data, setFlights] = useState(initState);
```
* Think of data as this.state.data, and setFlights as this.setState() function.
* In order to make ajax calls (one form of side effect), react provides the [useEffect](https://reactjs.org/docs/hooks-effect.html) hook. UseEffect takes two arguments, 
a function, and a value which react checks, before running the effect again (important to have a value for this, else
you will have an infinite render). We can make the ajax call inside the function.

```js
useEffect(
    () => {
      axios
        .get("https://api.spacexdata.com/v2/launches")
        .then(response => {
          setFlights({ flights: response.data });
        })
        .catch(error => {
          console.log(error);
        });
    },
    [initState]
  );
```
* We invoke the second element of the array returned, which is a function, which can be used to update
the state.
```js
setFlights({ flights: response.data });
``` 
* Everything else remains the same. the final hooks code looks as follows:

```js
// Trying react hooks
import React, { useState, useEffect } from "react";
import axios from "axios";
import "./App.css";

const App = () => {
  const initState = { flights: null };
  const [data, setFlights] = useState(initState);

  useEffect(
    () => {
      axios
        .get("https://api.spacexdata.com/v2/launches")
        .then(response => {
          setFlights({ flights: response.data });
        })
        .catch(error => {
          console.log(error);
        });
    },
    [initState]
  );
  if (!data.flights) {
    return <div> Fetching... </div>;
  }
  return <ul>{data.flights.map(flight => renderList(flight))}</ul>;
};

const renderList = flight => (
  <li key={flight.flight_number}>
    <p>Flight Number: {flight.flight_number}</p>
    <p>Mission name: {flight.mission_name}</p>
  </li>
);

export default App;
```
* Fewer lines of code, and cleaner compared to the class method.
