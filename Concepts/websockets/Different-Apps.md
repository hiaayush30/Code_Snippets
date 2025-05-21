## ws server and http server on different apps
### http-server

This Node.js application will serve as your RESTful API. It uses Express.js for routing and is written in TypeScript for better type safety and maintainability. It will listen on port 3000.
```typescript

// backend/http-api/src/server.ts
import express, { Request, Response } from 'express';
import cors from 'cors'; // Import cors for cross-origin requests

const app = express();
const HTTP_PORT = process.env.HTTP_PORT || 3000;

// Enable CORS for all origins (for development purposes)
// In a production environment, you should restrict this to your frontend's origin.
app.use(cors());
app.use(express.json()); // Enable JSON body parsing

// Basic route
app.get('/', (req: Request, res: Response) => {
    res.status(200).send('Hello from the HTTP API!');
});

// Example API endpoint
app.get('/api/data', (req: Request, res: Response) => {
    console.log('Received request for /api/data');
    res.status(200).json({
        message: 'Data fetched successfully from HTTP API (TypeScript)',
        timestamp: new Date().toISOString(),
        data: [
            { id: 1, name: 'Item A' },
            { id: 2, name: 'Item B' }
        ]
    });
});

// Start the server
app.listen(HTTP_PORT, () => {
    console.log(`HTTP Server running on http://localhost:${HTTP_PORT}`);
    console.log('Press Ctrl+C to stop');
});

```

### ws server
This Node.js application will run a dedicated WebSocket server using the ws library. It's also written in TypeScript and will listen on port 8081. It handles incoming messages and broadcasts them to all connected clients.
```typescript
// backend/websocket-api/src/server.ts
import { WebSocket, WebSocketServer } from 'ws';

const WS_PORT = process.env.WS_PORT || 8081;

// Create a WebSocket server instance
const wss = new WebSocketServer({ port: Number(WS_PORT) });

wss.on('connection', ws => {
    console.log('WebSocket client connected');

    ws.on('message', message => {
        const msgStr = message.toString(); // Convert Buffer to String
        console.log(`Received WebSocket message: ${msgStr}`);

        // Echo the message back to the sender
        ws.send(`Server received: ${msgStr}`);

        // Broadcast to all connected clients (including sender for simplicity,
        // but typically you might exclude the sender)
        wss.clients.forEach(client => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(`Broadcast from WS server: ${msgStr}`);
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
    ws.send('Welcome to the separate WebSocket server!');
});

console.log(`WebSocket Server running on ws://localhost:${WS_PORT}`);
console.log('Press Ctrl+C to stop');


```


### client side
This is a Next.js application that serves as the client-side interface. It uses React components to display data fetched from the HTTP API and to manage real-time communication via the WebSocket server. It will run on port 3000 by default (Next.js development server). We'll use fetch for HTTP requests and the native WebSocket API for WebSocket connections. Tailwind CSS is included for basic styling.

```typescript

// frontend/my-nextjs-app/app/page.tsx
'use client'; // This directive is required for client components in Next.js App Router

import React, { useState, useEffect, useRef } from 'react';

// Define the ports for your backend servers
const HTTP_API_URL = 'http://localhost:3000';
const WS_API_URL = 'ws://localhost:8081';

export default function Home() {
    const [httpResponse, setHttpResponse] = useState<string>('');
    const [wsMessages, setWsMessages] = useState<string[]>([]);
    const [messageInput, setMessageInput] = useState<string>('');
    const wsRef = useRef<WebSocket | null>(null); // Ref to hold the WebSocket instance
    const messagesEndRef = useRef<HTMLDivElement>(null); // Ref for auto-scrolling messages

    // Effect to connect to WebSocket on component mount
    useEffect(() => {
        // Only connect if not already connected or if connection was closed
        if (!wsRef.current || wsRef.current.readyState === WebSocket.CLOSED) {
            const ws = new WebSocket(WS_API_URL);

            ws.onopen = () => {
                console.log('WebSocket connection opened.');
                setWsMessages(prev => [...prev, 'System: Connected to WebSocket server!']);
            };

            ws.onmessage = (event) => {
                console.log('WebSocket message received:', event.data);
                setWsMessages(prev => [...prev, `Server: ${event.data}`]);
            };

            ws.onclose = () => {
                console.log('WebSocket connection closed.');
                setWsMessages(prev => [...prev, 'System: Disconnected from WebSocket server.']);
            };

            ws.onerror = (error) => {
                console.error('WebSocket error:', error);
                setWsMessages(prev => [...prev, `System: WebSocket error: ${error.message}`]);
            };

            wsRef.current = ws; // Store the WebSocket instance in the ref
        }

        // Cleanup function: close WebSocket when component unmounts
        return () => {
            if (wsRef.current && wsRef.current.readyState === WebSocket.OPEN) {
                wsRef.current.close();
            }
        };
    }, []); // Empty dependency array ensures this runs only once on mount

    // Effect to scroll to the bottom of the messages div
    useEffect(() => {
        messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
    }, [wsMessages]);

    // Function to fetch data from the HTTP API
    const fetchHttpData = async () => {
        try {
            const response = await fetch(`${HTTP_API_URL}/api/data`);
            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }
            const data = await response.json();
            setHttpResponse(JSON.stringify(data, null, 2)); // Pretty print JSON
        } catch (error: any) {
            console.error('Error fetching HTTP data:', error);
            setHttpResponse(`Error: ${error.message}`);
        }
    };

    // Function to send a message via WebSocket
    const sendWsMessage = () => {
        const ws = wsRef.current;
        if (ws && ws.readyState === WebSocket.OPEN && messageInput.trim()) {
            ws.send(messageInput.trim());
            setWsMessages(prev => [...prev, `You: ${messageInput.trim()}`]);
            setMessageInput(''); // Clear input after sending
        } else if (ws && ws.readyState !== WebSocket.OPEN) {
            setWsMessages(prev => [...prev, 'System: WebSocket not connected.']);
        } else {
            setWsMessages(prev => [...prev, 'System: Message cannot be empty.']);
        }
    };

    return (
        <div className="min-h-screen bg-gray-100 p-8">
            <div className="max-w-4xl mx-auto bg-white shadow-lg rounded-lg p-6">
                <h1 className="text-3xl font-bold text-gray-800 mb-6 border-b pb-4">
                    Separate Servers Demo (Next.js, Express, WebSocket)
                </h1>

                {/* HTTP API Section */}
                <section className="mb-8">
                    <h2 className="text-2xl font-semibold text-gray-700 mb-4">HTTP API (Port 3000)</h2>
                    <button
                        onClick={fetchHttpData}
                        className="bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded-md transition duration-300 ease-in-out"
                    >
                        Fetch Data from HTTP API
                    </button>
                    <pre className="mt-4 p-4 bg-gray-50 rounded-md text-sm overflow-auto max-h-60">
                        {httpResponse || 'Click the button to fetch data...'}
                    </pre>
                </section>

                {/* WebSocket Section */}
                <section>
                    <h2 className="text-2xl font-semibold text-gray-700 mb-4">WebSocket Chat (Port 8081)</h2>
                    <div className="border border-gray-300 rounded-md p-4 h-80 overflow-y-auto bg-gray-50 mb-4">
                        {wsMessages.map((msg, index) => (
                            <p key={index} className="text-gray-700 text-sm mb-1 last:mb-0">
                                {msg}
                            </p>
                        ))}
                        <div ref={messagesEndRef} /> {/* Scroll to this div */}
                    </div>
                    <div className="flex">
                        <input
                            type="text"
                            value={messageInput}
                            onChange={(e) => setMessageInput(e.target.value)}
                            onKeyPress={(e) => { if (e.key === 'Enter') sendWsMessage(); }}
                            placeholder="Type your message..."
                            className="flex-grow border border-gray-300 rounded-l-md p-2 focus:outline-none focus:ring-2 focus:ring-blue-400"
                        />
                        <button
                            onClick={sendWsMessage}
                            className="bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded-r-md transition duration-300 ease-in-out"
                        >
                            Send
                        </button>
                    </div>
                </section>
            </div>
        </div>
    );
}
```