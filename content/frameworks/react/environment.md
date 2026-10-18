# React Environment

## Usage

### Start dev environment without opening url in browser

- Add "BROWSER=none" to start-script.

```json
"scripts": {
  "start": "BROWSER=none react-scripts start",
  "build": "react-scripts build",
  "test": "react-scripts test",
  "eject": "react-scripts eject"
},
```

### Use environment variables

- Create file "./env" in the root.

```text
.env
package.json
```

- Place your variable with REACT_APP_ prefix.

```text
REACT_APP_PASSWORD=123
```

- Start your development environment.

```text
npm start
```

- Paste your variable into your code.

```js
console.log(process.env.REACT_APP_PASSWORD)
```
