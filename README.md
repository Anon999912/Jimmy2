<!welcome home>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Online Chat</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
        }
        .message {
            padding: 10px;
            margin-bottom: 5px;
            border-radius: 5px;
        }
        .sent {
            background-color: #dcf8c6;
            text-align: right;
            margin-left: auto;
        }
        .received {
            background-color: #f0f0f0;
            text-align: left;
            margin-right: auto;
        }
        #chat-container {
            max-height: 400px;
            overflow-y: auto;
            margin-bottom: 10px;
            border: 1px solid #e2e8f0;
            border-radius: 0.5rem;
            padding: 10px;
            background-color: rgba(255, 255, 255, 0.8);
            backdrop-filter: blur(10px);
        }
        .container {
            backdrop-filter: blur(12px);
        }
        #login-container {
            background-color: rgba(255, 255, 255, 0.9);
            border-radius: 0.75rem;
            padding: 2rem;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            max-width: 350px;
            margin: 0 auto;
            text-align: center;
        }
        #wallpaper-options {
            display: flex;
            justify-content: center;
            margin-top: 1rem;
            gap: 1rem;
        }
        #wallpaper-options button {
            width: 50px;
            height: 30px;
            border-radius: 0.25rem;
            cursor: pointer;
            border: none;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
            transition: transform 0.2s ease-in-out;
        }
        #wallpaper-options button:hover {
            transform: scale(1.1);
        }
        .option1 { background: linear-gradient(to bottom, #f0f2ff, #e0e7ff); }
        .option2 { background: linear-gradient(to bottom, #e0f7fa, #c2e5ed); }
        .option3 { background: linear-gradient(to bottom, #ffe082, #ffc107); }
        .option4 { background: linear-gradient(to bottom, #d1c4e9, #b39ddb); }
        #flower-icon {
            position: absolute;
            bottom: 10px;
            right: 10px;
            width: 50px;
            height: auto;
            opacity: 0.7;
            transition: opacity 0.3s ease;
            cursor: pointer;
            z-index: 10;
        }
        #flower-icon:hover {
            opacity: 1;
            transform: scale(1.1);
        }
        .media-message {
            max-width: 80%;
            height: auto;
            border-radius: 0.5rem;
            margin-bottom: 0.5rem;
        }
    </style>
</head>
<body class="bg-gray-100 p-4">
    <div id="login-container" class="hidden">
        <h2 class="text-2xl font-semibold text-gray-800 mb-4">Login</h2>
        <input type="text" id="login-username" placeholder="Username" class="border rounded-md p-2 mb-3 w-full focus:outline-none focus:ring-2 focus:ring-blue-500">
        <input type="password" id="login-password" placeholder="Password" class="border rounded-md p-2 mb-4 w-full focus:outline-none focus:ring-2 focus:ring-blue-500">
        <button id="login-button" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md focus:outline-none focus:shadow-outline w-full">Login</button>
        <p id="login-error" class="mt-2 text-red-500"></p>
    </div>

    <div class="container mx-auto bg-white shadow-md rounded-lg p-6" id="chat-app-container" style="display: none;">
        <h1 class="text-2xl font-semibold text-gray-800 mb-4 text-center">Online Chat</h1>
        <div id="chat-container" class="mb-4">
            </div>
        <div class="flex space-x-4">
            <input type="text" id="message-input" placeholder="Type your message..." class="flex-1 border rounded-md p-2 focus:outline-none focus:ring-2 focus:ring-blue-500">
            <input type="file" id="media-input" accept="image/*,video/*" class="hidden">
            <button id="send-button" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md focus:outline-none focus:shadow-outline">Send</button>
            <button id="media-button" class="bg-green-500 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-md focus:outline-none focus:shadow-outline">Media</button>
        </div>
        <div class="mt-4 text-center">
            <p id="my-id" class="text-gray-600">Your ID: karmesh</p>
            <p id="peer-id-display" class="text-gray-600">Peer ID: tayyaba</p>
            <p id="connection-status" class="text-gray-700 font-medium"></p>
        </div>
        <div id="wallpaper-options">
            <button class="option1"></button>
            <button class="option2"></button>
            <button class="option3"></button>
            <button class="option4"></button>
        </div>
    </div>
    <img id="flower-icon" src="https://www.svgrepo.com/show/119798/flower.svg" alt="Flowers Icon">

    <script>
        const chatContainer = document.getElementById('chat-container');
        const messageInput = document.getElementById('message-input');
        const sendButton = document.getElementById('send-button');
        const mediaButton = document.getElementById('media-button');
        const mediaInput = document.getElementById('media-input');
        const connectionStatus = document.getElementById('connection-status');
        const myIdDisplay = document.getElementById('my-id');
        const peerIdDisplay = document.getElementById('peer-id-display');

        const loginContainer = document.getElementById('login-container');
        const loginButton = document.getElementById('login-button');
        const loginUsernameInput = document.getElementById('login-username');
        const loginPasswordInput = document.getElementById('login-password');
        const loginError = document.getElementById('login-error');
        const chatAppContainer = document.getElementById('chat-app-container');
        const wallpaperOptions = document.getElementById('wallpaper-options');
        const flowerIcon = document.getElementById('flower-icon');

        let ws;
        let connected = false;
        const correctUsername = "tayesh";
        const correctPassword = "tayesh";
        const myId = "karmesh";
        const peerId = "tayyaba";
        let peerConnected = false;


        function initWebSocket() {
            ws = new WebSocket(`ws://${location.host}`);
            // ws = new WebSocket(`ws://yourdomain.com`);

            ws.onopen = () => {
                console.log('Connected to WebSocket server');
                connectionStatus.textContent = 'Connected';
                connected = true;
                // Simulate auto-connecting to tayyaba
                simulateConnect();
            };

            ws.onmessage = (event) => {
                try {
                    const data = JSON.parse(event.data);
                    displayMessage(data.content, 'received', data.type, data.senderName);
                } catch (error) {
                    console.error('Error parsing WebSocket message:', error);
                    displayMessage(event.data, 'received', 'text', 'Server');
                }
            };

            ws.onclose = () => {
                console.log('Disconnected from WebSocket server');
                connectionStatus.textContent = 'Disconnected';
                connected = false;
                peerConnected = false;
                setTimeout(initWebSocket, 5000);
            };

            ws.onerror = (error) => {
                console.error('WebSocket error:', error);
                connectionStatus.textContent = 'Error: ' + error.message;
            };
        }

        function simulateConnect() {
            peerConnected = true;
            connectionStatus.textContent = `Connected to ${peerId}`;
            peerIdDisplay.textContent = `Peer ID: ${peerId}`;
        }


        function sendMessage() {
            const message = messageInput.value;
            if (!message && (!mediaInput.files || mediaInput.files.length === 0) || !connected) return;

            const senderName = myId;
            const recipientName = peerId;

            if (mediaInput.files && mediaInput.files.length > 0) {
                const file = mediaInput.files[0];
                const reader = new FileReader();

                reader.onloadend = () => {
                    let fileType = file.type.startsWith('image/') ? 'image' : file.type.startsWith('video/') ? 'video' : 'file';
                    const messageData = {
                        type: fileType,
                        content: reader.result,
                        senderName: senderName,
                        recipientName: recipientName,
                        name: file.name
                    };
                    ws.send(JSON.stringify(messageData));
                    displayMessage(reader.result, 'sent', fileType, file.name);
                    messageInput.value = '';
                    mediaInput.value = '';
                };
                reader.onerror = (error) => {
                    console.error("Error reading file:", error);
                    connectionStatus.textContent = "Error reading file: " + error.message;
                    messageInput.value = '';
                    mediaInput.value = '';
                }

                if (file.type.startsWith('image/')) {
                    reader.readAsDataURL(file);
                } else if (file.type.startsWith('video/')) {
                    reader.readAsDataURL(file);
                } else {
                    reader.readAsArrayBuffer(file);
                }


            } else {
                const messageData = {
                    type: 'text',
                    content: message,
                    senderName: senderName,
                    recipientName: recipientName
                };
                ws.send(JSON.stringify(messageData));
                displayMessage(message, 'sent', 'text', senderName);
                messageInput.value = '';
            }
        }

        function displayMessage(content, type, messageType, senderName) {
            const messageDiv = document.createElement('div');
            messageDiv.className = `message ${type}`;
            const senderSpan = document.createElement('span');
            senderSpan.style.fontWeight = 'bold';
            senderSpan.style.display = 'block';
            senderSpan.style.textAlign = type === 'sent' ? 'right' : 'left';
            senderSpan.textContent = senderName ? senderName + ": " : "Anonymous: ";
            messageDiv.appendChild(senderSpan);

            if (messageType === 'text') {
                messageDiv.textContent += content;
            } else if (messageType === 'image') {
                const img = document.createElement('img');
                img.src = content;
                img.alt = 'Image';
                img.className = 'media-message';
                messageDiv.appendChild(img);
                const nameDiv = document.createElement('div');
                 nameDiv.textContent = name;
                 nameDiv.style.textAlign = type === 'sent' ? 'right' : 'left';
                 messageDiv.appendChild(nameDiv);

            } else if (messageType === 'video') {
                const video = document.createElement('video');
                video.src = content;
                video.controls = true;
                video.autoplay = false;
                video.className = 'media-message';
                messageDiv.appendChild(video);
                const nameDiv = document.createElement('div');
                 nameDiv.textContent = name;
                 nameDiv.style.textAlign = type === 'sent' ? 'right' : 'left';
                 messageDiv.appendChild(nameDiv);
            }
            else if (messageType === 'file') {
                const link = document.createElement('a');
                link.href = content;
                link.download = name;
                link.textContent = `File: ${name}`;
                link.style.textAlign = type === 'sent' ? 'right' : 'left';
                messageDiv.appendChild(link);
            }
            else {
                messageDiv.textContent = content;
            }

            chatContainer.appendChild(messageDiv);
            chatContainer.scrollTop = chatContainer.scrollHeight;
        }

        function showChatApp() {
            loginContainer.classList.add('hidden');
            chatAppContainer.style.display = 'block';
            initWebSocket();
        }

        function handleLogin() {
            const username = loginUsernameInput.value;
            const password = loginPasswordInput.value;

            if (username === correctUsername && password === correctPassword) {
                showChatApp();
            } else {
                loginError.textContent = "Invalid credentials. Please try again.";
            }
        }

        loginButton.addEventListener('click', handleLogin);
        loginPasswordInput.addEventListener('keydown', (event) => {
            if (event.key === 'Enter') {
                handleLogin();
            }
        });

        sendButton.addEventListener('click', sendMessage);
        messageInput.addEventListener('keydown', (event) => {
            if (event.key === 'Enter') {
                sendMessage();
            }
        });
        mediaButton.addEventListener('click', () => {
            mediaInput.click();
        });

        mediaInput.addEventListener('change', () => {
             sendMessage();
        });

        wallpaperOptions.addEventListener('click', (event) => {
            const target = event.target;
            if (target.tagName === 'BUTTON') {
                changeBackground(target.className);
            }
        });

        flowerIcon.addEventListener('click', () => {
             alert("Flowers are beautiful, aren't they?");
        });



        // Show login form on page load
        window.onload = function() {
            loginContainer.classList.remove('hidden');
        };
    </script>
</body>
</html>

