## In this setup, we'll use the built-in http module for the HTTP server and the ws library for the WebSocket server.
  The ws library allows you to attach a WebSocket server to an existing HTTP server instance. This means both HTTP requests and WebSocket upgrade requests will come through the same port, and your application will differentiate and handle them accordingly.
### How it works:
 - An http.Server instance is created to handle standard HTTP requests.
 - A ws.Server instance is created and configured to use the same http.Server instance.
 - When a client sends an HTTP request, the http.Server's request listener processes it.
 - When a client sends a WebSocket "upgrade" request, the ws.Server intercepts it from the http.Server and handles the WebSocket handshake and subsequent communication.
 ```typescript
// server.js
const express = require('express');
const http = require('http'); // Still need http for the underlying server
const WebSocket = require('ws');
const path = require('path');
const fs = require('fs');

const PORT = 8080;

// 1. Create an Express application
const app = express();

// Serve static files (like index.html) from the current directory
app.use(express.static(__dirname));

// Define HTTP routes using Express
app.get('/', (req, res) => {
    console.log(`HTTP Request received: ${req.method} ${req.url}`);
    // Express's express.static middleware will handle serving index.html
    // For direct route, we can send a file or render a template
    res.sendFile(path.join(__dirname, 'index.html'));
});

app.get('/api/hello', (req, res) => {
    console.log(`HTTP Request received: ${req.method} ${req.url}`);
    // Simple API endpoint
    res.json({ message: 'Hello from HTTP API (via Express)!' });
});

// Create the underlying HTTP server from the Express app
const httpServer = http.createServer(app);

// 2. Create a WebSocket server and attach it to the HTTP server
const wss = new WebSocket.Server({ server: httpServer });

wss.on('connection', ws => {
    console.log('WebSocket client connected');

    ws.on('message', message => {
        const msgStr = message.toString(); // Convert Buffer to String
        console.log(`Received WebSocket message: ${msgStr}`);

        // Echo the message back to the client
        ws.send(`Server received: ${msgStr}`);

        // Broadcast to all connected clients (except the sender)
        wss.clients.forEach(client => {
            if (client !== ws && client.readyState === WebSocket.OPEN) {
                client.send(`Broadcast from server: ${msgStr}`);
            }
        });
    });

    ws.on('close', () => {
        console.log('WebSocket client disconnected');
    });

    ws.on('error', error => {
        console.error('WebSocket error:', error);
    });

    // Send a welcome message to the new client
    ws.send('Welcome to the WebSocket server!');
});

// Start the HTTP server
httpServer.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
    console.log(`WebSocket server also running on ws://localhost:${PORT}`);
});
```

- client side component
```typescript
       function App() {
           const [httpResponse, setHttpResponse] = useState('');
           const [messages, setMessages] = useState([]);
           const [messageInput, setMessageInput] = useState('');
           const wsRef = useRef(null); // Use useRef to persist WebSocket instance

           const addMessage = (text, type) => {
               setMessages(prevMessages => [...prevMessages, { text, type, id: Date.now() }]);
           };

           useEffect(() => {
               // Initialize WebSocket connection
               if (!wsRef.current || wsRef.current.readyState === WebSocket.CLOSED) {
                   const ws = new WebSocket('ws://localhost:8080');

                   ws.onopen = () => {
                       addMessage('Connected to WebSocket server!', 'system');
                       console.log('WebSocket connection opened.');
                   };

                   ws.onmessage = event => {
                       addMessage(`Received: ${event.data}`, 'server');
                       console.log('WebSocket message received:', event.data);
                   };

                   ws.onclose = () => {
                       addMessage('Disconnected from WebSocket server.', 'system');
                       console.log('WebSocket connection closed.');
                   };

                   ws.onerror = error => {
                       addMessage('WebSocket error: ' + error.message, 'error');
                       console.error('WebSocket error:', error);
                   };

                   wsRef.current = ws;
               }

               // Clean up WebSocket connection on component unmount
               return () => {
                   if (wsRef.current && wsRef.current.readyState === WebSocket.OPEN) {
                       wsRef.current.close();
                   }
               };
           }, []); // Empty dependency array means this runs once on mount

           const sendMessage = () => {
               const ws = wsRef.current;
               if (ws && ws.readyState === WebSocket.OPEN && messageInput) {
                   ws.send(messageInput);
                   addMessage(`Sent: ${messageInput}`, 'client');
                   setMessageInput('');
               } else if (ws && ws.readyState !== WebSocket.OPEN) {
                   addMessage('WebSocket is not connected. Please refresh or check server.', 'error');
               }
           };

           const fetchApi = async () => {
               try {
                   const response = await fetch('/api/hello');
                   const data = await response.json();
                   setHttpResponse(JSON.stringify(data, null, 2));
               } catch (error) {
                   setHttpResponse('Error fetching API: ' + error.message);
                   console.error('Error fetching API:', error);
               }
           };

           return (
               <div>
                   <h1>HTTP & WebSocket Demo (React)</h1>

                   <h2>HTTP API Test</h2>
                   <button onClick={fetchApi}>Fetch HTTP API</button>
                   <pre className="api-response">{httpResponse}</pre>

                   <h2>WebSocket Chat</h2>
                   <div id="messages">
                       {messages.map(msg => (
                           <p key={msg.id} className={`message ${msg.type}`}>
                               {msg.text}
                           </p>
                       ))}
                   </div>
                   <input
                       type="text"
                       id="messageInput"
                       placeholder="Type a message..."
                       value={messageInput}
                       onChange={(e) => setMessageInput(e.target.value)}
                       onKeyPress={(e) => {
                           if (e.key === 'Enter') {
                               sendMessage();
                           }
                       }}
                   />
                   <button onClick={sendMessage}>Send</button>
               </div>
           );
       }
```