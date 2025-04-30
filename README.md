<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Real-Time Chat</title>
    <script src="https://unpkg.com/peerjs@1.3.2/dist/peerjs.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
        }
        .message {
            padding: 10px;
            margin-bottom: 5px;
            border-radius: 5px;
        }
        .sent {
            background-color: #dcf8c6; /* Light green for sent messages */
            text-align: right;
            margin-left: auto; /* Push sent messages to the right */
        }
        .received {
            background-color: #f0f0f0; /* Light gray for received messages */
            text-align: left;
            margin-right: auto; /* Push received messages to the left */
        }
        #chat-container {
            max-height: 400px; /* Limit the height of the chat container */
            overflow-y: auto; /* Enable vertical scrollbar */
            margin-bottom: 10px;
            border: 1px solid #e2e8f0;
            border-radius: 0.5rem;
            padding: 10px;
        }
    </style>
</head>
<body class="bg-gray-100 p-4">
    <div class="container mx-auto bg-white shadow-md rounded-lg p-6">
        <h1 class="text-2xl font-semibold text-gray-800 mb-4 text-center">Real-Time Chat</h1>
        <div id="chat-container" class="mb-4">
            </div>
        <div class="flex space-x-4">
            <input type="text" id="message-input" placeholder="Type your message..." class="flex-1 border rounded-md p-2 focus:outline-none focus:ring-2 focus:ring-blue-500">
            <button id="send-button" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md focus:outline-none focus:shadow-outline">Send</button>
        </div>
        <div class="mt-4 text-center">
            <p id="my-id" class="text-gray-600">Your ID: </p>
            <p id="peer-id-display" class="text-gray-600">Peer ID: </p>
            <div class="flex space-x-2 justify-center mt-2">
                <input type="text" id="peer-id-input" placeholder="Enter peer ID" class="border rounded-md p-2 focus:outline-none focus:ring-2 focus:ring-blue-500 w-48">
                <button id="connect-button" class="bg-green-500 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-md focus:outline-none focus:shadow-outline">Connect</button>
            </div>
            <p id="connection-status" class="mt-2 text-gray-700 font-medium"></p>
        </div>
    </div>

    <script>
        const chatContainer = document.getElementById('chat-container');
        const messageInput = document.getElementById('message-input');
        const sendButton = document.getElementById('send-button');
        const connectButton = document.getElementById('connect-button');
        const peerIdInput = document.getElementById('peer-id-input');
        const myIdDisplay = document.getElementById('my-id');
        const peerIdDisplay = document.getElementById('peer-id-display');
        const connectionStatus = document.getElementById('connection-status');

        let peer;
        let conn;
        let peerId;

        function initializePeer() {
            try {
                peer = new Peer(); // Initialize PeerJS
                peer.on('open', id => {
                    myIdDisplay.textContent = `Your ID: ${id}`;
                    peerId = id;
                });

                peer.on('connection', connection => {
                    conn = connection;
                    console.log('Received connection from peer:', conn.peer);
                    connectionStatus.textContent = `Connected to ${conn.peer}`;
                    peerIdDisplay.textContent = `Peer ID: ${conn.peer}`;
                    conn.on('data', data => {
                        displayMessage(data, 'received');
                    });
                    conn.on('close', () => {
                        connectionStatus.textContent = 'Connection closed.';
                        conn = null;
                    });
                    conn.on('error', err => {
                        console.error('Connection error:', err);
                        connectionStatus.textContent = 'Error: ' + err.message;
                        conn = null;
                    });
                });

                peer.on('disconnected', () => {
                    console.log('Disconnected from PeerJS server');
                    connectionStatus.textContent = 'Disconnected. Reconnecting...';
                    // Handle reconnection (optional)
                    setTimeout(initializePeer, 5000); //attempt to reconnect after 5 seconds
                });

                peer.on('error', err => {
                    console.error('PeerJS error:', err);
                    connectionStatus.textContent = 'Error: ' + err.message;
                    if (err.type === 'peer-unavailable') {
                         connectionStatus.textContent = 'Peer unavailable. Please check the ID and try again.';
                         conn = null;
                    }
                    // Consider re-initializing peer if the error is serious
                    //  initializePeer();
                });

            } catch (error) {
                console.error("Failed to initialize PeerJS:", error);
                connectionStatus.textContent = "Failed to initialize connection: " + error.message;
            }
        }


        function connectToPeer() {
            const peerIdToConnect = peerIdInput.value;
            if (!peerIdToConnect) {
                alert('Please enter a Peer ID to connect.');
                return;
            }

            if (peerIdToConnect === peerId) {
                alert('Cannot connect to yourself!');
                return;
            }
            try{
                conn = peer.connect(peerIdToConnect);
                conn.on('open', () => {
                    console.log('Connected to peer:', peerIdToConnect);
                    connectionStatus.textContent = `Connected to ${peerIdToConnect}`;
                    peerIdDisplay.textContent = `Peer ID: ${peerIdToConnect}`;
                    conn.on('data', data => {
                        displayMessage(data, 'received');
                    });
                    conn.on('close', () => {
                        connectionStatus.textContent = 'Connection closed.';
                        conn = null;
                    });
                     conn.on('error', err => {
                        console.error('Connection error:', err);
                        connectionStatus.textContent = 'Error: ' + err.message;
                        conn = null;
                    });
                });

                conn.on('error', err => {
                    console.error('Connection error:', err);
                    connectionStatus.textContent = 'Error: ' + err.message;
                    conn = null;
                });
            } catch(error){
                 console.error("Error connecting to peer:", error);
                 connectionStatus.textContent = "Error connecting to peer: " + error.message;
            }
        }

        function sendMessage() {
            const message = messageInput.value;
            if (!message || !conn) return;

            conn.send(message);
            displayMessage(message, 'sent');
            messageInput.value = ''; // Clear the input after sending
        }

        function displayMessage(message, type) {
            const messageDiv = document.createElement('div');
            messageDiv.className = `message ${type}`;
            messageDiv.textContent = message;
            chatContainer.appendChild(messageDiv);
            // Scroll to the bottom to show the latest message
            chatContainer.scrollTop = chatContainer.scrollHeight;
        }

        // Event Listeners
        sendButton.addEventListener('click', sendMessage);
        connectButton.addEventListener('click', connectToPeer);
        messageInput.addEventListener('keydown', (event) => {
            if (event.key === 'Enter') {
                sendMessage();
            }
        });

        // Initialize PeerJS when the page loads
        window.onload = initializePeer;
    </script>
</body>
</html>

