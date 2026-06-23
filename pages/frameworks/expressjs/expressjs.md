# Express (ExpressJS)

Minimal and flexible Node.js web application framework providing robust features for building web and mobile applications.

## Installation

```bash
npm install express
```

## Basic Setup

```javascript
const express = require('express');
const app = express();

// Middleware
app.use(express.json());
app.use(express.static('public'));

// Routes
app.get('/', (req, res) => {
  res.send('Hello World!');
});

// Start server
app.listen(3000, () => console.log('Server running on port 3000'));
```

## Routing

### Basic Routes

```javascript
app.get('/path', (req, res) => {});     // GET request
app.post('/path', (req, res) => {});    // POST request
app.put('/path', (req, res) => {});     // PUT request
app.delete('/path', (req, res) => {});  // DELETE request
app.patch('/path', (req, res) => {});   // PATCH request
```

### Route Parameters

```javascript
app.get('/users/:id', (req, res) => {
  console.log(req.params.id);
});

app.get('/posts/:id/comments/:commentId', (req, res) => {
  console.log(req.params.id, req.params.commentId);
});
```

### Query Strings

```javascript
app.get('/search', (req, res) => {
  console.log(req.query.q); // ?q=hello
});
```

## Middleware

Functions that execute during request lifecycle before route handlers.

```javascript
// Global middleware
app.use((req, res, next) => {
  console.log('Request logged');
  next(); // Pass to next middleware
});

// Route-specific middleware
app.get('/protected', (req, res, next) => {
  if (req.headers.authorization) {
    next();
  } else {
    res.status(401).send('Unauthorized');
  }
}, (req, res) => {
  res.send('Protected route');
});

// Error handling middleware (must be last)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).send('Something went wrong');
});
```

## Request & Response

```javascript
// Request properties
req.params      // Route parameters
req.query       // Query string parameters
req.body        // Request body (requires parsing middleware)
req.headers     // Request headers
req.method      // HTTP method
req.path        // Request path
req.cookies     // Cookies

// Response methods
res.send('text')           // Send response
res.json({ key: 'value' }) // Send JSON
res.sendFile('/path')      // Send file
res.status(200)            // Set status code
res.setHeader('key', 'value') // Set header
res.redirect('/path')      // Redirect
```

## Static Files

```javascript
app.use(express.static('public'));
app.use('/static', express.static('public'));
```

## Built-in Middleware

```javascript
express.json()           // Parse JSON bodies
express.urlencoded()     // Parse URL-encoded bodies
express.static()         // Serve static files
express.text()           // Parse text bodies
```

## Common Patterns

### Error Handling

```javascript
try {
  // async operation
} catch (error) {
  next(error); // Pass to error handler
}
```

### Authentication Middleware

```javascript
const auth = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (token) next();
  else res.status(401).send('Unauthorized');
};

app.get('/admin', auth, (req, res) => {
  res.send('Admin area');
});
```

### CORS

```javascript
const cors = require('cors');
app.use(cors());
```

## Configuration

```javascript
// Environment variables
app.set('env', process.env.NODE_ENV || 'development');
app.set('port', process.env.PORT || 3000);

// Get settings
const port = app.get('port');
```
